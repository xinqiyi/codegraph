# 混合 iOS + React Native 桥接 — 覆盖设计

**受众：** 继续此项工作的 Claude 代理（或人类），在 #165 落地纯 Objective-C 支持之后。
**使命：** 使 CodeGraph 的 `trace` / `callers` / `callees` / `impact` / 流程上下文调用能够端到端地跨**跨语言运行时分发边界**连接，这些边界今天静默地断裂流程：**Swift ↔ Objective-C** 在混合 iOS 代码库中，以及 **JavaScript ↔ 原生** 在 React Native / Expo 应用中。

> 本文档是**计划**，而非实现。此分支上没有代码落地——只有设计、验证语料库和成功标准。
> 编码在后续的每个阶段分支上开始。

此项工作是[动态分发覆盖指南](./dynamic-dispatch-coverage-playbook.md)第 6 节矩阵中的下一个项目：行 "Swift × Objective-C 桥接" 和一个新的 "React Native 桥接" 行。两者都是**解析器**模式（两边都存在命名引用——桥接规则是确定性的）——而不是合成器模式。参见指南第 3a 节中的 Django ORM 解析器参考。

---

## 1. 为什么这很重要（今天的差距）

在 #165 之后，CodeGraph 分别正确地索引了 Swift、Objective-C 和 JavaScript/TypeScript，**各自独立**。但价值在于跨语言流程——正是 iOS 应用和 React Native 应用所在之处：

- **混合 iOS 应用：** `MyViewController.swift` 调用 `imageDownloader.download(url:completion:)`，即 `ImageDownloader.m` 中的 `-[ImageDownloader downloadURL:completion:]`。今天：`trace("MyViewController.viewDidLoad", "downloadURL:completion:")` 调用返回无路径。Swift 调用点解析为一个 `call_expression`，其选择器无去处；ObjC 方法作为节点存在，但没有入边。代理读取两个文件以重建桥接。
- **React Native 应用：** `App.js` 中的 `useEffect(() => NativeModules.Geolocation.getCurrentPosition(cb))` 到达 `RNCGeolocation.m` 中的 `RCT_EXPORT_METHOD(getCurrentPosition:(RCTResponseSenderBlock)cb)`。今天：JS 调用点没有到 ObjC 实现的出边；ObjC handler 没有来自 JS 的入边。`impact(getCurrentPosition)`（ObjC 端）没有显示 JS 调用者。
- **Expo 模块：** `await ExpoCamera.takePictureAsync(options)`（JS）到达 `ExpoCamera.swift` 中的 `AsyncFunction("takePictureAsync") { ... }`（Expo Modules API）。同样断裂。

在每种情况下，**两边都存在一个名称**，代理或名称匹配器可以关联它——Swift 的自动桥接 ObjC 选择器、`RCT_EXPORT_METHOD` 的字面量第一个参数、Expo `Function("name")` 字面量。修复是一个**解析器**，它知道每个通道的桥接规则，并发射带有 `provenance:'heuristic'` 和 `metadata.synthesizedBy:'<channel>'` 的 `references` 边。

指南中承重的警告在这里比平常更适用：

> **部分覆盖比没有更差。** 桥接一个边界但不桥接下一个边界会暴露一跳，代理然后会深入并读取以完成。始终将流程端到端闭合并重新测量——绝不发布半桥接的流程。

对于混合 iOS，这意味着**两个方向**（Swift→ObjC 和 ObjC→Swift）和**所有桥接种类**（方法、属性、初始化器、协议）必须在测量之前闭合。对于 React Native，JS→原生 AND 原生→JS（`RCTEventEmitter`、`sendEvent`）必须都闭合，AND 在**传统桥接和 TurboModules 两者**上，否则混合使用它们的应用将会半桥接。

---

## 2. 要建模的桥接机制

每一行都是指南词汇表中的独立**分发通道**——每个通道有自己的解析器（如果没有静态引用则为合成器）、自己的验证、自己在第 6 节矩阵中的行。

