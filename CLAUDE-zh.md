# CLAUDE.md

本文档为在此仓库中处理代码时的 Claude Code（claude.ai/code）提供指导。

## 项目概览

CodeGraph 是一个本地优先的代码智能库 + CLI + MCP 服务器。它使用 tree-sitter 解析任何受支持的代码库，将符号/边/文件存储在 SQLite（FTS5）中，并通过 MCP 向 AI 代理（Claude Code、Cursor、Codex CLI、opencode）暴露知识图谱。每个项目的数据存储在 `.codegraph/` 中。提取是确定性的 — 源自 AST，而非 LLM 摘要。

作为 `@colbymchenry/codegraph` 在 npm 上分发；同一个二进制文件同时作为安装程序、索引器和 MCP 服务器。

## 构建、测试、运行

```bash
npm run build           # tsc + 复制 schema.sql 和 *.wasm 到 dist/；chmods dist/bin/codegraph.js
npm run dev             # tsc --watch
npm run clean           # rm -rf dist

npm test                # vitest run（全部）
npm run test:watch
npm run test:eval       # 仅 __tests__/evaluation/
npm run eval            # 构建然后通过 tsx 运行 __tests__/evaluation/runner.ts

npm run cli             # 构建然后运行本地 dist 二进制文件

# 单个测试文件 / 模式
npx vitest run __tests__/installer-targets.test.ts
npx vitest run __tests__/extraction.test.ts -t "TypeScript"
```

`copy-assets`（从 `build` 调用）将 `src/db/schema.sql` 和所有 `src/extraction/wasm/*.wasm` 文件复制到 `dist/`。**任何新的 SQL 或语法 wasm 必须被复制，否则不会包含在发布中。**

Node 引擎：`>=18.0.0 <25.0.0`。在 Node 25.x 上会硬退出（参见 `src/bin/node-version-check.ts`）。

## 架构

### 分层流水线

```
files → ExtractionOrchestrator (tree-sitter) → DB (nodes/edges/files)
              ↓
       ReferenceResolver (imports, name-matching, framework patterns)
              ↓
       GraphQueryManager / GraphTraverser (callers, callees, impact)
              ↓
       ContextBuilder (markdown/JSON for AI consumption)
```

公共 API 表面是 `src/index.ts` — `CodeGraph` 类连接所有层并重新导出类型。库用户只接触此文件；MCP 服务器和 CLI 也驱动它。

### 模块布局

- `src/index.ts` — `CodeGraph` 类：`init`/`open`/`close`、`indexAll`、`sync`、`searchNodes`、`getCallers`/`getCallees`、`getImpactRadius`、`buildContext`、`watch`/`unwatch`。
- `src/db/` — `DatabaseConnection`、`QueryBuilder`（预编译语句）、`schema.sql`。由 `better-sqlite3`（原生）支持（可用时），透明回退到 `node-sqlite3-wasm`。`codegraph status` 显示哪个后端在运行；wasm 是慢速路径。
- `src/extraction/` — `ExtractionOrchestrator`、tree-sitter 包装器、`languages/` 下按语言的提取器（每种语言一个文件），以及非 tree-sitter 格式的独立提取器（`svelte-extractor.ts`、`vue-extractor.ts`、`liquid-extractor.ts`、Delphi 的 `dfm-extractor.ts`）。`parse-worker.ts` 将繁重解析放在主线程外运行。
- `src/resolution/` — `ReferenceResolver` 协调 `import-resolver.ts`（带有用于 tsconfig 路径别名 + cargo workspace 成员 glob 的 `path-aliases.ts`）、`name-matcher.ts` 和 `frameworks/`（Express、Laravel、Rails、FastAPI、Django、Flask、Spring、Gin、Axum、ASP.NET、Vapor、React Router、SvelteKit、Vue/Nuxt、Cargo workspaces）。框架发出 `route` 节点和 `references` 边。
- `src/graph/` — `GraphTraverser`（BFS/DFS、影响半径、路径查找）和 `GraphQueryManager`（高级查询）。
- `src/context/` — `ContextBuilder` + 用于 markdown/JSON 输出的格式化器。
- `src/search/` — 全文查询解析器和 FTS5 辅助工具。
- `src/sync/` — `FileWatcher`（原生 FSEvents/inotify/RDCW）带防抖 + 过滤，以及 git 钩子辅助工具。
- `src/mcp/` — MCP 服务器（`MCPServer`、`tools.ts`、`transport.ts`）。`server-instructions.ts` 是服务器在 MCP `initialize` 响应中返回的内容 — 保持与面向用户的工具指南同步。
- `src/installer/` — 见下文。
- `src/bin/codegraph.ts` — CLI（commander）。子命令：`install`、`init`、`uninit`、`index`、`sync`、`status`、`query`、`files`、`context`、`affected`、`serve --mcp`。
- `src/ui/` — 终端 UI（微光进度条、工作器）。

