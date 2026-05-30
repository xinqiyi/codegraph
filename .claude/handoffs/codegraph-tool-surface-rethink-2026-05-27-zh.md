---
name: codegraph-tool-surface-rethink-2026-05-27
date: 2026-05-27 15:11
project: codegraph
branch: feat/go-multi-module-trace-quality
summary: PR #494 多语言审计揭示了结构性的 ~$0.04-$0.08 小仓库 MCP 工具定义开销；用户转向质疑是否真的需要 codegraph_context / 5+ 工具——建议仅保留 `explore` + `trace`。
---

# 交接：codegraph 是否应缩减为仅 `explore` + `trace`？

## 从这里继续——先阅读此处
**当前状态：** PR #494（`feat/go-multi-module-trace-quality`，13 个提交，全部 1076 个测试通过）交付了 cosmos/etcd Go 项目的所有安全优化以及跨语言扩展（生成文件检测、IFACE_OVERRIDE_LANGS、同属内联、路径接近度、工具门控——文件数 <150 时限制为 5 个核心工具）。经验证明：将工具缩减到 5 个以下会导致每个小仓库退步（3 工具门控：cobra 损失 17→48%；1 工具门控：express -43% 的胜利翻转 +107% 的损失）。用户刚刚提出了正确的问题：**"为什么我们需要 codegraph_context，或者这么多庞大的工具？它真正需要的只是 explore，以及 trace——如果你想问我的话。"**

**立即下一步：** 将会话视为设计转向而非成本差距打地鼠的延续来开始下一个会话。正确的回应是一份专注而诚实的分析：每个工具实际上做了什么 explore + trace 单独做不到的事，codegraph_context 的价值在哪里成立（或不成立），以及从默认表面移除 context/search/node 实际会损失多少可衡量的流覆盖率。现在不要开始砍工具——先呈现分析。

> 建议下一条消息："请向我展示每个 codegraph_* 工具在真实的流程问题上实际做了什么 explore + trace 单独做不到的事，以及在我们最近的审计中代理实际选择了哪些工具。如果 context/search/node 不值得它们的位置，请提出砍掉它们，并在 cosmos-Q1 + etcd-Q1 + prometheus + cobra 各 n=2 上测量。"

## 目标
决定 codegraph 的 10 工具 MCP 表面是否应缩减为 ~2 个核心工具（explore + trace），如用户所提议。本次会话中的经验迭代表明，5 个被省略的"辅助"工具（callers、callees、impact、status、files）只在小仓库上增加成本，并不值得它们的位置。现在真正的问题是：**同样的逻辑是否适用于 context + search + node？** 如果是，codegraph 变成 2 个工具 + 更小的 MCP 表面 = 更低的固定提示开销 = 从根本上缩小小仓库的成本差距，而非修补它。如果不是，请指出它们做独特工作的具体流程。

## 关键发现（本次会话）

- **PR #494 状态**：13 个提交，全部 1076 个测试通过，https://github.com/colbymchenry/codegraph/pull/494。已推送：
  - 生成文件检测：`src/extraction/generated-detection.ts`（多语言模式，应用于 `findSymbol`/`findAllSymbols`/`handleSearch`/`handleExplore` 文件排序/`context/formatter.ts`）
  - Go gRPC 桥接：`src/resolution/callback-synthesizer.ts:341` 中的 `goGrpcStubImplEdges`（cosmos-sdk 上 467 条桥接边）
  - 跟踪失败内联 + 路径接近配对 + 弱规范路径惩罚 + 来自 TO 文件的同属内联：均在 `src/mcp/tools.ts` 的 `handleTrace` 中
  - `IFACE_OVERRIDE_LANGS` 从 `{java,kotlin}` 扩展为 `{java,kotlin,csharp,typescript,javascript,swift,scala}`；循环迭代 `class` 和 `struct` 类型
  - 工具定义修剪（~7KB → 5KB）在 `src/mcp/tools.ts`
  - 小仓库工具门控：`ToolHandler.getTools()` 在 `fileCount < 150` 时过滤为 5 个核心工具
  - 小层级 explore 预算在 `getExploreOutputBudget(fileCount < 150)`：总计 13K / 4 个文件 / `includeRelationships: true`
  - `handleContext` 默认 `maxNodes` 在 `fileCount < 150` 时从 20 降至 8