| # | 方向 | 通道 | 映射规则 | 所在位置 | 难度 |
|---|---|---|---|---|---|
| 1 | Swift → ObjC | 直接调用，通过 `-Bridging-Header.h` 导入的 ObjC 类 | Swift 调用 `obj.x(y:z:)` ↔ ObjC 选择器 `-x:z:`（字面映射，见第 3a 节） | `frameworks/swift-objc.ts` 中的解析器 | 中等 |
| 2 | ObjC → Swift | `@objc` 暴露 | Swift `@objc func foo(bar:)` ↔ ObjC `-fooWithBar:`（自动名称）；`@objc(custom:)` 覆盖 | `frameworks/swift-objc.ts` 中的解析器 | 中等 |
| 3 | Swift ↔ ObjC | 属性/getter/setter 桥接 | Swift `var name: String` ↔ ObjC `-name` / `-setName:` | `frameworks/swift-objc.ts` 中的解析器 | 低 |
| 4 | Swift ↔ ObjC | 初始化器桥接 | Swift `init(name:age:)` ↔ ObjC `-initWithName:age:` | `frameworks/swift-objc.ts` 中的解析器 | 低 |
| 5 | Swift ↔ ObjC | 协议桥接（`@objc protocol`） | 跨语言的一致性边 | `frameworks/swift-objc.ts` 中的解析器 | 中等 |
| 6 | JS → ObjC（RN 传统桥） | `NativeModules.<Mod>.<fn>` ↔ `RCT_EXPORT_METHOD(<fn>:...)` 或 `RCT_REMAP_METHOD(<jsName>, <selector>:...)` | 名称匹配，键控于 ObjC 端的 `RCT_EXPORT_MODULE()` 字面量 | `frameworks/react-native.ts` 中的解析器 | 中等 |
| 7 | JS → Java/Kotlin（RN 传统桥，Android） | `NativeModules.<Mod>.<fn>` ↔ `@ReactMethod` 注解方法在 `ReactContextBaseJavaModule` 子类上，`getName()` 返回 `<Mod>` | 解析器——与 #6 相同形状，JVM 端 | 中等 |
| 8 | JS ↔ 原生（RN TurboModules / Codegen） | `TurboModuleRegistry.get('Mod')` ↔ 生成的规范接口（`NativeMod` TS 类型）↔ 匹配规范的 ObjC++/Kotlin 实现 | 将规范文件作为事实来源读取的解析器 | 困难 |
| 9 | 原生 → JS（事件） | ObjC `[self sendEventWithName:@"x" body:b]`（扩展 `RCTEventEmitter`）↔ JS `new NativeEventEmitter(NativeModules.Mod).addListener('x', cb)` | EventEmitter 风格合成器（匹配现有的 `callback-synthesizer.ts` 用于语言内 EventEmitter） | 中等 |
| 10 | JS → 原生（Expo 模块） | JS `ExpoX.fn(args)` ↔ Swift `Function("fn") { ... }` 或 `AsyncFunction("fn") { ... }` 在带有 `Name("ExpoX")` 的 `Module` 子类内部 | `frameworks/expo-modules.ts` 中的解析器 | 中等 |
| 11 | JS → 原生（Fabric 视图组件） | JS `<MyView prop={v}/>` ↔ ObjC/Swift `RCT_EXPORT_VIEW_PROPERTY(prop, ...)` 或 Codegen 视图规范 | 解析器 + JSX 跳（与现有 JSX 合成器组合） | 困难（推迟） |

**难度**列驱动分阶段——见第 6 节。

### 2a. 为什么这些是解析器，而不是合成器

在每一行中，**桥接规则从一个名称中是确定性的**：
- Swift 的 `@objc` 暴露是一个文档化的自动映射；`@objc(custom:)` 是一个显式覆盖；两者都可静态提取。
- `RCT_EXPORT_METHOD` 接受一个字面量选择器；`RCT_EXPORT_MODULE()` 接受一个可选的字面量模块名（默认：类名减去 `RCT` 前缀）；`NativeModules.Mod.fn` 是对已知全局对象的字面量属性访问。
- Expo Modules `Function("name") { ... }` 和 `Module { Name("ExpoX"); ... }` 是 `Module` 定义内部的字面量字符串。
- TurboModules 规范接口是带有 `TurboModuleRegistry.get<...>('<Name>')` 的字面量 `Native<Name>` 导出。

因此工作是：**提取桥接端名称 → 让解析器匹配它们**。与 `djangoResolver` 将 `_iterable_class` 解析为 `ModelIterable` 相同形状——不需要全图关联遍历。

一个例外是 **#9 原生→JS 事件**，其中注册站点看起来很像语言内 EventEmitter 模式，现有的回调合成器已经处理了。将该合成器扩展一个跨语言通道是自然的匹配。

---

## 3. 具体桥接规则（参考表）

### 3a. Swift → ObjC 选择器映射（自动）

Swift 使用标准规则从 Swift 方法派生 ObjC 选择器：

| Swift 声明 | ObjC 选择器 |
|---|---|
| `func greet()` | `greet` |
| `func say(_ msg: String)` | `say:` |
| `func set(name: String)` | `setWithName:` |
| `func setName(_ name: String)` | `setName:` |
| `func move(to point: CGPoint)` | `moveTo:` |
| `func move(from a: CGPoint, to b: CGPoint)` | `moveFrom:to:` |
| `init(name: String)` | `initWithName:` |
| `init(name: String, age: Int)` | `initWithName:age:` |
| `var name: String`（getter） | `name` |
| `var name: String`（setter） | `setName:` |
| `@objc(customSel:) func f(...)` | `customSel:`（显式覆盖） |