### NodeKind / EdgeKind

定义在 `src/types.ts` 中。提取器和解析器都必须使用这些精确的字符串。

- **NodeKind**：`file`、`module`、`class`、`struct`、`interface`、`trait`、`protocol`、`function`、`method`、`property`、`field`、`variable`、`constant`、`enum`、`enum_member`、`type_alias`、`namespace`、`parameter`、`import`、`export`、`route`、`component`。
- **EdgeKind**：`contains`、`calls`、`imports`、`exports`、`extends`、`implements`、`references`、`type_of`、`returns`、`instantiates`、`overrides`、`decorates`。

### 多代理安装程序

`src/installer/` 是 `codegraph install`（以及裸 `codegraph`/`npx @colbymchenry/codegraph` 调用）的入口点。架构：

- `targets/registry.ts` 列出每个支持的代理。
- `targets/types.ts` 定义 `AgentTarget` 接口 — 添加第 5 个代理（Continue、Zed、Windsurf…）只需 **在 `targets/` 中新建一个文件 + 在 `registry.ts` 中添加一条记录**。每个目标拥有自己的配置文件位置和 MCP 服务器 JSON/TOML/JSONC 写入。（目标不再写入指令文件 — 见下文。）
- 当前目标：`claude.ts`、`cursor.ts`、`codex.ts`、`opencode.ts`。
- `targets/toml.ts` 是一个手写的 TOML 序列化器，限定在 `[mcp_servers.codegraph]` 范围内（由 Codex 使用）。兄弟表和 `[[array_of_tables]]` 原样保留。无新依赖。
- opencode 默认读取 `opencode.jsonc`；安装程序优先使用现有的 `.jsonc`，回退到 `.json`，对于全新安装创建 `.jsonc`。编辑通过 `jsonc-parser` 精确进行，以便用户的注释和格式在安装/重新安装/卸载的往返过程中保持不变。
- `instructions-template.ts` 不再包含指令体 — 它只导出 `<!-- CODEGRAPH_START -->`/`<!-- CODEGRAPH_END -->` 标记。安装程序**已停止**向每个代理的指令文件（`CLAUDE.md` / `~/.codex/AGENTS.md` / `~/.config/opencode/AGENTS.md` / `~/.gemini/GEMINI.md` / `.cursor/rules/codegraph.mdc` / Kiro steering 文档）写入 `## CodeGraph` 块，因为它逐字复制了 MCP `initialize` 指令（问题 #529）。每个目标的 `install`（升级时自愈）和 `uninstall` 使用标记来**剥离**先前安装留下的块。`server-instructions.ts` 是面向代理指南的唯一真实来源。
- 所有安装程序更改需要在 `__tests__/installer-targets.test.ts` 中有匹配的测试覆盖 — 有大约 47 个参数化契约测试，涵盖安装幂等性、兄弟节点保留、卸载逆转安装、字节相等重新运行返回 `unchanged` 以及 Codex 的部分状态恢复。

### Cursor MCP 工作目录问题