- **Cosmos Q1 翻转**：胜出（$0.257 vs $0.449，n=1；n=2 平均 $0.341 vs $0.350 持平）。突破在于 `inlineEndpoint` 的"TO 文件中的其他函数"同属——`msgServer.Send` 的实际被调用者 `k.Keeper.SendCoins` 是一个嵌入式接口调用树，tree-sitter 无法静态解析，因此静态 `getCallees` 只返回工具函数；*实际*流程位于 `x/bank/keeper/send.go` 的文件同伴中。见 `handleTrace` 约第 1430 行。
- **工具门控的经验下限**（n=2-3 审计）：
  - 5 个工具（search+context+node+explore+trace）= 当前设置，有效
  - 3 个工具（search+context+trace）= cobra 损失 17→48%，sinatra 损失 18→96%；当 node/explore 不可用时代理回退到 Read
  - 1 个工具（仅 search）= 灾难性，express 从 -43% 胜利 → +107% 损失
- **n=3 测量确认结构性下限：** cobra WITH 始终 $0.28（方差 <5%），WITHOUT 始终 $0.24。$0.04 的差距是结构性的，不是噪声。
- **用户的转向问题挑战了这一结论：** 他们的假设是 context+search+node 可能也是收入低于成本。我们现有的审计无法直接回答这个问题——每个测试都提供了所有 10 个（或 5 个）工具。要测试这一点，仅暴露 explore+trace 进行受控批量测试并重新测量。
- **跨语言状态（各单次运行）：** 胜出 = Go（多模块）、Rust、Java、C#、Kotlin、Swift、Svelte、prometheus、ky（后门控）、express（JS）。持平 = cobra（n=2 持平 $0.27/$0.27）、excalidraw、django、redis、json、Masonry、flutter、vapor、spring。失败 = sinatra、slim、flask、scala-play、Fusion、vue-core（方差）、Drupal、NestJS、FastAPI、Laravel、ASP.NET、axum、actix、Rocket、gorilla/mux、SvelteKit、Charts 桥接（轻微）、RN segmented-control（轻微）。
- **失败模式是结构性的，而非语言特定的。** 所有失败都是小型示例/入门仓库，其中无 codegraph 的 grep+read 路径成本约为 $0.20-0.30，而 codegraph 的 MCP 开销无法分摊。

## 注意事项

- **PR-494 标题是 Go 多模块 PR，但其内容现在已是跨领域的**——生成文件检测、IFACE_OVERRIDE_LANGS、工具门控，所有语言无关。不要让标题限制其内容范围。
- **WITHOUT 组的方差巨大**——同仓库单次运行成本可能在 $0.04 到 $0.80 之间波动，取决于该轮代理是大量使用 grep 还是 read。**永远不要从 n=1 得出结论。** 本次会话有许多单次运行结果需要确认。
- **Cobra（~50 个文件）是金丝雀**——每个有助于 ky 或 sinatra 的激进削减都至少使 cobra 退步一次。正因如此，它是测试最多的小仓库。
- **不要再尝试 1 工具或 3 工具门控**——两者都在 `getTools()` 注释中明确记录为回归（`src/mcp/tools.ts` 约第 660 行）。缩减到 5 个以下会迫使代理使用 Read。
- **Kong 的第一次审计是 0 字节索引**——并行的 `audit.sh` 运行针对同一个 .codegraph 目录可能会互相损坏。如果 kong/任何仓库的审计显示极其错误的数字，请在迭代结果前检查 `stat /tmp/codegraph-corpus/<repo>/.codegraph/codegraph.db`。
- **48 个并行审计启动会静默失败**——系统资源限制。最大保持在 6-8 个并行。使用 `wait` 在批次之间等待。
- **MCP 守护进程在进程启动时缓存工具列表**——当迭代 `getTools()` 时，你必须在重建之间执行 `pkill -f "codegraph.js serve --mcp"`，否则你将测试过时代码。
- **`maxCharsPerFile` 单调不变性**由 `__tests__/explore-output-budget.test.ts` 固定（规范是`更大的层级绝不能得到比更小层级更小的 maxCharsPerFile`）。请遵守它。

