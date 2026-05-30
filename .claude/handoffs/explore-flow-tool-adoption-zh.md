---
name: explore-flow-tool-adoption
date: 2026-05-24 00:55
project: codegraph
branch: architectural-improvements
summary: 调查了为什么 codegraph 的读取节省不会转化为挂钟时间；根本原因是代理的工具选择（trace 使用不足）。交付了一系列修复；突破是 "explore-surfaces-flow"——这是第一个通过适配代理已在使用的工具而在真实代理运行中显现的机制。
---

# 交接：codegraph 检索——工具采用与 explore-surfaces-flow

## 从这里继续——先阅读此处
**当前状态：** 一项关于使用 codegraph 让代理更快回答流程问题的长期调查。`architectural-improvements` 上有 6 个提交（全部通过探针验证，套件绿色 815）。突破：**`codegraph_explore` 现在能从代理已传递的符号集合中呈现执行流程**（`PmsProductController getList PmsProductService list PmsProductServiceImpl` → 以 `getList → service-interface → impl` 引导输出，利用合成边）。这是整个过程中第一个在真实代理运行中实际显现的机制（spring-mall A/B：流程在两个运行中都显现了，读取从 2.0→1.5）——因为它适配了代理**已在使用**的工具，而不是试图让它使用 `trace`。

**立即下一步：** 用户正在权衡如何下一步提升工具使用质量（他们的开放性问题）。决定选择：（a）**扩展 explore-flow 以更可靠地显现**（spring-halo 的查询没有命名一个连接的同名链→无流程），（b）接受我们达到了模型行为天花板并**结束**，或（c）用户的想法——更好的工具描述*示例*（≈ 引导，根据证据杠杆率低）或一个*查询构建器工具*（增加一次调用 + 新工具采用问题）。我的看法：继续**适配正在使用的工具**（唯一有效的方法）；示例/新工具是"改变代理"的方向，整个会话中都失败了。

> 建议下一条消息："explore-flow 仅在 3 个仓库中的 2 个上显现了——深入调查为什么 spring-halo 的 explore 查询没有产生流程，并使其更可靠地显现"——或——"我们达到了模型行为天花板；让我们停止并为这个分支写 CHANGELOG/PR"

## 目标
让 AI 代理快速回答**流程问题**（"X 如何到达 Y"、请求→处理器→服务、状态→渲染）：~0 次 Read/Grep、少量 codegraph 调用、更低的挂钟时间。`codegraph_trace` 是最快的工具（1 次调用 = 完整路径），但代理使用不足。最终目标 = trace 的速度，无论代理如何到达那里。

## 关键发现（主线）
- **障碍是代理的工具选择，而非图谱。** 在整个矩阵中，codegraph 将读取减少了 -75%，但挂钟时间仅减少 -16%（`docs/benchmarks/codegraph-ab-matrix.md`）。下限是往返次数 + 合成轮次。代理可靠地调用 `context`/`explore`，很少调用 `trace`（37 个流程单元中 3 个）。完整分析：`docs/benchmarks/call-sequence-analysis.md`。
- **引导无法改变它**（臂 B/F/G，3 种措辞变体）：MCP `initialize` 指令 / 工具描述无法与 CLI `--append-system-prompt` 的显著程度匹配，而强制使用 trace 在其不连接的地方会导致退步。已恢复。
- **自足性有效**（已提交）：自足的 `trace`（跳转体 + 目标被调用者内联）让未引导的代理停止——但仅限于它调用 trace 时。
- **突破——适配代理正在使用的工具。** `explore` 的查询是一个覆盖流程的精确符号集合，因此 `explore` 在其命名的符号中**找到**调用路径并以它引导输出。第一个在真实运行中显现并减少读取的机制。
- **失败的做法：** 选项 1（context-surfaces-flow）——模糊的描述无法区分终点→自信地给出**错误**功能的流程；已恢复。trace 多源 BFS 对模糊名称的处理——同样给出错误功能；已恢复。

## 注意事项
- **同名消歧必须匹配 qualifiedName 的段，而非子串**（`src/mcp/tools.ts` 中的 `buildFlowFromNamedSymbols`）：`list` 是 `getList` 的子串→保留了每个 getList。在 `::`/`.` 上分割 `qualifiedName` 并匹配段。
- **BFS 必须将连续的未命名跳转上限设为 1**——全图 BFS 会在一个神级函数的扇出中迷失（excalidraw `render()` → 指针处理器 → mutateElement）。≤1 桥接可以跨越缺失的中间节点而不迷失。
- **`getCallees` 也返回非 `calls` 类型的边**（引用）——需要过滤 `c.edge.kind === 'calls'`。
- **解析器/合成器的更改需要干净的重建索引**：`rm -rf .codegraph && codegraph init -i`（init 的边计数仅包含 contains 类型——查询数据库获取真实计数）。explore-flow 的更改是查询时的（无需重建索引）。
- **n=2 A/B 有噪声**——报告范围/模式，永远不要从一个运行得出结论。前台的 `sleep` 被阻塞→使用 `run_in_background` 运行 A/B 批次。
- Java/Kotlin 的 `qualifiedName` 是 `Class::method`（因此 `matchesSymbol` 解析 `Class.method` 限定的 trace 端点——代理已传递这些）。