Cursor 使用错误的 cwd 启动 MCP 子进程，并且不在 `initialize` 中传递 `rootUri`。安装程序将 `--path` 注入到 Cursor 的 MCP 参数中 — 本地安装使用绝对路径，全局安装使用 `${workspaceFolder}`。如果您触及 Cursor 的接线，请保留此行为。

### MCP 服务器指令

`src/mcp/server-instructions.ts` 在 MCP `initialize` 响应中发送回代理。这是每个代理看到的关于如何使用工具的**第一件事**，自问题 #529 起，它是面向代理工具指南的**唯一真实来源** — 安装程序不再向 `CLAUDE.md` / `AGENTS.md` / `.cursor/rules/codegraph.mdc` 写入重复的 `## CodeGraph` 指令块。在这里编辑工具指南，而不是其他地方。

## 检索性能与动态分发覆盖（不可退化）

CodeGraph 的核心价值是让代理用几次**快速**的 codegraph 调用和**零 Read/Grep** 来回答**结构性/流程性**问题（"X 如何到达 Y"、追踪、影响、调用者）。优化目标是**挂钟延迟 + 工具调用次数** — *不要为 token 成本进行优化*。（成本是**更低**的，而不是早期说法中的"持平"：当前构建在 7 个 README 仓库上进行的带与不带 A/B 测试，每组中位数 4，平均节省了 **35% 成本 · 57% token · 46% 时间 · 71% 工具调用** — 再现了已发布的 README。其机制是**在更小的累积上下文上进行的少得多的轮次** — 不是缓存能力：不带 CodeGraph 那组的大量 token 量*主要*是便宜的缓存读取，这就是为什么 token 数量节省（57%）看起来比成本节省（35%）更大。通过**对每轮助手使用量求和**来测量 token 数，而不是 `result.usage`（在当前 Claude Code 中仅为最后一轮）。参见 `docs/benchmarks/call-sequence-analysis.md`。）驱动这一切的机制是：**当 codegraph 的答案不够充分时，代理会立即回退到 Read/Grep。** 因此每个更改都通过一个问题来评判 — codegraph 的答案是否足够充分以*阻止*代理进行读取？

**目标行为：** 一个流程问题在小型代码库上通过 **1 次 codegraph 调用解决，扩展到大型代码库上 3-5 次**，**Read/Grep = 0**。在审查 PR 或尝试新功能时，不要对此造成退化。

### 使工具适应代理 — 不要试图改变代理

决定检索更改是否成功的杠杆。**在构建任何东西之前进行测试：这是否使代理_已经调用_的工具利用它_已经给出_的输入做更多事情？如果相反地需要代理以不同方式行事 — 选择不同的工具、以不同方式查询、从示例中学习 — 它将撞上低显著性墙并且不会成功。**

CodeGraph 影响代理的唯一渠道是低显著性的：MCP `initialize` 指令（`server-instructions.ts`）和工具描述。更改它们**不能**可靠地改变代理的工具_选择_或查询风格 — 已验证：将 trace-first 引导移植到 server-instructions + 工具描述（3 个措辞变体）从未再现 CLI `--append-system-prompt` 所达到的效果，并且相对基线**退化**了挂钟时间。新工具表现更差（很少被选择 — 代理甚至对 `trace` 也选择不足）；"更好的示例"是相同的引导。代理的工具选择确实会随着宿主模型在工具使用方面的改进而自行改善 — 但这并非我们可以强制实现的。

有效的是在代理当前所在的层面与它相遇：
- **充分性** — `codegraph_trace` 内联每个跳点的代码体 + 目标自身的被调用者，因此一次 trace 调用就结束了流程调查（无需后续的 explore/node/Read）。
- **explore-flow** — `codegraph_explore` 的查询是一个精确的符号名称集合（包括限定的 `Class.method`），涵盖代理所追求的流程；explore 在这些命名的符号中找到调用路径（利用合成边）并用它引领输出 — 通过代理可靠进行的调用提供 trace 质量的流程。（`buildFlowFromNamedSymbols`：片段/共名消歧；最多 1 个未命名桥接，因此它永远不会漫游到神级函数的扇出中。）