## 如何测试与验证

- `npm test` → "Tests 1076 passed | 2 skipped"。必须保持绿色。
- `npm run build 2>&1 | tail -3` → 检查 dist 是否重建干净。
- `pkill -f "codegraph.js serve --mcp" ; sleep 2` → 在 agent-eval 之前始终运行，否则守护进程会提供过时代码。
- 单问题审计：`AGENT_EVAL_OUT=/tmp/cg-NAME /Users/colby/Development/Personal/codegraph/scripts/agent-eval/run-all.sh <repo-path> "<question>" headless`。输出 `run-headless-with.jsonl` 和 `run-headless-without.jsonl`。
- 解析：`node scripts/agent-eval/parse-run.mjs /tmp/cg-NAME/run-headless-{with,without}.jsonl` → 成本、时长、轮次、工具序列。
- **要得出真实结论，至少 n=2。** n=3 是区分方差和信号的正确标准——上次会话中 cobra 的数据显示 WITH 方差 <5%，但 WITHOUT 波动达 95%。
- **用户想要的 explore + trace 实验**：修改 `getTools()` 以将可见工具过滤为 `new Set(['codegraph_explore', 'codegraph_trace'])` 适用于所有仓库（或先仅用于小层级），重新运行 cosmos-Q1、etcd-Q1、prometheus、cobra 各 n=2 并比较。

## 仓库状态

- 分支 `feat/go-multi-module-trace-quality`，最后提交 `ae5364c docs(mcp): pin empirical lower bound on tool gating after n=2 micro test`
- 未提交：干净
- PR：https://github.com/colbymchenry/codegraph/pull/494（13 个提交，准备审查，除非我们重新设计工具表面）

## 未解决的线程 / TODO

- [ ] **用户的转向**：证明或否定仅 explore + trace 是否足够。设置一个 4 仓库 × n=2 的批次（cosmos-Q1、etcd-Q1、prometheus、cobra），仅暴露 explore+trace，与当前 5 工具 / 10 工具的基线比较。
- [ ] 如果仅 explore+trace 胜出→全面削减工具表面。**这是一个破坏性 API 变更**——callers/callees/impact/status/files/node 将从默认暴露中消失。需要一种干净的方式为直接通过 MCP 编写脚本的用户保留它们（环境变量？`--full-tools` 标志？）。
- [ ] 如果仅 explore+trace 失败→确定 context/search/node 中哪一个在做结构性工作，并提出仅削减其他工具。
- [ ] **无论哪种方式都需要更新 README**：当前"~35% 更便宜"的声明平均了 7 个中大型仓库。要么将该范围限定为"真实代码库（~200+ 文件）"，要么在工具表面变更后重新测量。
- [ ] Liquid、Pascal/Delphi、React Router、TurboModules、Expo Modules、Paper view managers——仍然是 README 中未经测试的类别。桥接 Swift↔ObjC/RN-legacy/RN-events/Fabric 在第 3 波中测试过——1 胜、2 平、1 轻微失败。其余仍然是空白。
- [ ] 如果我们按现状交付 PR，在 `[Unreleased]` 下为 13 个提交写一条 CHANGELOG 条目——当前 CHANGELOG 条目涵盖了提交 1-2（生成文件检测 + gRPC 桥接 + trace UX）；提交 3-13 需要它们自己的条目。

