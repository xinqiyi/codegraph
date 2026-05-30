# 设计 + 状态：通用回调 / 观察者边合成

**状态：** 阶段 1-3 已实现并验证为**原型，未提交到 `main`**（截至 2026-05-22）。本文档是继续此项工作的交接材料。
**动机：** 弥补静态提取在观察者 / 事件发射器 / 信号模式下留下的动态分发漏洞，在这些模式中，一个*分发器*通过共享存储调用在其他地方注册的回调——这样诸如"更新如何到达屏幕"之类的流程实际上存在于图中。

---

## TL;DR（太长不读）给新的会话

我们合成静态解析遗漏的 `dispatcher → callback` 边。它有效：

- **字段观察者**（excalidraw `Scene.onUpdate`/`triggerUpdate`）：合成 `triggerUpdate → triggerRender`。`trace(mutateElement, triggerRender)` 现在 = 3 跳。
- **EventEmitter**（express `on('mount', …)`/`emit('mount')`）：合成 `use → onmount`。
- **精度很高：** excalidraw 在 27k 条边中获得了 **1** 条合成边（正确的那条）；阶段 3 后节点计数增加了 +3（没有爆炸）。

**涉及的文件（均未提交到 `main`）：**
- `src/resolution/callback-synthesizer.ts` — 全图合成遍历（阶段 1 + 2）。
- `src/resolution/index.ts` — 在 `resolveAndPersistBatched()` 末尾调用 `synthesizeCallbackEdges()`（在基础边持久化之后）+ 导入。
- `src/extraction/tree-sitter.ts` — `visitFunctionBody` 现在提取**命名的**嵌套函数（阶段 3），因此内联的命名处理器成为可链接的节点。

**如何复现 / 测试：**
```bash
npm run build
rm -rf /tmp/codegraph-corpus/excalidraw/.codegraph
( cd /tmp/codegraph-corpus/excalidraw && codegraph init -i )
# 合成边（provenance='heuristic', metadata.synthesizedBy in {callback,event-emitter}）：
sqlite3 /tmp/codegraph-corpus/excalidraw/.codegraph/codegraph.db \
  "select s.name||' → '||t.name||'  '||coalesce(e.metadata,'') from edges e \
   join nodes s on e.source=s.id join nodes t on e.target=t.id where e.provenance='heuristic';"
# 端到端追踪（使用开发探针）：
node scripts/agent-eval/probe-trace.mjs /tmp/codegraph-corpus/excalidraw triggerUpdate triggerRender
```
探针脚本（仅开发，位于 `scripts/agent-eval/`）：`probe-node.mjs`（符号 + 踪迹）、`probe-trace.mjs`（调用路径）、`probe-context.mjs`、`probe-explore.mjs`。EventEmitter 测试夹具位于 `/tmp/cb-fixture/bus.js`（临时的一一重新创建或移入 `__tests__/`）。

---

## 漏洞

```ts
class Scene {
  private callbacks = new Set<Callback>();
  onUpdate(cb: Callback) { this.callbacks.add(cb); }          // 注册器
  triggerUpdate() { for (const cb of this.callbacks) cb(); }  // 分发器
}
this.scene.onUpdate(this.triggerRender);                      // 注册站点
```

运行时边 `triggerUpdate → triggerRender` 在静态上不存在：`triggerUpdate` 的唯一字面调用是 `cb()`（匿名）。测量结果：`triggerUpdate` 的唯一被调用者是 `randomInteger`；`trace(triggerUpdate, triggerRender)` 返回无路径。

## 为什么是全图遍历，而不是 `FrameworkResolver.resolve()`

`resolve(ref)` 回答"这个**命名的**引用指向什么"，一次一个引用。回调边**没有要解析的引用**（`cb()` 是匿名的）并且需要**跨文件、多站点关联**（注册器、注册点、分发器）。因此它是一个全图遍历，在基础解析之后进行，语言级别（任何面向对象的观察者），位于 `src/resolution/callback-synthesizer.ts` — **不在** `frameworks/` 下。