失败的是反向操作 — 将精确答案折叠到一个**模糊输入**工具中。`codegraph_context` 接收的是描述而不是符号，因此它无法消歧流程的端点并呈现_错误的功能_。精确输出需要精确输入。

此轴下的剩余杠杆是**覆盖率**：每个被静态连接的流程（一个新的动态分发合成器）然后自动被 explore-flow/`trace` 发现，无需代理更改。响应式/协调器运行时（Halo 的 `ReactiveExtensionClient`、MediatR、Vue Proxy）是前沿 — 那里的流程没有静态边，因此没有东西会被发现（正确的 — 沉默胜过错误）。完整调查 + A/B 记录：`docs/benchmarks/call-sequence-analysis.md`。

### 探索预算 — 保持两个预算随仓库大小单调递增

`src/mcp/tools.ts` 中的两个函数根据索引文件数来缩放 explore。这是预期的分辨率（这里的回归会静默地迫使代理回到 Read）：

| 仓库 | 文件数 | explore 调用 | 字符/调用 | 每文件 |
|---|---|---|---|---|
| express（小型） | 147 | 1 | 18K | 3800 |
| excalidraw/django（中型） | 643–3043 | 2 | 28K | 6500 |
| vscode（大型） | 10446 | 3 | 35K | 7000 |
| ~20k / ~40k | — | 4 / 5 | 38K | 7000 |

- `getExploreBudget(fileCount)` → **调用**预算：`<500→1, <5000→2, <15000→3, <25000→4, ≥25000→5`（最大 5）。
- `getExploreOutputBudget(fileCount)` → **每调用**输出（字符/文件/每文件）。**不变性：更大的层绝不能得到比更小的层更小的 `maxCharsPerFile`。**（促使此文档的退化：`<5000` 层的 2500 *低于* `<500` 层的 3800，因此在一个神级文件仓库上 — excalidraw 的 415 KB `App.tsx` — 一次 explore 返回了文件的 <1% 并迫使了 Read。）
- Explore 输出**绝不能告诉代理"使用 Read"** — 引导到另一个 `codegraph_explore` 并"将返回的源代码视为已读取"。

### 动态分发覆盖 — 流程必须在图中端到端存在

静态 tree-sitter 提取会遗漏计算/间接调用，因此流程在动态分发处中断，代理读取以重建它们。合成器/解析器桥接这些，以便 `trace`/`explore` 端到端连接（`src/resolution/callback-synthesizer.ts`、`src/resolution/frameworks/`）。当前通道：回调/观察者、EventEmitter、**React 重渲染**（`setState`→`render`）、**JSX 子组件**（`render`→子组件）、Django ORM 描述符。所有合成边都具有 `provenance:'heuristic'` 和 `metadata.synthesizedBy` + `registeredAt`（接线点），在 `trace`、`node` 追踪和 `context` 调用路径中内联显示。

**原则：部分覆盖率比完全没有更糟糕。** 桥接一个边界但不桥接下一个会暴露一个跳点，代理随后会深入探究并读取以完成它。在 excalidraw 上测量：仅 react-render *增加*了读取到 5-7；只有完成流程（添加 jsx-child 跳点）才将其降低到 0-1。**始终端到端地关闭流程并重新测量** — 绝不发布一个半桥接的流程。

### 验证方法（每个新语言/框架必需）

对于每个**语言 × 框架**，在**小型、中型和大型**真实仓库上使用**每个不少于 3 个不同的流程提示**进行验证：