## 如何测试与验证
- 探针流程显现（无代理）：`node scripts/agent-eval/probe-explore.mjs <repo> "<SymbolA SymbolB SymbolC>"` → 查找 `## Flow` 部分。`probe-trace.mjs <repo> <from> <to>` 用于 trace。
- 合成器：`sqlite3 <repo>/.codegraph/codegraph.db "select count(*) from edges where json_extract(metadata,'$.synthesizedBy')='interface-impl'"`；节点数在重建索引前后保持稳定（合成仅添加边）。
- 代理 A/B（真实测试）：`bash scripts/agent-eval/run-arms.sh <repo> "<Q>" I <run>`（臂 I = body-trace 构建，无引导）。通过 `/tmp` 中的 `cmp2.mjs` 风格脚本解析。通过标准 = 流程显现（`flowShown=Y`）+ 读取 ≤ 基线。
- `npm test`（vitest，815 通过）；`__tests__/mcp-tool-allowlist.test.ts` 覆盖了允许列表。

## 仓库状态
- 分支 `architectural-improvements`，最后提交 `bafae81 feat(mcp): codegraph_explore surfaces the execution flow from its named symbols`。
- 未提交：干净（仅有未跟踪的 `.claude/handoffs/`）。
- 6 个会话提交：`eab5cf3` 自足 trace + `CODEGRAPH_MCP_TOOLS` 允许列表 · `a6183d7` 研究日志 + 臂测试框架 · `bde8c19` node/trace 行号 · `98baf41` Java/Kotlin 接口→实现合成器 · `6f3c468` 手册 · `bafae81` explore-surfaces-flow。
- 未推送/合并。无版本提升。CHANGELOG 的 `[Unreleased]` 已包含全部内容。

## 未解决的线程 / TODO
- [ ] **用户的开放性问题**（在下一轮回答）：更好的工具描述*示例* vs *查询构建器工具* vs 继续适配正在使用的工具。证据支持最后一个。
- [x] explore-flow 可靠性：现在解析限定的 TOKEN（`Class.method`）——代理最精确的输入之前被文件扩展名剥离丢弃了（`2765c3c`）。spring-halo 的发布流程是有意不呈现的——它是**响应式/协调器调度**（`publishPost` 调用 `ReactiveExtensionClient.get`/`awaitPostPublished`，而非 `PostService.publish`），因此没有静态调用链。这是下一个覆盖边界（响应式运行时——如 MediatR、Vue Proxy），不是 explore-flow 的 bug。
- [ ] 整个分支的发布准备（本弧线 + 之前的框架扫描）：CHANGELOG 版本块 + `package.json` 版本提升 + PR 到 main。发布仅通过 `.github/workflows/release.yml`——不要 `npm publish`。
- [ ] 前沿：MediatR（`_mediator.Send`→Handle）和 Vue/Compose 响应式运行时仍然是未桥接的动态调度。

## 近期记录（从旧到新）
### 第 N 轮 — "改进 A/B 矩阵；trace 有效，读取接近 0——还有什么？"
- 诊断：读取已达下限，挂钟时间下限 = 往返次数 + 合成。构建了 `seq-matrix.mjs`；发现 trace 采用率为 3/37。
### 第 N 轮 — "explore/context/trace 是否相互竞争？一个工具？"
- 消融臂 A–E（`run-arms.sh`/`arms-F.sh` + `CODEGRAPH_MCP_TOOLS` 允许列表）。explore = 68% 的有效载荷，承担负载；trace 路径范围化但采用不足；仅 trace 不够。
### 第 N 轮 — "原型体内联 trace + A/B"
- 臂 F：自足 trace 在附加提示引导下胜出。但引导不是一个可交付的渠道。
### 第 N 轮 — "移植引导 + 重新运行"
- 臂 G（3 种变体）全部相对于基线退步；臂 H（体 trace，无引导）≈ 基线。引导已恢复；体 trace + 行号 + 允许列表已提交。
### 第 N 轮 — "准备连通性（Spring 接口-DI）"
- 构建了 `interfaceOverrideEdges`（Java/Kotlin 接口→实现，重载感知）。探针：3 跳 trace 连接。但 A/B 为空——代理从未调用 trace。已提交（探针验证，采用门控）。
### 第 N 轮 — "使 context 呈现流程（选项 1）"
- 失败：模糊查询→错误功能的流程。已恢复。
### 第 N 轮 — "更改 explore 以在后端执行 trace"
- 胜出：explore 的查询是一个精确的符号集合。`buildFlowFromNamedSymbols`（同名段匹配 + ≤1 桥接）。探针完美（Spring + excalidraw 完整链）；A/B：流程呈现 + 适度读取减少。已提交 `bafae81`。
### 第 N 轮 — "更新记忆 + 交接；更好的示例 / 查询构建器工具怎么样？"
- 本交接 + 记忆更新。战略性答案待定（适配工具 > 改变代理）。