> 另一个动态分发类的对应机制——**命名的**属性/描述符分发（例如 django `self._iterable_class(...)`)——是 `claimsReference` 钩子（`resolution/types.ts` + `resolution/index.ts` 预过滤器）+ 一个 `FrameworkResolver.resolve()`（`frameworks/python.ts` 中的 django ORM 解析器）。那个*确实*适合 `resolve()`，因为引用是命名的。两者都是同一个覆盖率工作的一部分；请参阅"相关工作"部分。

---

## 已实现的算法（以及与原始设计的分歧）

### 字段-观察者通道（`fieldChannelEdges`，阶段 1）
1. **候选者**通过方法/函数**名称**筛选——注册器 `^(on[A-Z]\w*|subscribe|addListener|addEventListener|register|watch|listen|addCallback)$`；分发器包含 `(emit|trigger|notify|dispatch|fire|publish|flush)`。
2. **通过主体确认**（通过 `ctx.readFile` + 切片节点行读取）：注册器有 `this.<F>.add|push|set(`；分发器有 `for (… of [Array.from(]this.<F>)` + 一个调用，或 `this.<F>.forEach(`。
3. **配对——分歧：** 设计说按*类*配对；实现按**相同文件 + 相同字段 `F`** 配对（文件作为类的代理——可靠地获取包含类更困难）。适用于常见的每类一个文件的情况；多类文件需重新审视。
4. **注册：** `queries.getIncomingEdges(registrar.id, ['calls'])` → 对于每个，在边行读取调用者的源码并**通过正则恢复参数**（`<registrarName>\s*\(\s*(?:this\.)?(\w+)`）。分歧：设计偏好 tree-sitter 重新解析；实现使用正则（仅命名引用——箭头/内联参数在此处被遗漏）。
5. **合成** `dispatcher → fn`（`getNodesByName(arg)` → 方法|函数）。上限为 `MAX_CALLBACKS_PER_CHANNEL = 40`。

### EventEmitter 通道（`eventEmitterEdges`，阶段 2）
- **面向文件的扫描**（`ctx.getAllFiles()` + `readFile`，对 `.emit(`/`.on(`/等进行子字符串预过滤）。`ON_RE` = `\.(?:on|once|addListener)\(\s*['"]([^'"]+)['"]\s*,\s*(?:function\s+(\w+)|(?:this\.)?(\w+))`；`EMIT_RE` = `\.(?:emit|fire|dispatchEvent)\(\s*['"]([^'"]+)['"]`。
- 分发器 = `emit('e')` 调用的**封闭函数**（`enclosingFn` 找到包含该行的最紧密的函数/方法/组件节点）。处理器 = 通过 `getNodesByName` 查找 on-handler 名称。
- 通过**事件名字面量**关联；合成分发器 → 处理器。
- **精度——分歧：** 设计提议了接收者类型匹配；实现使用**事件扇出上限**（`EVENT_FANOUT_CAP = 6`）——跳过有超过 6 个处理器或分发器的事件（通用名称如 `error`/`change` 在没有类型信息的情况下会过度链接）。

### 来源——分歧
`Edge.provenance` 是一个固定的枚举（`'tree-sitter'|'scip'|'heuristic'`），因此合成边使用 **`provenance: 'heuristic'`** + `metadata: { synthesizedBy: 'callback'|'event-emitter', via/event/field }`。设计的 `'callback-synthesis'` 来源和高/中/低**置信度级别未实现**——扇出上限 + 注册器名称唯一性 + 仅命名处理器替代作为精度保护。