1. **选择框架的规范流程**（"X 如何到达 Y"：state→render、request→handler→view、query→SQL、action→reducer→store…）。
2. **确定性探测**（针对构建的 `dist/` 运行 `scripts/agent-eval/probe-{trace,node,context,explore}.mjs`）：`trace(from,to)` 端到端连接无中断；**无节点爆炸**（重新索引前后 `select count(*) from nodes` 稳定）；合成边**精确性**抽查（`select … where provenance='heuristic'`）。
3. **代理 A/B 测试**（`scripts/agent-eval/run-all.sh <repo> "<Q>"`）：带与不带 codegraph，**每组 ≥2 次运行**（运行间方差很大 — 绝不在 n=1 时下结论）。记录**持续时间、总工具调用数、Read、Grep**。可选通过块读取钩子（`scripts/agent-eval/hook-settings.json`）证明强制 Read-0 的充分性。
4. **通过标准：** 一个普通的流程问题在仓库的 explore 调用预算内达到 **~0 Read/Grep**，比不使用 codegraph 时运行**更快**，并且在**对照仓库上无退化**。将数字记录在 `docs/design/dynamic-dispatch-coverage-playbook.md`（覆盖率矩阵）中。

完整剧本 + 每种机制的设计：`docs/design/dynamic-dispatch-coverage-playbook.md` 和 `docs/design/callback-edge-synthesis.md`。

### 工作示例 — Excalidraw（TS/React，中型，643 文件）

每种语言/框架需要复制的模板。问题：*"更新元素如何重新渲染屏幕上的画布？"*（完整流程跨越三个 React 边界：观察者回调、`setState`→`render` 和 JSX 子组件）。

| 阶段 | 持续时间 | Read | Grep | codegraph |
|---|---|---|---|---|
| 无 codegraph | 115-139s | 9-10 | 10-11 | 0 |
| 损坏（explore 预算退化） | 131-139s | 5-10 | 3-5 | 6-14 |
| 修复（预算 + 消息 + 合成） | 64-112s | 0-2 | 2-4 | 3-**10** |
| + trace-first 引导 | **51-74s** | **0-2** | 0-4 | **3-4** |

n=4 次无钩子运行/阶段，相同提示。将流程问题引导到 `codegraph_trace` 后：**最佳运行 0 Read / 0 Grep / 3 codegraph / 51s**；**4 次中有 2 次完全干净**（0 Read，0 Grep）。引导消除了过度钻取方差 — 调用次数从 3-10 收紧到 3-4，trace 采纳率从 3/4 → 4/4，并且 `search`+`callers` 路径重建的挣扎下降到 0。运行间方差仍然是真实的；报告范围，从不报告单次运行。**残余的读取/grep 都是 nonce 数据流**（`canvasNonce` — 一个没有图边的局部属性）；那是 def-use/数据流前沿，有意未覆盖（跟踪每个局部变量会爆炸图）。已验证：`trace(mutateElement, renderStaticScene)` 跨越所有三个边界在 **6 个跳点**中连接（`mutateElement → triggerUpdate → [callback] triggerRender → [react-render] render → [jsx] StaticCanvas → renderStaticScene`），每个跳点显示内联源代码 + 接线点；节点计数稳定在 9,289；1 个回调 + 46 个 react-render + 280 个 jsx-render 合成边（无爆炸，精确性已检查）。

## 测试

测试位于 `__tests__/` 中，并镜像它们覆盖的模块。除了明显的之外，值得注意的测试：

- `installer-targets.test.ts` — 跨所有 4 个代理目标的参数化契约套件（见上面的安装程序说明）。
- `evaluation/` — `runner.ts` + `test-cases.ts` 对合成项目进行 codegraph 测试并评分；通过 `npm run eval` 运行（先构建）。不是 `npm test` 的一部分。
- `sqlite-backend.test.ts` — 涵盖原生 + wasm 后端选择和回退。
- `pr19-improvements.test.ts`、`frameworks-integration.test.ts` — 针对特定过去 PR/事件的回归覆盖；不要重命名这些，名称锚定到 git 历史。

测试使用 `fs.mkdtempSync` 创建临时目录并在 `afterEach` 中清理。它们写入真实文件并使用真实的 SQLite — 没有数据库模拟。

### Windows 门控测试