## 近期记录（最近 ~10 轮，从旧到新）

### 第 N-9 轮 — 用户："我们是否按照手册测试了每种语言/框架？"
- **Claude：** 诚实审计了覆盖率——README 声明中约 70%（22 种语言 + 14 个框架 + 8 座桥接）尚未验证。提供了三种范围（快速/中等/完整手册）。
- **结果：** 用户选择了"完整手册"；后来细化为"仅快速测试完全未测试的类别"。

### 第 N-8 轮 — 第 1 波快速测试（8 并行：JS/Dart/Drupal/NestJS/FastAPI/Vapor/Spring/Rails）
- **Claude：** 通过 `run-all.sh` 启动了 8 个审计。结果：express 胜出（-43%），3 持平，4 失败（Drupal、NestJS、FastAPI、Rails）。NestJS 损失 113%，FastAPI 损失 194% 是最突出的。
- **结果：** 记录了失败模式与仓库大小相关，而非语言。

### 第 N-7 轮 — 第 2 波（7 框架：Laravel、ASP.NET、axum、actix、Rocket、gorilla/mux、SvelteKit）+ 第 3 波（4 桥接：Charts、async-storage、RN-geolocation、RN-segmented-control）
- **Claude：** 所有第 2 波框架在成本上失败（小型示例/入门仓库）。所有第 3 波桥接在成本上胜出/持平，并且在 Read 上胜出。
- **结果：** 确认失败模式是小仓库结构性的，而非框架特定的。桥接（codegraph 的标志性功能）有效。

### 第 N-6 轮 — 用户："我们有很多失败。改进它们直到全部清晰胜出。你不能停止、放弃或推脱。"
- **Claude：** 对 8 个最严重的失败运行了 n=3 以区分方差和信号。第一次尝试使用 48 个并行进程静默失败（系统限制）。以 6 个一波的方式重新运行。
- **结果：** n=3 cobra：WITH 始终 $0.28（方差 <5%），WITHOUT 始终 $0.24。**确认失败是结构性的，不是方差。**

### 第 N-5 轮 — 测试了 3 工具门控（仅 search + context + trace）
- **Claude：** 修改了 `getTools()` 以在 `<150` 文件时仅暴露 3 个工具。重新运行了 cobra/ky/sinatra。结果：cobra 损失 17→48%，ky 损失 47→42%，sinatra 损失 18→96%。已恢复。
- **结果：** 缩减到 5 个工具以下会导致退步，因为代理回退到额外的 Read。

### 第 N-4 轮 — 在微仓库（<50 文件）上测试了 1 工具门控（仅 search）
- **Claude：** 修改了门控。cobra 损失 126%，ky 损失 44%，express 从 -43% 胜利翻转 +107% 损失。灾难性的。已恢复。
- **结果：** 记录了经验下限：5 个工具。提交了 `ae5364c`，其中包含警告未来调优者不要重新争论的注释。

### 第 N-3 轮 — Claude 提出了三条前进路径（自动禁用 / README 诚实 / context 重写）
- **Claude：** 使用 AskUserQuestion 提供了在 <50 文件时自动禁用、诚实的 README 或最后的 context 缩减。
- **结果：** 用户转向了，没有选择其中任何一个。

### 第 N-2 轮 — 用户："为什么我们需要 codegraph_context，或者这么多庞大的工具？它真正需要的只是 explore，以及 trace——如果你想问我的话。"
- **Claude：** 认识到这是一个设计转向，而非成本差距迭代的延续。正确的下一步是实际测试用户的假设，而不是再写一条回复为现状辩护。
- **结果：** 本交接为新的会话捕获了转向，以便正确回答。

### 第 N-1 轮 — 用户：`/handoff save`
- **Claude：** 写入了此文件。
- **结果：** 交接已持久化。下个会话将阅读它并以测量而非观点的方式处理 explore+trace-only 的设计问题。