完整的规则集在 [Apple — Importing Swift into Objective-C](https://developer.apple.com/documentation/swift/importing-swift-into-objective-c) 中——特别是"方法名翻译"和"初始化器名翻译"部分。解析器在**提取时单向**实现此映射（Swift 声明产生桥接的 ObjC 名称，作为别名附加在 Swift 方法节点上），因此 ObjC 端的名称解析通过正常名称匹配找到 Swift 方法。

### 3b. React Native 传统桥——名称解析

```objc
// 原生端（ObjC）
@implementation RCTGeolocation
RCT_EXPORT_MODULE();                                    // 模块名："Geolocation"（RCT 前缀去除）
RCT_EXPORT_METHOD(getCurrentPosition:(RCTResponseSenderBlock)cb) { ... }
@end
```
```js
// JS 端
import { NativeModules } from 'react-native';
NativeModules.Geolocation.getCurrentPosition(cb);       // 解析到上面的 ObjC 方法
```

规则：
1. 在原生端，为每个包含 `RCT_EXPORT_MODULE()` 的类提取一个合成的 `module` 节点。名称 = 如果存在则显式字符串参数，否则类名去掉 `RCT` 前缀。
2. 每个 `RCT_EXPORT_METHOD(<sel>)` 和 `RCT_REMAP_METHOD(<jsName>, <sel>)` 成为附加到该模块节点的方法节点，具有 JS 可见名称（`RCT_EXPORT_METHOD` 的 `<sel>` 的首关键词，或 `RCT_REMAP_METHOD` 的 `<jsName>`）。
3. 在 JS 端，解析器匹配字面量属性链 `NativeModules.<Mod>.<fn>` 与来自原生端的 `(module, jsName)` 对。
4. 解析器从 JS 调用点到原生方法发射 `references`（`provenance:'heuristic'`，`synthesizedBy:'rn-bridge'`）。

### 3c. React Native TurboModule——名称解析

```ts
// 规范（TS）— codegen 事实来源
export interface Spec extends TurboModule {
  getCurrentPosition(cb: (loc: Location) => void): void;
}
export default TurboModuleRegistry.getEnforcing<Spec>('Geolocation');
```
```objc
// ObjC++ 实现
@implementation RCTGeolocation
- (void)getCurrentPosition:(RCTResponseSenderBlock)cb { ... }
@end
```
```js
import Geolocation from './NativeGeolocation';
Geolocation.getCurrentPosition(cb);  // 通过规范解析到 ObjC 方法
```

规则：
1. 规范文件是事实来源：解析 `TurboModuleRegistry.get*<Spec>('<Name>')` 找到模块名，然后读取 `Spec` 接口方法。
2. 每个规范方法匹配到原生实现的同名方法（通过选择器首关键词，在通过名称约定或读取任何 `JSI_EXPORT_MODULE` 宏（如果存在）识别的类中）。
3. 规范文件的 JS 导入通过规范获得名称解析。
4. 发射与 #3b 相同的 `references` 边，带有 `synthesizedBy:'rn-turbomodule'`。

### 3d. Expo Modules——名称解析

```swift
// 原生（Swift，expo-modules-core API）
public class ExpoCameraModule: Module {
  public func definition() -> ModuleDefinition {
    Name("ExpoCamera")
    AsyncFunction("takePictureAsync") { (options: CameraOptions) in /* ... */ }
    View(ExpoCameraView.self) {
      Prop("type") { (view: ExpoCameraView, type: String) in /* ... */ }
    }
  }
}
```
```js
import { requireNativeModule } from 'expo-modules-core';
const ExpoCamera = requireNativeModule('ExpoCamera');
await ExpoCamera.takePictureAsync({ quality: 1 });
```

规则：
1. 在原生端：一个扩展 `Module` 的类，其 `definition()`（或较新 API 的 `init { /* DSL */ }`）包含一个 `Name("X")` 调用定义模块。每个 `Function("y")` / `AsyncFunction("y")` 字面量定义一个方法。尾部闭包是实现主体——提取为名为 `y` 的方法节点，附加到模块 `X`。
2. 在 JS 端：`requireNativeModule('X')` 产生一个绑定；解析其上的属性访问到命名方法。
3. 视图模块的 `Prop("name")` 行为类似于 RN 的 `RCT_EXPORT_VIEW_PROPERTY`——与视图组件前沿的其余部分一起推迟。

---

## 4. 需要存在哪些边

对于每个通道，闭合的流程是：

- **JS 调用点 → 桥接方法节点**（`references`，启发式，`synthesizedBy:'<channel>'`）
- **桥接方法节点 → 原生实现方法**（已经提取；对于 #6/#7，桥接方法就是原生实现；对于 #10，闭包主体就是实现）
- **原生实现方法 → 其自身的被调用者**（已经在语言内提取）

对于 Swift↔ObjC 具体来说，最干净的模型是**声明节点上的别名名称**：扩展 Swift 方法提取以计算 ObjC 自动桥接名称，并将其存储为解析器考虑的备用名称。不需要 Swift 和 ObjC 方法节点之间的新边——正常名称解析就足够了，因为两边在提取后都同意桥接的选择器。

MCP 读取工具已经内联显示启发式边（参见来自 #312/#403 的 `metadata.synthesizedBy` 管道）；这些新边沿着该路径走，不需要额外的管道。

---

## 5. 验证语料库（小/中/大标准）

遵循 CLAUDE.md 的验证方法论——**在小/中/大代码库上各使用 ≥3 个流程提示词，使用确定性探针 + 代理 A/B，≥2 次运行/臂**。以下选择是实现分支上要提交的候选；实现 PR 在验证每个代码库仍然干净地构建索引后确认选择。

### 5a. 混合 iOS（Swift+ObjC）——选择 3 个

| 层级 | 代码库 | 原因 | 规范流程 |
|---|---|---|---|
| **小** | [Charts](https://github.com/danielgindi/Charts)（约 150 个文件 Swift+ObjC） | Swift 优先的库，带有 ObjC 兼容层；众所周知 | "设置 `data` 在 `ChartView` 上如何到达渲染器？" |
| **小（备选）** | [Lottie-ios](https://github.com/airbnb/lottie-ios)（约 300 个文件，曾是混合的；当前可能是纯 Swift——验证） | 动画引擎，众所周知的混合 | "`AnimationView.play()` 如何到达图层合成器？" |
| **中** | [Realm-Cocoa](https://github.com/realm/realm-swift)（约 500 个文件） | 重度 Swift-on-top-of-ObjC：Swift API 包装了一个 ObjC 核心，它又包装了 C++ Realm Core | "`Realm.write { realm.add(obj) }` 如何到达 ObjC 持久化层？" |
| **大** | [Wikipedia-iOS](https://github.com/wikimedia/wikipedia-ios)（约 2500 个 Swift+ObjC 文件） | 真实应用，深度混合，活跃开发 | "点击搜索结果如何到达文章获取网络调用？" |
| **大（备选）** | [WordPress-iOS](https://github.com/wordpress-mobile/WordPress-iOS) | 较重 ObjC 遗留代码 + Swift 添加 | "新帖子草稿保存如何到达 Core Data 持久化？" |

每个代码库的标准：
1. 纯语言探针仍然通过（Swift 内 Swift 的 trace；ObjC 内 ObjC 的 trace）——与 #165 的纯 ObjC 基线相比无回归。
2. **跨语言探针通过：** 上述规范流程端到端地用 `trace` 追踪，在语言边界处无断裂。
3. **Agent A/B（使用 vs 不使用 codegraph，≥2 次运行/臂）：** 在探索调用预算内 Read = 0；比不使用 codegraph 更快；在纯 Swift 或纯 ObjC 控制代码库上没有回归（例如 Texture）。
4. **无节点计数爆炸** vs 桥接前基线（`select count(*) from nodes` 前后比较）。

### 5b. React Native——选择 3 个

| 层级 | 代码库 | 原因 | 规范流程 |
|---|---|---|---|
| **小** | [react-native-svg](https://github.com/software-mansion/react-native-svg)（约 100 个文件 JS+ObjC+Java） | 小型、范围良好的原生模块集 | "设置 `<Path d=.../>` 如何到达 iOS Core Graphics 调用？" |
| **中** | [react-native-screens](https://github.com/software-mansion/react-native-screens)（约 300 个文件，JS+原生） | 真实导航原语，传统桥接和 Fabric | "导航到新屏幕如何到达 UINavigationController？" |
| **中（备选）** | [react-native-firebase](https://github.com/invertase/react-native-firebase)（跨包约 1000 个文件） | 许多原生模块，两个平台——对模块发现施加压力 | "`firestore().collection('x').get()` 如何到达 iOS Firebase SDK 调用？" |
| **大** | [facebook/react-native](https://github.com/facebook/react-native) RNTester 子集（约 3000 个文件） | 框架本身 + 示例应用；规范桥接用法 | "按下 RNTester 的 GeolocationExample 中的按钮如何到达 iOS Core Location 调用？" |

每个代码库的标准：
1. 纯 JS 探针不变（`useState` → 重新渲染流程仍然解析——现有 react 合成器未回归）。
2. **JS → ObjC 桥接探针通过**每个代码库上 ≥1 个已知 `RCT_EXPORT_METHOD`。
3. **JS → TurboModule 探针通过**在使用 TurboModules 的代码库上（react-native main 两者都有；各选一个）。
4. **原生 → JS 事件探针通过** ≥1 个发射器（NativeEventEmitter 模式）。
5. **Agent A/B** 如上。关键：一个*跨越桥接*的问题（例如"按下按钮 X 如何到达网络调用"）必须在使用 codegraph 时在 ≥1 次运行中将 Read 降至 0。
6. **无回归**在纯 JS 控制代码库上（现有 react-realworld / excalidraw 测量不变）。

### 5c. Expo——选择 2 个（范围较小，API 表面较窄）

| 层级 | 代码库 | 原因 |
|---|---|---|
| **小/中** | [expo/expo](https://github.com/expo/expo) — 一个 SDK 模块如 `expo-camera` 或 `expo-location` | 最干净的 Expo Modules API 示例；活跃 |
| **大** | 完整的 `expo/expo` 单仓库（所有 SDK 模块 + JS API） | 跨许多包对模块名称解析进行压力测试 |

规范流程："`await Camera.takePictureAsync()`（JS）如何到达原生相机 API 调用（Swift `AVCaptureSession` 或 Kotlin `CameraDevice`）？"

---

## 6. 分阶段——什么先来

根据指南的难度梯度和半桥接规则，顺序由什么能**在最小代码库上首先端到端闭合流程**决定。

### 阶段 1——Swift ↔ ObjC 桥接（上面的行 1-5）
范围最小，确定性名称映射，不涉及 JS。在 Charts/Realm/Wikipedia 语料库上验证后再继续。**在阶段 1 通过第 5a 节所有三个代码库的标准之前，不要进入阶段 2。**

### 阶段 2——React Native 传统桥（行 6-7，ObjC + Java/Kotlin）
iOS 和 Android 端必须在同一个 PR 中闭合——仅桥接一个平台会暴露另一端的半覆盖跳，代理会读取。在第 5b 节语料库上验证。

### 阶段 3——原生 → JS 事件（行 9）
用跨语言通道扩展现有的回调合成器。在相同的第 5b 节语料库上验证（大多数 RN 库使用至少一个事件发射器）。

### 阶段 4——Expo Modules（行 10）
基于阶段 1 的 Swift 提取之上。较小的语料库（第 5c 节）。

### 阶段 5——RN TurboModules / Codegen（行 8）
需要将规范文件作为跨语言事实来源读取。在第 5b 节语料库的 TurboModule 用户（react-native main，0.73 后的库）上验证。

### 阶段 6——Fabric 视图组件（行 11）
推迟——与现有的 JSX 合成器和 TurboModules 的视图端组合。当第 5b 节语料库中的 ≥1 个代码库的桥接否则已闭合但 Fabric 流程仍然断裂时处理。

---

## 7. 反目标（我们不会尝试去做）

- **Android Kotlin/Java 提取质量**——范围外。我们使用 Kotlin/Java 提取器已经产生的。如果它们遗漏了 `@ReactMethod` 注解的字面量名称，我们可能会添加一个小的提取器改进，但我们不会重新设计 JVM 提取。
- **动态 / 计算的桥接键**——`NativeModules[someVar]`、`requireNativeModule(name)` 其中 `name` 是一个参数等。我们只解析字面量键访问（匹配[代理评估 Lua 前沿](./dynamic-dispatch-coverage-playbook.md)——仅匿名模式推迟）。
- **桥接头文件内容解析**——我们*确实*索引 `.h` 文件（已经通过 #165 的内容嗅探完成），但我们**不**将桥接头的 `#import` 列表解析为特殊的"什么对 Swift 可见"清单。将其视为普通的 ObjC 头文件。
- **`performSelector:` 上的运行时分发**——范围外；匹配相同的"仅命名"反目标。
- **JSI（原始，非 TurboModule）**——范围外。使用裸 JSI 的应用通过自定义的 `Host*` 接口调用原生，该接口没有文档化的声明式规范。等待这些应用迁移到 TurboModules。
- **Swift 仅对 ObjC 协议的泛型 / ObjC 类上的 Swift 扩展**——扩展方法如果 `@objc` 仍然可以在 ObjC 中调用，因此它们通过相同的阶段 1 路径。泛型不行——我们静默遗漏它们。可接受；匹配 Java/Kotlin 泛型前沿。

---

## 8. 覆盖矩阵条目——已测量

| 语言 | 框架 | 规范流程 | 机制 | 状态 |
|---|---|---|---|---|
| Swift × Objective-C | 桥接 | Swift 调用 → ObjC 选择器；ObjC 调用 → @objc Swift 方法 | R | ✅ 阶段 1（第 8a 节） |
| JavaScript × Objective-C/Java/Kotlin | React Native 传统桥 | `NativeModules.<M>.<f>` → `RCT_EXPORT_METHOD` / `@ReactMethod` | R | ✅ 阶段 2（第 8b 节） |
| JavaScript × 原生 | React Native TurboModules | 规范接口 ↔ 实现 | R（规范作为事实来源） | ✅ 部分——名称匹配路径落地（第 8b 节） |
| Objective-C/Java/Kotlin → JavaScript | React Native 事件发射器 | `[self sendEventWithName:]` → `addListener` | S（跨语言通道） | ✅ 阶段 3（第 8e 节） |
| JavaScript × Swift/Kotlin | Expo Modules | `requireNativeModule('X').fn(...)` → `Function("fn") { }` | R（提取合成方法节点） | ✅ 阶段 4（第 8f 节） |
| JavaScript × 原生 | React Native Fabric 视图 | `<MyView p=v/>` → Codegen 规范组件 + NativeProps | R（提取）+ S（原生实现）+ JSX | ✅ 阶段 6（第 8g 节） |

### 8a. 阶段 1 测量——Swift ↔ ObjC

| 代码库 | 源文件 | 桥接边（框架解析） | 示例边 |
|---|---|---|---|
| **Charts**（小） | 269（205 Swift + 59 ObjC/.h） | 28 objc→swift，1 swift→objc | `handleOption:forChartView:` → `animate` · `setupPieChartView:` → `setExtraOffsets` · `setDataCount:range:` → `setColor` |
| **realm-swift**（中） | 369（151 Swift + 218 ObjC 家族） | 36 objc→swift，1185 swift→objc | `valueForUndefinedKey:` → `get` · `setValue:forUndefinedKey:` → `set` · `promote:on:` → `initialize` |
| **wikipedia-ios**（大） | 1734（1234 Swift + 500 ObjC/.h） | 52 objc→swift，983 swift→objc | 跨许多功能模块的真实 iOS 应用桥接 |

所有三个：语言内基线不变，无节点计数爆炸，`trace` 跨边界连接规范流程（在 Charts 上验证：`trace(handleOption:forChartView:, animate)` 直接浮现桥接边）。

### 8b. 阶段 2 + 5（部分）测量——React Native 桥

| 代码库 | 源文件 | 桥接边（框架解析） | 备注 |
|---|---|---|---|
| **react-native-svg**（小/中） | 约 700（93 .mm + 115 .java + 6 .kt + 49 js + 92 ts + 154 tsx） | 9 tsx→java 通过 TurboModule 规范 | RNSvg 的 iOS 使用 TurboModule 自动生成（没有 `RCT_EXPORT_METHOD`）；解析落在 Java 上。全部 9 个精确：`isPointInStroke`、`isPointInFill`、`getTotalLength`、`getPointAtLength`、`getCTM`、`getScreenCTM`、`getBBox`、`toDataURL`。 |
| **AsyncStorage**（小，纯传统桥） | 约 60（28 kt + 2 mm + 16 ts + 14 tsx + …） | **8/8 精确** | 规范的传统桥测试——Kotlin `@ReactMethod` + ObjC `RCT_EXPORT_METHOD`。JS `setItem` → Kotlin `legacy_multiSet`；`getItem` → `legacy_multiGet`；`clear` → `legacy_clear`；等。 |
| **react-native-firebase**（大） | 约 1100（111 .java + 63 .m + 13 .mm + 239 js + 427 ts + 9 tsx） | `RCTEventEmitter` 阻止列表后 18（之前是 78） | 初始 78 包括 60 个针对 `addListener:` / `remove:` 的误报（每个 RCTEventEmitter 都声明它们；每个对 `.addListener(...)` 的 JS 调用都解析为噪音）。阻止列表削减到 18，全部精确：`httpsCallable:region:emulatorHost:...`、`signInWithProvider`、`configureProvider`、`removeFunctionsStreaming:`。 |
| **react-native-screens**（中） | 1211 | 0——空的 TurboModule 规范，没有 `RCT_EXPORT_METHOD`，全部是 Fabric/Codegen 视图端 | RNScreens 完全存在于阶段 6（Fabric，推迟）。桥在这里拒绝过度匹配是正确的行为。 |

### 8c. 验证期间发现的架构修复

解析器的 `initialize()` 在 CodeGraph 构造时运行——在任何文件被索引之前——因此其 `detect()` 咨询索引文件列表的框架解析器（UIKit / SwiftUI 扫描导入、`swift-objc-bridge` 查找 Swift 和 ObjC 文件、`react-native-bridge` 查找 RN 标记）在初始遍历中都返回 false 并静默丢弃自己。这影响了代码库中每个读取 `context.getAllFiles()` / `context.readFile()` 而不是直接扫描文件系统的框架解析器——一个预先存在的潜在 bug，不是桥特定的。修复：`indexAll()` 现在在提取完成后调用 `resolver.initialize()`，因此 `detect()` 针对已填充的索引运行。

### 8d. 桥精度阻止列表（经验教训）

| 桥 | 阻止的名称 | 原因 |
|---|---|---|
| swift-objc | `init`、`description`、`hash`、`isEqual`、`copy`、`count`、`value`、`data`、`string`、`object`、`add`、`remove`、`update`、`load`、`save`、`reload`、`cancel`、`start`、`stop`、`pause`、`resume`、`close`、`open`、`show`、`hide`、`dealloc`、`release`、`retain`、`autorelease`、… | 每个 NSObject 子类都实现这些；将它们桥接到任意的项目本地 ObjC 方法会产生噪音。常规名称匹配器自行处理它们。 |
| react-native | `addListener`、`removeListeners`、`remove`、`invalidate`、`startObserving`、`stopObserving` | 每个 `RCTEventEmitter` 子类都通过 `RCT_EXPORT_METHOD` 声明这些。`.addListener(...)` / `.remove(...)` 的 JS 调用者通过 `NativeEventEmitter`（JS 抽象）走，而不是直接通过原生桥。 |

### 8e. 阶段 3 测量——RN 原生 → JS 事件通道

合成器模式；扩展 `src/resolution/callback-synthesizer.ts` 带有一个跨语言事件通道，键控于字面量事件名。在 **RNFirebase**（大）上验证：

| 合成事件通道 | 边数 | 示例 |
|---|---|---|
| `messaging_message_received` | 2 | `application:didReceiveRemoteNotification:fetchCompletionHandler:` → TS `onMessage`（和 `UNUserNotificationCenter` willPresent 变体 → 相同 `onMessage`） |
| `messaging_notification_opened` | 1 | `userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler:` → TS `onNotificationOpenedApp` |

每条边是 `provenance:'heuristic'`，`metadata.synthesizedBy:'rn-event-channel'`。与语言内通道相同的 `EVENT_FANOUT_CAP = 6`——具有太多处理器或分发器的通用事件名跳过而不是过度链接。

合成器还处理 RN 库中常见的**订阅包装器模式**（`messaging().onMessage(listener)` 其中 `listener` 是一个流向用户代码的参数）：当 JS handler 参数不是命名符号时，它将监听器归属于封闭的 JS 函数（可达性正确，归属于抽象层）。

### 8f. 阶段 4 测量——Expo Modules

框架 `extract()` 解析 Swift / Kotlin 源代码中 `class X: Module`（或 Kotlin 中的 `: Module()`）内部的字面量 `Function("X") { … }` / `AsyncFunction("X") { … }` / `Property("X") { … }` / `Constants` 声明，并为每个字面量发射一个名为 `X` 的 `method` 节点。标准名称匹配器通过现有的 `obj.method` → 方法名路径将 JS 调用点如 `Foo.takePictureAsync(...)` 解析到这些合成节点。

在真实 Expo SDK 包上验证：

| 包 | 索引的文件 | 提取的 Expo 方法节点 | 跨语言边 |
|---|---|---|---|
| **expo-haptics** | 14 | 6（3 Swift + 3 Kotlin：`notificationAsync`、`impactAsync`、`selectionAsync` / `performHapticsAsync`） | 模块节点已注册；消费者应用调用者通过名称匹配解析 |
| **expo-camera** | 72 | 41（Swift + Kotlin；涵盖 `takePictureAsync`、`record`、`resumePreview`、`getAvailableLenses`、`scanFromURLAsync`、`requestCameraPermissionsAsync`、视图端 `width` / `height` 属性等） | 9 条 swift→expo，7 条 kotlin→expo 内部边。包中的 JS 端调用点用 TS 包装器遮蔽了原生名称（`pausePreview()` 在 `CameraView.tsx` 上定义）；名称匹配正确优先选择本地 TS 方法。`Camera.takePictureAsync()` 的外部消费者应用直接解析到原生方法。 |

五个测试覆盖提取器 + 一个端到端夹具：`字面量 AsyncFunction("uniqueExpoHapticCall") 的 JS 调用点解析到原生实现节点` — 确认了当名称未被遮蔽时无解析器桥路径有效。

### 8g. 阶段 6 测量——Fabric / Codegen 视图组件

两部分设计：

1. **框架提取器**（`src/resolution/frameworks/fabric.ts`）——解析 TS / TSX 规范文件中 `codegenNativeComponent<Props>('Name', ...)` 声明。发射：
   - 每个声明一个 `component` 节点（以 JS 可见的组件名命名；匹配 JSX 合成器的名称+种类过滤器）。
   - `NativeProps` 接口的每个声明字段一个 `property` 节点——将诸如 `onTap`、`nativeContainerBackgroundColor` 之类的 JSX 可调用属性浮现为可发现的图节点。

2. **合成器**（`callback-synthesizer.ts` 中的 `fabricNativeImplEdges`）——遍历每个 `fabric-component:*` 节点并查找一个与其名称匹配且带有 RN 约定后缀之一（空 / `View` / `ViewManager` / `ComponentView` / `Manager`）的原生类。从组件到每个匹配发射一条 `calls` 边，带有 `metadata.synthesizedBy:'fabric-native-impl'`。该约定足够精确，在格式良好的 RN 库中没有名称冲突。

结合现有的 `reactJsxChildEdges` JSX 合成器，这关闭了完整的 JSX → 原生流程：消费者应用 JSX `<MyView prop=v/>` → Fabric `component` 节点 `MyView` → 原生类 `MyViewView`（或 `MyViewManager` / `MyViewComponentView` / …）。

在 **react-native-screens**上重新验证（该语料库代码库完全属于 Fabric 且在阶段 2 中显示 0 条桥接）：

| 指标 | 计数 |
|---|---|
| `codegenNativeComponent` 规范声明 | 54 |
| 提取的 Fabric 组件节点 | 27（每个非 web 规范一个；`*.web.ts` 变体通过规范有效性过滤掉） |
| 提取的 Fabric 属性节点 | 272（跨所有组件的完整 NativeProps 接口表面） |
| `fabric-native-impl` 桥接边 | 68 |

示例桥接边：

| JS 组件 | 原生类 | 后缀 |
|---|---|---|
| `RNSFullWindowOverlay` | `RNSFullWindowOverlay`（ObjC） | （精确） |
| `RNSFullWindowOverlay` | `RNSFullWindowOverlayManager`（ObjC） | `Manager` |
| `RNSModalScreen` | `RNSModalScreenManager`（ObjC） | `Manager` |
| `RNSScreenContainer` | `RNSScreenContainerView`（ObjC） | `View` |

四个测试覆盖提取器 + 一个完整端到端夹具（`App（TSX）→ MyView（fabric-component）→ MyViewView（ObjC 类）`），断言 JSX→组件边 AND 组件→原生类边在索引后都存在。

---

## 9. 需要在阶段 1 中解决的开放问题

这些并不阻塞阶段 1 的开始——它们是*在*编写 Swift↔ObjC 解析器时要决定的第一件事：

1. **声明上的别名 vs 新桥接边？** 将自动桥接的 ObjC 选择器作为备用名称存储在 Swift 方法节点上更便宜，且与名称解析的工作方式一致。替代方案（在匹配节点之间合成一条跨语言 `references` 边）在 `trace` 输出中更显式，但为每个 `@objc` 符号添加 N 条边。**默认：别名。** 验证别名是否在 `callers`/`callees`/`trace` 结果中出现。
2. **`trace` 如何显示跨语言跳？** MCP `trace` 工具内联每个跳的主体。Swift → ObjC 跳应该在渲染输出中明显（"Swift `func foo(bar:)` → 桥接到 ObjC 选择器 `-fooWithBar:` → ObjC `-[ImageDownloader fooWithBar:]`"）。可能需要在 `trace.ts` 中做一个小的渲染器调整来标记桥接。
3. **解析器桥接规则住在哪里？** 建议使用 `src/resolution/frameworks/swift-objc.ts` 作为自动名称映射（一个纯函数），由 Swift 提取器（以在提取时计算别名）和测试导入。将映射保持在一个地方。
4. **`@objcMembers` 呢？** 类级别导出——应用于所有成员，除非 `@nonobjc`。通过在 Swift 提取器中检查类的修饰符并默认每个成员的 `@objc` 性质来处理。

---

## 10. 完成标准（这样我们就知道何时停止）

阶段 1（Swift↔ObjC）在以下条件满足时完成：
- 所有三个第 5a 节语料库通过：纯语言探针不变；跨语言规范流程探针端到端找到路径；Agent A/B 显示使用 codegraph 时在 ≥1 次运行中 Read = 0，比不使用更快。
- 指南第 6 节中的覆盖矩阵行填入数字。
- CHANGELOG `[Unreleased]` 条目存在，以用户端语气编写。

每个后续阶段具有相同的形状——自己的第 5 节语料库、自己的矩阵行、自己的 CHANGELOG 条目——并且**在前一个阶段通过之前不发布**。半桥接在这里不是可选的；它们积极使 CodeGraph 在这些代码库上比根本没有桥接更糟糕。