因平台而异的行为（路径解析、驱动器号、`SENSITIVE_PATHS`、`%APPDATA%` 配置目录、CRLF）必须门控，不能假设。使用 `it.runIf(process.platform === 'win32')(...)` 进行仅 Windows 的断言，使用 `it.runIf(process.platform !== 'win32')(...)` 进行仅 POSIX 的断言 — 例如 `/etc` 在 POSIX 上是敏感路径，但在 Windows 上解析为 `C:\etc`（不存在），因此未门控的 `/etc` 断言在 Windows 上会失败。真实验证 Windows 端（见下文）；不要合并一个您尚未看到运行的 Windows 门控测试。

## 跨平台验证

开发机器 — 以及默认的 `npm test` 目标 — 是 **macOS**，因此本地运行覆盖 macOS 路径。其他两个平台不在此处；当更改是平台敏感的（文件监视、套接字/命名管道、路径和符号链接处理、进程生命周期、inotify 预算）时，真实验证它们而不是猜测。

### Linux（Docker）

当被要求在 Linux 上测试或验证时，使用 **Docker** — 没有 Linux 机器，但 Docker 在 macOS 主机上运行。从仓库构建一次性镜像并在其中运行测试套件：

- `FROM node:22-bookworm`；使用排除 `node_modules`/`dist`/`.git`/`.codegraph` 的 `.dockerignore` 进行 `COPY` 仓库；`RUN npm ci && npm run build`。不要重复使用 Mac 的 `node_modules` — `esbuild`/`rollup` 提供平台特定的二进制文件。
- 使用 **`docker run --rm --init`** 运行。`--init` 对于任何进程生命周期测试（守护进程回收、#277 PPID 看门狗、空闲超时）是负载关键的：没有僵尸回收 PID 1，被 SIGKILL 杀死/退出的进程会作为僵尸存在，`process.kill(pid, 0)` 仍然报告它*存活*，因此退出检测断言会虚假失败，即使进程确实退出了。
- Linux 是 inotify 监视预算真正发挥作用的地方：通过 `/proc/<pid>/fdinfo/*`（对 `readlink` 为 `anon_inode:inotify` 的 fd 上的 `^inotify ` 行求和）计算进程的监视数。

### Windows（Parallels VM + SSH）

对于任何 Windows 特定的 PR、错误或实现，在真实的 Windows VM 上验证而不是猜测。连接详情存储在 git 忽略的 **`.parallels`** 文件（仓库根目录）中（VM 名称、客户机 IP、SSH 用户/密钥）。`prlctl exec` 需要 Parallels Pro 并且不可用，因此 SSH 是桥梁。

- 从 Mac 主机连接/运行：`ssh <user>@<guest_ip> "..."`。对于多行工作，通过 stdin 将 PowerShell 管道传输并**首先从注册表刷新 PATH**（winget 安装后 sshd 的会话具有过时的 PATH）：
  ```
  ssh colby@10.211.55.3 "powershell -NoProfile -ExecutionPolicy Bypass -Command -" <<'PS'
  $env:Path = [Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [Environment]::GetEnvironmentVariable("Path","User")
  Set-Location C:\dev\codegraph
  PS
  ```
- 克隆到 **Windows 本地**路径（`C:\dev\codegraph`）并在那里运行 `npm ci` — 永远不要针对共享的 Mac 仓库运行 npm，因为 `esbuild`/`rollup` 提供平台特定二进制文件。
- 客户机工具链（winget）：Node LTS、Git 和 **VC++ ARM64 可再分发组件**（`@rollup/rollup-win32-arm64-msvc` 需要，vitest 会拉取它）。
- 直接从贡献者的 fork 获取 PR 头以避开 `pull/<n>/head` 延迟：`git fetch <fork-url> <branch>` 然后 `git checkout -f FETCH_HEAD`。
- 已知的预先存在的 Windows 失败（它们在 `main` 上重现，与您的更改无关 — 在归咎于您的 PR 之前对照 `origin/main` 确认，并且不要让它们掩盖新的回归）：`security.test.ts > Session marker symlink resistance > does not follow a pre-planted symlink`（符号链接创建在 Windows 上需要权限）；以及 `mcp-initialize.test.ts` / `mcp-roots.test.ts` 套件，它们在 `afterEach` 中因 `EPERM` 移除临时目录而失败，因为生成的 `serve --mcp`（其 `--liftoff-only` 重新执行的孙子进程）仍然持有 cwd / SQLite 文件打开 — 一个 Windows 文件锁定怪癖，不是逻辑错误。