### 阶段 3——内联回调提取（`tree-sitter.ts`）
在真实代码库上阻塞 EventEmitter 的真正原因：内联处理器（`on('mount', function onmount(){})`）不是**节点**，因此没有任何东西可以链接到它们。根因：`visitFunctionBody` *穿过*嵌套函数而没有提取它们。修复：在 `visitForCallsAndStructure` 中，当一个主体节点是 `functionType` 且 `extractName` 返回真实名称时，调用 `extractFunction`（它会提取该函数并遍历其自身的主体）并返回。**仅命名**——匿名箭头会落入现有的递归（因此它们内部的调用仍然归属于封闭函数）。这约束了它：excalidraw +3 个节点，没有爆炸，没有回归。

---

## 验证结果（实际）

| 代码库 | 结果 |
|---|---|
| excalidraw | 1 条合成边 `triggerUpdate → triggerRender`（27,214 条边中）；`trace(mutateElement, triggerRender)` = 3 跳；节点 9,286 → 9,289 |
| express | 阶段 3 后：`use → onmount` `{event-emitter, event:"mount"}`（`onmount` 现在在 `application.js:109` 被提取） |
| `/tmp/cb-fixture/bus.js` | `tick → handleRefresh`，`persist → handleSave`（命名方法 EventEmitter 处理器） |
| excalidraw / express | 没有阶段 1 回归；节点计数稳定 |

---

## 剩余工作（按优先级排序，供下一个会话使用）

1. **匿名箭头处理器**——`on('e', () => foo())` 仍然不产生边（没有节点，在阶段 3 中有意不提取）。修复是**合成器链接通过主体**：解析箭头的主体并链接 `dispatcher →（箭头内部的调用）`。剩余的最高召回收益；处理最常见的现代回调形式。
2. **连接到 `resolveAndPersist`**（增量同步）——合成目前仅在 `resolveAndPersistBatched`（完整索引）中运行。增量重新索引不会刷新合成边。
3. **EventEmitter 精度的接收者类型匹配**（替换/增强扇出上限）——使用 `type_of` 边，使得 `x.emit('change')` 仅在 `x`、`y` 是相同类型时才链接到 `y.on('change', fn)`。允许扇出上限放宽。
4. **Tree-sitter 参数恢复**（替换字段通道阶段 4 中的正则）——对箭头、多参数、换行包装的调用更健壮。
5. **单个回调字段**（`this.onChange = cb; … this.onChange()`）——标量存储变体的字段观察者；尚未构建。
6. **广泛的精度/召回审核**——在整个语料库上运行；统计每个代码库的合成边，抽查，确认在 EventEmitter 密集的代码库上没有爆炸。
7. **测试 + CHANGELOG**——该夹具已准备好作为合成器的 vitest 案例；为阶段 3（命名嵌套函数提取；确认其他语言不受影响——更改在共享遍历器中）添加提取器测试，为 django 端添加解析器测试。

## 边界情况 / 模型
- **跨实例的过度近似**是可接受的（可达性，而非实例精度）。`unregister`/`off` 被忽略。
- 合成边是**附加的**——从不替换静态边；工具可以通过 `provenance='heuristic'` + `metadata.synthesizedBy` 进行过滤。

## 相关工作（相同覆盖工作）
这是关闭动态分发覆盖的一半工作。`main` 上的其他成果：
- **命名属性/描述符解析器：** `claimsReference`（`resolution/types.ts`，`resolution/index.ts` 中的预过滤器）+ django ORM 解析器（`frameworks/python.ts`，`_iterable_class` → `ModelIterable.__iter__`）。
- **检索/UX 更改**（与覆盖分离）：`explore` 完整小文件 + 粘合修复，`node`-带踪迹，`codegraph_trace`，`context` 调用路径——全部在 `src/mcp/tools.ts` / `src/context/index.ts` 中。
- **完整调查上下文 + 发现：** auto-memory `project_codegraph_read_displacement`（为什么覆盖——而不是提示/钩子/新工具——是让代理使用 codegraph 而非 Read 的杠杆）。
