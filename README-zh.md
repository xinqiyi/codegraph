<div align="center">

# CodeGraph

### 为 Claude Code、Cursor、Codex、OpenCode、Hermes Agent、Gemini、Antigravity 和 Kiro 注入语义代码智能

**约节省 22% 成本 · 减少约 50% 工具调用 · 100% 本地化**

### [文档与网站 →](https://colbymchenry.github.io/codegraph/)

[![npm version](https://img.shields.io/npm/v/@colbymchenry/codegraph.svg)](https://www.npmjs.com/package/@colbymchenry/codegraph)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Self-contained](https://img.shields.io/badge/Node.js-bundled%20%C2%B7%20none%20required-brightgreen.svg)](https://nodejs.org/)

[![Windows](https://img.shields.io/badge/Windows-supported-blue.svg)](#supported-platforms)
[![macOS](https://img.shields.io/badge/macOS-supported-blue.svg)](#supported-platforms)
[![Linux](https://img.shields.io/badge/Linux-supported-blue.svg)](#supported-platforms)

[![Claude Code](https://img.shields.io/badge/Claude_Code-supported-blueviolet.svg)](#supported-agents)
[![Cursor](https://img.shields.io/badge/Cursor-supported-blueviolet.svg)](#supported-agents)
[![Codex](https://img.shields.io/badge/Codex-supported-blueviolet.svg)](#supported-agents)
[![opencode](https://img.shields.io/badge/opencode-supported-blueviolet.svg)](#supported-agents)
[![Hermes Agent](https://img.shields.io/badge/Hermes_Agent-supported-blueviolet.svg)](#supported-agents)
[![Gemini](https://img.shields.io/badge/Gemini-supported-blueviolet.svg)](#supported-agents)
[![Antigravity](https://img.shields.io/badge/Antigravity-supported-blueviolet.svg)](#supported-agents)
[![Kiro](https://img.shields.io/badge/Kiro-supported-blueviolet.svg)](#supported-agents)

</div>

## 快速开始

**无需 Node.js** — 一条命令即可获取适合您操作系统的构建版本：

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh

# Windows (PowerShell)
irm https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.ps1 | iex
```

已有 Node？使用 npm 即可（任何版本均可）：

```bash
npx @colbymchenry/codegraph        # 零安装，或：
npm i -g @colbymchenry/codegraph
```

<sub>CodeGraph 自带运行时 — 无需编译，无需原生构建，在所有平台上运行一致。交互式安装程序会自动配置您的代理 — Claude Code、Cursor、Codex CLI、opencode、Hermes Agent、Gemini CLI、Antigravity IDE、Kiro。</sub>

### 初始化项目

```bash
cd your-project
codegraph init -i
```

<sub>`codegraph init` 只创建本地的 `.codegraph/` 索引目录；加上 `-i`（`--index`）还会在同一个步骤中构建初始图。如果没有 `-i`，之后运行 `codegraph index` 来填充索引。</sub>

<div align="center">

![1_C_VYnhpys0UHrOuOgpgoyw](https://github.com/user-attachments/assets/f168182f-4d9a-44e0-94d7-08d018cc8a3a)

</div>

### 卸载

改变主意了？一条命令即可从所有已配置的代理中移除 CodeGraph：

```bash
codegraph uninstall
```

<sub>逆向安装过程 — 移除 CodeGraph 的 MCP 服务器配置、指令和权限，从每个已配置的代理中剥离。您的项目索引（`.codegraph/`）保持不变；使用 `codegraph uninit` 逐项目移除。使用 `--target` 从特定代理移除，或使用 `--yes` 以非交互方式运行。</sub>

---

## 为什么选择 CodeGraph？

当 Claude Code 探索代码库时，它会启动 **Explore 代理**，这些代理使用 grep、glob 和 Read 扫描文件 — 每次工具调用都会消耗 token。

**CodeGraph 为这些代理提供了一个预索引的知识图谱** — 符号关系、调用图和代码结构。代理可以即时查询图谱，而不是扫描文件。

### 基准测试结果

在 **7 个真实世界开源代码库**（涵盖 7 种语言）上进行了测试，比较了一个代理（Claude Code，无头模式）回答一个架构问题时 **使用** 和 **不使用** CodeGraph 的表现。每个单元格是每组 4 次运行的中位数节省量。_在 Opus 4.8（2026-05-29）上重新验证，使用自适应 `codegraph_explore` 大小调整的构建版本。_

> **平均：节省 22% 成本 · 减少 47% token · 快 20% · 减少 50% 工具调用**

| 代码库 | 语言 | 成本 | Token | 时间 | 工具调用 |
|----------|----------|------|--------|------|------------|
| **VS Code** | TypeScript · ~1 万文件 | 节省 13% | 减少 63% | 快 11% | 减少 82% |
| **Excalidraw** | TypeScript · ~640 | 节省 40% | 减少 71% | 快 51% | 减少 82% |
| **Django** | Python · ~3 千 | 节省 9% | 减少 35% | 快 7% | 减少 38% |
| **Tokio** | Rust · ~790 | 节省 31% | 减少 59% | 快 29% | 减少 61% |
| **OkHttp** | Java · ~645 | 节省 4% | 减少 16% | 快 11% | 减少 40% |
| **Gin** | Go · ~110 | 节省 28% | 减少 40% | 快 25% | 减少 35% |
| **Alamofire** | Swift · ~110 | 节省 32% | 减少 43% | 快 6% | 减少 13% |

CodeGraph 在每个代码库上都减少了 **工具调用和总 token 数**，并且对大型代码库实现了 **零文件读取**，而未使用 CodeGraph 的代理则将其预算花费在 grep/find/Read 发现上。**每个代码库现在都更便宜，而不仅仅是更快** — 此前两个成本异常值（Django 和 OkHttp，其答案跨越同一接口的多个可互换实现）从比原生搜索更贵转变为更便宜，因为自适应 `codegraph_explore` 大小调整不再提供每个兄弟节点的完整代码体。在最小的代码库上，差距仍然最小（现代模型的原生搜索已经很便宜），但在所有代码库上始终保持正向收益；最大的收益仍然是更少的工具调用和更快的答案。

<details>
<summary><strong>各代码库详细对比 — 使用 vs 不使用（4 次运行中位数）</strong></summary>

**VS Code** · ~1 万文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 1 分 58 秒 | 2 分 13 秒 | 快 11% |
| 文件读取 | 0 | 8 | -8 |
| Grep/Bash | 0 | 9 | -9 |
| 工具调用 | 3 | 17 | 减少 82% |
| 总 token | 607k | 1.65M | 减少 63% |
| 成本 | $0.66 | $0.76 | 节省 13% |

**Excalidraw** · ~640 文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 1 分 23 秒 | 2 分 48 秒 | 快 51% |
| 文件读取 | 0 | 11 | -11 |
| Grep/Bash | 0 | 9 | -9 |
| 工具调用 | 4 | 20 | 减少 82% |
| 总 token | 596k | 2.06M | 减少 71% |
| 成本 | $0.53 | $0.89 | 节省 40% |

**Django** · ~3 千文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 1 分 43 秒 | 1 分 51 秒 | 快 7% |
| 文件读取 | 5 | 10 | -5 |
| Grep/Bash | 0 | 4 | -4 |
| 工具调用 | 8 | 13 | 减少 38% |
| 总 token | 752k | 1.16M | 减少 35% |
| 成本 | $0.56 | $0.62 | 节省 9% |

**Tokio** · ~790 文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 2 分 3 秒 | 2 分 53 秒 | 快 29% |
| 文件读取 | 3 | 9 | -6 |
| Grep/Bash | 0 | 7 | -7 |
| 工具调用 | 7 | 17 | 减少 61% |
| 总 token | 869k | 2.14M | 减少 59% |
| 成本 | $0.63 | $0.92 | 节省 31% |

**OkHttp** · ~645 文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 1 分 18 秒 | 1 分 27 秒 | 快 11% |
| 文件读取 | 2 | 4 | -2 |
| Grep/Bash | 0 | 4 | -4 |
| 工具调用 | 5 | 8 | 减少 40% |
| 总 token | 739k | 883k | 减少 16% |
| 成本 | $0.54 | $0.56 | 节省 4% |

**Gin** · ~110 文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 1 分 8 秒 | 1 分 30 秒 | 快 25% |
| 文件读取 | 0 | 3 | -3 |
| Grep/Bash | 0 | 5 | -5 |
| 工具调用 | 6 | 9 | 减少 35% |
| 总 token | 532k | 887k | 减少 40% |
| 成本 | $0.36 | $0.50 | 节省 28% |

**Alamofire** · ~110 文件
| 指标 | 使用 CodeGraph | 不使用 CodeGraph | 变化 |
|---|---|---|---|
| 时间 | 2 分 19 秒 | 2 分 28 秒 | 快 6% |
| 文件读取 | 5 | 9 | -4 |
| Grep/Bash | 1 | 4 | -3 |
| 工具调用 | 11 | 12 | 减少 13% |
| 总 token | 1.22M | 2.14M | 减少 43% |
| 成本 | $0.71 | $1.04 | 节省 32% |

</details>

<details>
<summary><strong>完整基准测试详情</strong></summary>

**方法。** 每组是 `claude -p`（Claude Opus 4.8）在无头模式下运行，使用 `--strict-mcp-config`：**使用 CodeGraph** = 启用 CodeGraph 的 MCP 服务器，**不使用 CodeGraph** = 空的 MCP 配置。内置的 Read/Grep/Bash 对两者都可用。每个代码库相同的问题，**每组 4 次运行，报告的是中位数**。成本 = 运行的 `total_cost_usd`；Token = 处理的总 token 数（包括输入含缓存 + 输出）；时间 = 挂钟时间；工具调用 = 每次工具调用，包括模型生成的任何子代理中的调用。代码库以 `--depth 1` 克隆，并由提供服务的同一 CodeGraph 构建版本索引。2026-05-29 在使用自适应 `codegraph_explore` 大小调整的构建版本上重新验证。这些数字低于之前的 Opus 4.7 验证 — 不是 CodeGraph 的回归，而是更强的原生基线：Opus 4.8 在主线程上高效地执行 grep/read，而不是分散到大型 Explore 子代理扫描中，因此不使用 CodeGraph 的那组比以前更精简。各代码库的数字会因不使用 CodeGraph 组的波动程度而有所不同（4 次运行的中位数将其平滑，但尾部仍然存在 — 例如 Django 的不使用 CodeGraph 组在一次运行中达到了 $2.71/14 分钟）。

**问题：**
| 代码库 | 问题 |
|----------|-------|
| VS Code | "扩展主机如何与主进程通信？" |
| Excalidraw | "Excalidraw 如何渲染和更新画布元素？" |
| Django | "Django 的 ORM 如何从 QuerySet 构建并执行查询？" |
| Tokio | "tokio 如何在其运行时上调度和执行异步任务？" |
| OkHttp | "OkHttp 如何通过其拦截器链处理请求？" |
| Gin | "gin 如何通过其中间件链路由请求？" |
| Alamofire | "Alamofire 如何构建、发送和验证请求？" |

**为什么 CodeGraph 胜出：** 有了索引，代理可以直接回答 — 使用 `codegraph_context` 来定位区域，然后一个 `codegraph_explore` 获取相关源码 — 然后就完成了，通常零文件读取。没有它，代理在发现（find/ls/grep）上花费大部分预算，然后才读取正确的代码。CodeGraph 只有在被**直接**查询时才有帮助，因此其指令引导代理直接回答，而不是将探索委托给文件读取子代理 — 否则子代理无论如何都会读取文件，而 CodeGraph 就成了额外的开销。

</details>

---

## 主要特性

| | |
|---|---|
| **智能上下文构建** | 一次工具调用即可返回入口点、相关符号和代码片段 — 无需昂贵的探索代理 |
| **全文搜索** | 通过 FTS5 在整个代码库中按名称即时查找代码 |
| **影响分析** | 在进行更改之前，追踪任何符号的调用者、被调用者以及完整的影响范围 |
| **始终新鲜** | 文件监视器使用原生 OS 事件（FSEvents/inotify/ReadDirectoryChangesW）并带防抖自动同步 — 图谱随您编码保持最新，零配置 |
| **20+ 种语言** | TypeScript, JavaScript, Python, Go, Rust, Java, C#, PHP, Ruby, C, C++, Objective-C, Swift, Kotlin, Dart, Lua, Luau, Svelte, Liquid, Pascal/Delphi |
| **框架感知路由** | 识别 Web 框架路由文件，并将 URL 模式链接到其处理程序，覆盖 14 个框架 |
| **混合 iOS / React Native / Expo** | 弥合静态解析无法跨越的跨语言流：Swift ↔ ObjC 桥接、React Native 传统桥 + TurboModules + Fabric 视图组件、原生 → JS 事件发射器、Expo Modules |
| **100% 本地化** | 没有数据离开您的机器。无需 API 密钥。无需外部服务。仅需 SQLite 数据库 |

<details>
<summary><strong>自动同步的工作原理 — 以及为什么您不需要手动运行 <code>codegraph sync</code></strong></summary>

当您的代理（Claude Code、Cursor、Codex、opencode）启动 `codegraph serve --mcp` 时，三层机制确保索引与您的代码保持同步 — 并确保代理在编辑和下一次同步之间的短暂窗口中永远不会收到错误的旧答案：

1. **带防抖自动同步的文件监视器。** 一个原生的 FSEvents / inotify / ReadDirectoryChangesW 监视器捕获每个源文件的创建/修改/删除，并在防抖窗口后触发重新索引（默认 `2000ms`，可通过 `CODEGRAPH_WATCH_DEBOUNCE_MS` 调整，限制在 `[100ms, 60s]` 之间）。编辑爆发会合并为一次同步。

2. **逐文件陈旧性提示。** 在短暂的防抖窗口期间，MCP 工具响应如果引用了一个仍待处理的文件，会添加一个 `⚠️` 横幅，命名该文件并告诉代理直接 `Read` 它。未被响应引用的待处理文件会以一个小页脚呈现。无论哪种方式，代理都会收到明确的信号 — 已在 Claude Code 中验证，代理在打开文件之前会明确说"直接读取文件以获取实时内容"。

3. **连接时的追赶同步。** 当 MCP 服务器（重新）连接时，codegraph 在回答第一个查询之前，会对工作目录运行一次快速 `(size, mtime)` + 内容哈希比对 — 这样在 MCP 服务器未运行时所做的编辑（终端的 `git pull`、其他编辑器的编辑、前一个已退出的代理会话）会在下一个会话的第一次工具调用时被吸收。

```
agent writes src/Widget.ts
  → watcher fires (<100ms)
  → debounce (default 2s)
  → sync; Widget.ts is in the index
  → next agent query sees it
```

**随时验证** 使用 `codegraph_status`（通过 MCP）或 `codegraph status`（CLI）。如果有任何待处理的内容，您会看到一个 `### Pending sync:` 部分，列出文件名及其编辑时间。

需要手动 `codegraph sync` 的少数情况：监视器被禁用（沙盒环境，或 `CODEGRAPH_NO_DAEMON=1`），或者您在代理会话之外编写脚本操作索引，并希望在脚本开始时进行预同步。

→ 完整深入阅读请参见 [指南 → 索引项目](https://colbymchenry.github.io/codegraph/guides/indexing/#stay-fresh-automatically)。

</details>

---

## 框架感知路由

CodeGraph 检测 Web 框架路由文件，并通过 `references` 边将 `route` 节点链接到其处理程序类或函数。查询视图/控制器的调用者现在会显示绑定它的 URL 模式。

| 框架 | 识别的形态 |
|---|---|
| **Django** | `path()`, `re_path()`, `url()`, `include()` in `urls.py`（CBV `.as_view()`，点分隔路径） |
| **Flask** | `@app.route('/path', methods=[...])`，蓝图路由 |
| **FastAPI** | `@app.get(...)`, `@router.post(...)`，所有标准方法 |
| **Express** | `app.get(...)`, `router.post(...)`，带中间件链 |
| **NestJS** | `@Controller` + `@Get/@Post/...`，GraphQL `@Resolver` + `@Query/@Mutation`, `@MessagePattern`/`@EventPattern`, `@SubscribeMessage` |
| **Laravel** | `Route::get()`, `Route::resource()`, `Controller@action`，元组语法 |
| **Drupal** | `*.routing.yml` 路由（`_controller`, `_form`，实体处理器）；`.module`/`.theme`/`.install`/`.inc` 中的 `hook_*` 实现 |
| **Rails** | `get '/x', to: 'users#index'`，哈希火箭 `=>` 语法 |
| **Spring** | 方法上的 `@GetMapping`, `@PostMapping`, `@RequestMapping` |
| **Gin / chi / gorilla / mux** | `r.GET(...)`, `router.HandleFunc(...)` |
| **Axum / actix / Rocket** | `.route("/x", get(handler))` |
| **ASP.NET** | 操作方法上的 `[HttpGet("/x")]` 属性 |
| **Vapor** | `app.get("x", use: handler)` |
| **React Router** / **SvelteKit** | 路由组件节点 |

---

## 混合 iOS / React Native / Expo 桥接

真实的 iOS 和 React Native 代码库跨越多种语言 — Swift 调用者调用一个已自动桥接的 Objective-C 选择器，JS 文件通过 React Native 桥调用原生模块，JSX 组件委托给原生视图管理器。静态 tree-sitter 提取在每个语言边界处停止。CodeGraph 将它们桥接起来，使得 `trace`、`callers`、`callees` 和 `impact` 能够跨边界端到端连接。

| 边界 | JS / Swift 端 | 原生端 | 方式 |
|---|---|---|---|
| **Swift → ObjC** | Swift `obj.foo(bar:)` | ObjC 选择器 `-fooWithBar:` | `@objc` 自动桥接规则（包括 init/property/protocol 形式）+ Cocoa 介词前缀（`With`/`For`/`By`/`In`/`On`/`At`/…） |
| **ObjC → Swift** | ObjC `[obj fooWithBar:]` | Swift `@objc func foo(bar:)` | 反向桥接名称候选；从源码验证 `@objc` 暴露 |
| **React Native 传统桥** | JS `NativeModules.X.fn(...)` | ObjC `RCT_EXPORT_METHOD` / `RCT_REMAP_METHOD` · Java/Kotlin `@ReactMethod` | 解析宏/注解声明以构建 JS 名称到原生方法的映射 |
| **React Native TurboModules** | JS `import M from './NativeM'; M.fn(...)` | 符合 Codegen 规范的原生实现 | 将 `Native<X>.ts` 规范接口视为真实来源 |
| **RN 原生 → JS 事件** | JS `new NativeEventEmitter(...).addListener('e', cb)` | ObjC `[self sendEventWithName:@"e" body:...]` · Swift `sendEvent(withName: "e", ...)` · Java/Kotlin `.emit("e", ...)` | 以字面事件名称为键的合成跨语言事件通道 |
| **Expo Modules** | JS `requireNativeModule('X').fn(...)` | Swift / Kotlin `Module { Name("X"); AsyncFunction("fn") { ... } }` | 解析 Expo DSL 字面量；合成方法节点通过现有名称匹配解析 |
| **Fabric 视图组件** | JSX `<MyView prop={v}/>` | TS Codegen 规范 + 原生实现类 | 规范 → `component` 节点；基于约定的名称+后缀查找（`View`/`ComponentView`/`Manager`/`ViewManager`）桥接到原生 |
| **传统 Paper 视图管理器** | JSX `<MyView prop={v}/>` | ObjC `RCT_EXPORT_VIEW_PROPERTY` · Java/Kotlin `@ReactProp` | 与 Fabric 相同 — Paper 时代的声明也会产生 `component` + `property` 节点 |

**在真实代码库上验证**（每个桥接方式的小/中/大型）：

| 桥接方式 | 小型 | 中型 | 大型 |
|---|---|---|---|
| Swift ↔ ObjC | [Charts](https://github.com/danielgindi/Charts) | [realm-swift](https://github.com/realm/realm-swift) | [Wikipedia-iOS](https://github.com/wikimedia/wikipedia-ios) |
| RN 传统桥 | [AsyncStorage](https://github.com/react-native-async-storage/async-storage) | [react-native-svg](https://github.com/software-mansion/react-native-svg) | [react-native-firebase](https://github.com/invertase/react-native-firebase) |
| RN 原生 → JS 事件 | [RNGeolocation](https://github.com/Agontuk/react-native-geolocation-service) | — | react-native-firebase |
| Expo Modules | expo-haptics | expo-camera | Expo SDK 扫描（7 个包） |
| Fabric / Paper 视图 | [react-native-segmented-control](https://github.com/react-native-segmented-control/segmented-control) | [react-native-screens](https://github.com/software-mansion/react-native-screens) | [react-native-skia](https://github.com/Shopify/react-native-skia) |

每个桥接方式发出的边标记为 `provenance:'heuristic'`，`metadata.synthesizedBy` 设置为稳定的通道名称（例如 `swift-objc-bridge`、`rn-event-channel`、`fabric-native-impl`、`expo-module-extract`），以便代理一目了然地了解每一步是如何进入图谱的。

---

## 快速入门指南

### 1. 运行安装程序

```bash
npx @colbymchenry/codegraph
```

安装程序将：
- 询问要配置哪个/哪些代理 — 自动检测已安装的代理：**Claude Code**、**Cursor**、**Codex CLI**、**opencode**、**Hermes Agent**、**Gemini CLI**、**Antigravity IDE**、**Kiro**
- 提示将 `codegraph` 安装到 PATH（以便代理可以启动 MCP 服务器）
- 询问配置适用于所有项目还是仅当前项目
- 为每个选定的代理写入 MCP 服务器配置（codegraph 使用指南由 MCP 服务器本身提供，因此不会向 `CLAUDE.md` / `AGENTS.md` 等添加指令文件）
- 当 Claude Code 是目标之一时，设置自动允许权限
- 初始化您当前的项目（仅限本地安装）

**非交互式（脚本/CI）：**

```bash
codegraph install --yes                              # 自动检测代理，全局安装
codegraph install --target=cursor,claude --yes       # 指定目标列表
codegraph install --target=auto --location=local     # 检测到的代理，项目本地
codegraph install --print-config codex               # 打印配置片段，不写入文件
```

| 标志 | 值 | 默认值 |
|---|---|---|
| `--target` | `auto`, `all`, `none`, 或逗号分隔列表（`claude,cursor,...`） | 提示 |
| `--location` | `global`, `local` | 提示 |
| `--yes` | （布尔值） | 每步提示 |
| `--no-permissions` | （布尔值）跳过 Claude 自动允许列表 | 权限开启 |
| `--print-config <id>` | 为单个代理转储配置片段并退出 | — |

### 2. 重启您的代理

重启您的代理（Claude Code / Cursor / Codex CLI / opencode / Hermes Agent / Gemini CLI / Antigravity IDE / Kiro）以便加载 MCP 服务器。

### 3. 初始化项目

```bash
cd your-project
codegraph init -i
```

构建每个项目的知识图谱索引。一次全局 `codegraph install` 即可在您打开的每个项目中使用 — 无需为每个项目重新运行安装程序。

就是这样 — 当存在 `.codegraph/` 目录时，您的代理将自动使用 CodeGraph 工具。

<details>
<summary><strong>手动设置（备用方案）</strong></summary>

**全局安装：**
```bash
npm install -g @colbymchenry/codegraph
```

**添加到 `~/.claude.json`：**
```json
{
  "mcpServers": {
    "codegraph": {
      "type": "stdio",
      "command": "codegraph",
      "args": ["serve", "--mcp"]
    }
  }
}
```

**添加到 `~/.claude/settings.json`（可选，用于自动允许）：**
```json
{
  "permissions": {
    "allow": [
      "mcp__codegraph__codegraph_search",
      "mcp__codegraph__codegraph_context",
      "mcp__codegraph__codegraph_callers",
      "mcp__codegraph__codegraph_callees",
      "mcp__codegraph__codegraph_impact",
      "mcp__codegraph__codegraph_node",
      "mcp__codegraph__codegraph_status",
      "mcp__codegraph__codegraph_files"
    ]
  }
}
```

</details>

<details>
<summary><strong>代理工具指南</strong></summary>

CodeGraph 的 MCP 服务器会**自动**在 MCP `initialize` 响应中向您的代理提供使用指南 — 无需管理指令文件，也不会向您的 `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` 添加任何内容。简而言之，它告诉代理：

- **直接使用 CodeGraph 回答结构性问题** — 它就是预构建的索引，因此 grep/read 循环只是重复已经完成的工作。将返回的源代码视为已读取。
- **根据意图选择工具：** `codegraph_context` 用于定位区域，`codegraph_trace` 用于"X 如何到达 Y"，`codegraph_explore` 用于调查多个符号，`codegraph_search` 用于查找符号，`codegraph_callers`/`codegraph_callees` 用于遍历调用流，`codegraph_impact` 用于编辑前分析，`codegraph_node` 用于获取单个符号的源代码。
- **信任结果 — 不要用 grep 重新验证**，并在编辑后检查陈旧性横幅。
- 如果 `.codegraph/` 尚不存在，建议运行 `codegraph init -i`。

确切文本位于 `src/mcp/server-instructions.ts` — 这是唯一真实来源。

</details>

---

## 工作原理

```
┌───────────────────────────────────────────────────────────────────┐
│                           Claude Code                             │
│                                                                   │
│   "一个请求如何到达数据库？"                                        │
│       直接调用 CodeGraph 工具 — 无需 Explore 子代理                │
│                                 │                                 │
└─────────────────────────────────┬─────────────────────────────────┘
                                  │
                                  ▼
┌───────────────────────────────────────────────────────────────────┐
│                       CodeGraph MCP 服务器                        │
│                                                                   │
│       context · trace · explore · callers · callees · impact      │
│                                 │                                 │
│                                 ▼                                 │
│                        SQLite 知识图谱                             │
│          符号 · 边 · 文件 · FTS5 全文搜索                          │
└───────────────────────────────────────────────────────────────────┘
```

1. **提取** — [tree-sitter](https://tree-sitter.github.io/) 将源代码解析为 AST。特定于语言的查询提取节点（函数、类、方法）和边（调用、导入、继承、实现）。

2. **存储** — 所有内容进入本地 SQLite 数据库（`.codegraph/codegraph.db`），支持 FTS5 全文搜索。

3. **解析** — 提取后，解析引用：函数调用 → 定义，导入 → 源文件，类继承，以及特定于框架的模式。

4. **自动同步** — MCP 服务器使用原生 OS 文件事件监视您的项目。更改会经过防抖处理（2 秒静默窗口），仅过滤源文件，并进行增量同步。图谱随您编码保持最新 — 无需配置。

---

## CLI 参考

```bash
codegraph                         # 运行交互式安装程序
codegraph install                 # 运行安装程序（显式）
codegraph uninstall               # 从代理中移除 CodeGraph（安装的逆操作）
codegraph init [path]             # 在项目中初始化（添加 --index 同时索引）
codegraph uninit [path]           # 从项目中移除 CodeGraph（添加 --force 跳过确认）
codegraph index [path]            # 完整索引（--force 重新索引，--quiet 减少输出）
codegraph sync [path]             # 增量更新
codegraph status [path]           # 显示统计信息
codegraph query <search>          # 搜索符号（--kind, --limit, --json）
codegraph files [path]            # 显示文件结构（--format, --filter, --max-depth, --json）
codegraph context <task>          # 为 AI 构建上下文（--format, --max-nodes）
codegraph callers <symbol>        # 查找调用某函数/方法的内容（--limit, --json）
codegraph callees <symbol>        # 查找某函数/方法调用的内容（--limit, --json）
codegraph impact <symbol>         # 分析更改符号会影响哪些代码（--depth, --json）
codegraph affected [files...]     # 查找受更改影响的测试文件（见下文）
codegraph serve --mcp             # 启动 MCP 服务器
```

### `codegraph affected`

通过传递导入依赖关系，查找受更改的源文件影响的测试文件。

```bash
codegraph affected src/utils.ts src/api.ts         # 将文件作为参数传递
git diff --name-only | codegraph affected --stdin   # 从 git diff 中管道输入
codegraph affected src/auth.ts --filter "e2e/*"     # 自定义测试文件模式
```

| 选项 | 描述 | 默认值 |
|--------|-------------|---------|
| `--stdin` | 从标准输入读取文件列表 | `false` |
| `-d, --depth <n>` | 最大依赖遍历深度 | `5` |
| `-f, --filter <glob>` | 用于识别测试文件的自定义 glob | 自动检测 |
| `-j, --json` | 以 JSON 格式输出 | `false` |
| `-q, --quiet` | 仅输出文件路径 | `false` |

**CI/钩子示例：**

```bash
#!/usr/bin/env bash
AFFECTED=$(git diff --name-only HEAD | codegraph affected --stdin --quiet)
if [ -n "$AFFECTED" ]; then
  npx vitest run $AFFECTED
fi
```

---

## MCP 工具

作为 MCP 服务器运行时，CodeGraph 向 Claude Code 暴露以下工具：

| 工具 | 用途 |
|------|---------|
| `codegraph_search` | 在代码库中按名称查找符号 |
| `codegraph_context` | 为任务构建相关代码上下文 |
| `codegraph_trace` | 一次调用即可追踪两个符号之间的调用路径（"X 如何到达 Y"）— 每个跳点内联其代码体，追踪 grep 无法追踪的动态分发跳点（回调、React 重渲染、接口→实现） |
| `codegraph_callers` | 查找调用某函数的内容 |
| `codegraph_callees` | 查找某函数调用的内容 |
| `codegraph_impact` | 分析更改符号会影响哪些代码 |
| `codegraph_node` | 获取特定符号的详细信息（可选包含源代码） |
| `codegraph_explore` | 一次调用返回按文件分组的多个相关符号的源代码以及关系映射 |
| `codegraph_files` | 获取已索引的文件结构（比文件系统扫描更快） |
| `codegraph_status` | 检查索引健康状况和统计信息 |

---

## 库用法

```typescript
import CodeGraph from '@colbymchenry/codegraph';

const cg = await CodeGraph.init('/path/to/project');
// 或者：const cg = await CodeGraph.open('/path/to/project');

await cg.indexAll({
  onProgress: (p) => console.log(`${p.phase}: ${p.current}/${p.total}`)
});

const results = cg.searchNodes('UserService');
const callers = cg.getCallers(results[0].node.id);
const context = await cg.buildContext('fix login bug', { maxNodes: 20, includeCode: true, format: 'markdown' });
const impact = cg.getImpactRadius(results[0].node.id, 2);

cg.watch();   // 文件更改时自动同步
cg.unwatch(); // 停止监视
cg.close();
```

---

## 配置

没有任何配置 — CodeGraph 是零配置的，**无需编写或维护配置文件**。语言支持根据文件扩展名自动确定，无需为每种语言进行任何配置。

开箱即用地跳过的内容：

- **依赖、构建和缓存目录** — `node_modules`、`vendor`、`dist`、`build`、`target`、`.venv`、`Pods`、`.next` 以及每个[支持的栈](#supported-languages)中的类似目录 — 因此图谱是您的代码，而不是第三方噪音。即使没有 `.gitignore` 也是如此。
- **`.gitignore` 中的任何内容** — 在 git 仓库中通过 git 识别，在非 git 项目中通过直接读取 `.gitignore`（根目录和嵌套目录）识别。
- **大于 1 MB 的文件** — 生成的包、压缩的 JS、供应商代码块。

要排除其他内容，请将其添加到 `.gitignore`。要将默认排除的目录**重新包含**（比如您确实希望索引一个供应商依赖项），添加否定规则 — `!vendor/`。默认值统一应用，因此提交依赖项或构建目录不会强制将其纳入图谱；`.gitignore` 否定规则是显式的选择加入。

## 支持平台

每个版本都提供自包含构建（捆绑的 Node 运行时 — 无需编译），适用于所有三种桌面操作系统，同时支持 Intel/AMD（x64）和 ARM（arm64）：

| 平台 | 架构 | 安装方式 |
|----------|---------------|---------|
| Windows | x64, arm64 | PowerShell 安装程序或 npm |
| macOS | x64, arm64 | shell 安装程序或 npm |
| Linux | x64, arm64 | shell 安装程序或 npm |

请参见[快速开始](#get-started)了解一行安装命令。

## 支持的代理

交互式安装程序会自动检测并配置以下每个代理 — 配置 MCP 服务器（它提供自己的使用指南，因此不会写入指令文件）：

- **Claude Code**
- **Cursor**
- **Codex CLI**
- **opencode**
- **Hermes Agent**
- **Gemini CLI**
- **Antigravity IDE**
- **Kiro**

## 支持的语言

| 语言 | 扩展名 | 状态 |
|----------|-----------|--------|
| TypeScript | `.ts`, `.tsx` | 完全支持 |
| JavaScript | `.js`, `.jsx`, `.mjs` | 完全支持 |
| Python | `.py` | 完全支持 |
| Go | `.go` | 完全支持 |
| Rust | `.rs` | 完全支持 |
| Java | `.java` | 完全支持 |
| C# | `.cs` | 完全支持 |
| PHP | `.php` | 完全支持 |
| Ruby | `.rb` | 完全支持 |
| C | `.c`, `.h` | 完全支持 |
| C++ | `.cpp`, `.hpp`, `.cc` | 完全支持 |
| Objective-C | `.m`, `.mm`, `.h` | 部分支持（类、协议、方法、`@property`、`#import`、消息发送；`.mm` ObjC++ 可能解析不完整） |
| Swift | `.swift` | 完全支持 |
| Kotlin | `.kt`, `.kts` | 完全支持 |
| Scala | `.scala`, `.sc` | 完全支持（类、特质、方法、类型别名、Scala 3 枚举） |
| Dart | `.dart` | 完全支持 |
| Svelte | `.svelte` | 完全支持（脚本提取、Svelte 5 runes、SvelteKit 路由） |
| Vue | `.vue` | 完全支持（script + script-setup 提取、Nuxt 页面/API/中间件路由） |
| Liquid | `.liquid` | 完全支持 |
| Pascal / Delphi | `.pas`, `.dpr`, `.dpk`, `.lpr` | 完全支持（类、记录、接口、枚举、DFM/FMX 表单文件） |
| Lua | `.lua` | 完全支持（函数、带接收器的方法、局部变量、`require` 导入、调用边） |
| Luau | `.luau` | 完全支持（Lua 的所有内容，加上 `type`/`export type` 别名、类型化签名和 Roblox 实例路径 `require`） |

## 故障排除

**"CodeGraph 未初始化"** — 首先在项目目录中运行 `codegraph init`。

**索引速度慢** — 检查 `node_modules` 和其他大型目录是否被排除。使用 `--quiet` 减少输出开销。

**MCP 遇到 "database is locked"** — 当前版本不应出现此问题：CodeGraph 捆绑了自己的 Node 运行时，并使用 Node 内置的 `node:sqlite`（WAL 模式），其中并发读取永远不会阻塞写入。如果您仍然看到此问题：

- **您安装的是旧版本（0.9 之前）。** 重新安装以获取捆绑的运行时 — `curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh`（macOS/Linux），`irm https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.ps1 | iex`（Windows），或 `npm i -g @colbymchenry/codegraph@latest`。
- **`codegraph status` 显示 `Journal:` 不是 `wal`** — 无法在此文件系统上启用 WAL（常见于网络共享和 WSL2 `/mnt`），因此读取可能会阻塞写入。将项目（及其 `.codegraph/` 文件夹）移动到本地磁盘。

**MCP 服务器无法连接** — 确保项目已初始化/已索引，验证 MCP 配置中的路径，并检查 `codegraph serve --mcp` 是否能在命令行中正常工作。

**缺少符号** — MCP 服务器在保存时自动同步（等待几秒钟）。如有需要，手动运行 `codegraph sync`。检查文件的语言是否受支持，并且不在 `.gitignore` 或默认排除的目录中（例如 `node_modules`、`dist`）。

## Star 历史

<a href="https://www.star-history.com/?repos=colbymchenry%2Fcodegraph&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=colbymchenry/codegraph&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=colbymchenry/codegraph&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=colbymchenry/codegraph&type=date&legend=top-left" />
 </picture>
</a>

## 许可证

MIT

---

<div align="center">

**为 AI 编码代理而构建 — Claude Code、Cursor、Codex CLI、opencode、Hermes Agent、Gemini CLI、Antigravity IDE 和 Kiro**

[报告 Bug](https://github.com/colbymchenry/codegraph/issues) · [请求功能](https://github.com/colbymchenry/codegraph/issues)

</div>