## 发布

发布到 npm 并镜像为 [GitHub Releases](https://github.com/colbymchenry/codegraph/releases)。`CHANGELOG.md` 是真实来源；GitHub Release 说明是从中提取的。

### 编写变更日志条目

**默认：在 `## [Unreleased]` 下编写条目** — 这是保留给在发布之间完成的工作的部分。**不要预先创建 `## [X.Y.Z]` 块**用于下一个版本：Release 工作流的第一步是 `scripts/prepare-release.mjs`，它自动将 `[Unreleased]` 下的所有内容提升到新的 `## [X.Y.Z] - <YYYY-MM-DD>` 块（或者在已存在 `[X.Y.Z]` 块时合并进去 — 但您不需要一个）。预阶段正是导致 v0.9.5 稀疏发布说明事件的原因：在其余工作落地之前手动添加的一个稀疏的 `[0.9.5]` 块被提取器选中，而不是其上方的更大得多的 `[Unreleased]` 部分。不要这样做。

任何条目的格式规则（任何位置 — `[Unreleased]` 或其他）：

1. **编写友好的、面向用户的说明 — 而不是面向工程师的。** 归类在 `### New Features` 和 `### Fixes` 下（句子大小写）。仅在**发布有此内容时**显示 `### Breaking Changes` 和 `### Security` 作为自己的部分；将改进性质的更改折叠到 New Features 中。省略空的部分。（这取代了旧的 Keep-a-Changelog `Added/Changed/Fixed/Removed/Deprecated` 分组：GitHub Release 页面通过 `scripts/extract-release-notes.mjs` **逐字**提取每个版本块，而旧的高密度、以实现为中心的条目呈现为不可读的文字墙 — 因此整个 CHANGELOG 被重写为此格式，并且每个已发布的版本都被重新描述以匹配。）
2. **每个项目符号一个简单明了的句子：** 更改了什么以及为什么对用户重要。以能力开头，或以现在已修复的症状开头。
3. **剥离内部细节。** 没有内部文件路径（`src/...`）、没有内部符号/函数/类名称、没有基准测试数字/百分比/节点或边计数。**保留：** 语言和框架名称（Go、Spring、NestJS、…）、用户输入或设置的内容（`codegraph install`、`codegraph_trace`、`CODEGRAPH_*` 环境变量）、代理/IDE 名称（Claude Code、Cursor、opencode、Kiro、…）以及贡献者致谢时的简短 `Thanks @user`。
4. 条目中的问题/PR 引用使用编号（`(#403)` 等）；GitHub 渲染器会在已发布的发布说明中自动链接它们。
5. **不要自己添加 `[X.Y.Z]: https://...` 链接引用** — `prepare-release.mjs` 在提升版本时自动追加它（幂等：如果已存在，重新运行是空操作）。

多词标题如 `### New Features` 在正常的发布路径上是安全的：`prepare-release.mjs` **情况 A** 将整个 `[Unreleased]` 主体逐字移动到 `[X.Y.Z]`。（只有其很少使用的**情况 B** *合并*使用不会匹配它们的单词 `^### (\w+)$` 正则表达式拆分子部分 — 并且情况 B 仅在预先创建了 `[X.Y.Z]` 块时触发，而上面的规则已经禁止了这点。）

### 发布流程（用户运行这些）

发布由 **GitHub Actions "Release" 工作流**（`.github/workflows/release.yml`）构建和发布。它运行 `scripts/prepare-release.mjs` 以将 `[Unreleased]` 提升为 `[<version>]`（并自动提交 + 推送该 CHANGELOG 更改回 `main`，以便磁盘上的真相与已发布的说明匹配），然后为每个平台打包 Node 运行时（`scripts/build-bundle.sh`）并发布 GitHub Release 和 npm 精简安装程序（`scripts/pack-npm.sh`：一个 shim 包 + 每平台包）。手动发布现在是**错误**的 — 单纯的 `npm publish` 会发布根包（非捆绑），这会破坏任何使用 Node < 22.5 的人。

**Claude 除非被明确要求，否则不会升级版本。** 维护者通常自己升级 — 通常通过 GitHub Web UI 直接编辑 `package.json`。不要主动在无关工作中提交版本升级，也不要在总结 PR 时提出一个。

当维护者确实升级版本时，严格需要的唯一编辑是 `package.json` — 工作流的 "Sync package-lock.json" 步骤检测 `package.json` 和 `package-lock.json` 之间的不匹配，运行 `npm install --package-lock-only --ignore-scripts` 以重写锁定文件的版本字段（顶层 + `packages.""`），并自动提交 + 将结果推送回 `main`，附带 `[skip ci]`。因此对 `package.json` 的 GitHub Web UI 单文件编辑就足以触发一个干净的发布。（如果他们同时在本地编辑了两个文件，那也没问题 — 同步步骤是空操作。）

一旦 `package.json` 在 `main` 上达到目标版本，触发 **Actions → Release → Run workflow**（在 `main` 上）。工作流：

1. 如果已漂移，将 `package-lock.json` 同步到 `package.json` 的版本；提交 + 推送该更改。
2. 运行 `prepare-release.mjs <X.Y.Z>` → 在 `CHANGELOG.md` 中将 `[Unreleased]` 提升为 `[X.Y.Z] - <today>`，追加链接引用，使用 `[skip ci]` 提交 + 推送移动。
3. 在一个 runner 上构建所有平台包，生成 `SHA256SUMS`。
4. 使用来自刚提升的 `[X.Y.Z]` 块的说明创建 GitHub Release。
5. 发布 npm shim + 每平台包。需要 `NPM_TOKEN` 仓库密钥。

**不要自己运行 `npm publish`、`git push` 或 `git tag`** — 这些是对共享状态的发布操作。写入文件，将命令交给用户。

## 内部规则

- `0.7.x` 系列正在进行多代理推广。对 `src/installer/`（特别是 `targets/`）的任何更改都需要相应的测试覆盖和 CHANGELOG 条目 — 安装程序回归会静默地破坏每个新安装。
- 在更改 MCP 工具的功能或代理应如何使用它们时，编辑 `src/mcp/server-instructions.ts` — 它是面向代理工具指南的**唯一真实来源**（问题 #529）。安装程序不再向 `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / `.cursor/rules/codegraph.mdc` / Kiro steering 写入重复的指令块，因此不再需要保持同步。（仓库自己的已检入 `.cursor/rules/codegraph.mdc` 是 dogfooding 配置 — 如果您在此仓库上使用 Cursor，也更新它，但它不会随着发布一起分发。）
- CodeGraph 提供**代码上下文**，而不是产品需求。对于新功能，询问用户关于 UX、边缘情况和验收标准 — 图谱不会告诉您。
- **当用户引用问题、PR 评论或外部报告时，在得出结论之前将其锚定到日期和版本。** 检查评论的 `createdAt` 对照：
  - **最后一个已发布版本** — `grep -m1 '^## \[' CHANGELOG.md` 显示文件顶部的版本（更旧的版本在后面）。日期在最新 `## [X.Y.Z] - YYYY-MM-DD` 之前的评论是在反应*已发布*的状态 — 仅在 `main` 上或未合并分支上的工作不适用。
  - **最后一个主分支提交** — `git log --first-parent main -1 --format='%ai %h %s'`。在最后一次发布之后但在 main 上的修复之前的评论可能已经在那里解决但尚未发布。
  - **当前分支顶端** — 您自己未合并的工作显然不可能是评论所反应的。
  始终在同意用户报告的问题未修复（或修复不完整）之前区分"已发布"、"已合并但未发布"和"进行中"。用户关于最近 PR 说"您的修复只覆盖了 X"通常是指向*已发布*版本的不足 — 您正在开发中的分支可能已经解决了它们，但他们无从知道。
