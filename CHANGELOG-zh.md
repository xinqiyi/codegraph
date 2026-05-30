# 变更日志

CodeGraph 的所有重要变更都在此记录。每条记录也作为标记为 `vX.Y.Z` 的 [GitHub Release](https://github.com/colbymchenry/codegraph/releases) 提供，大多数人会在那里查看。

本项目遵循 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) 并遵守[语义化版本控制](https://semver.org/spec/v2.0.0.html)。

## [Unreleased]

### 新功能

- `codegraph init` 现在默认构建初始索引 — 您不再需要 `-i`/`--index` 标志（仍然接受，因此现有命令和脚本继续工作）。(#483)
- Go：Gin 中间件链现在在 `codegraph_trace` 和 `codegraph_explore` 中端到端连接 — 追踪请求可以到达通过 `.Use()` / `.GET()` 注册的中间件和路由处理程序，而不会在框架动态分发链的地方中断。
- `codegraph_explore` 在接口密集的流程上现在更精简：当查询涉及同一接口的多个可互换实现（如 HTTP 拦截器链）时，它以签名形式显示其余部分而不是每个完整的代码体，同时保持分发机制和您询问的任何特定方法为完整形式。相同答案使用更少的 token，因此这类问题不再比普通的 grep/read 更昂贵 — 在测试中，两个回报最慢的代码库（一个 Java 和一个 Python 框架）从比原生搜索略贵变为明显更便宜。不具可互换性的独特代码仍像以前一样完整显示。使用 `CODEGRAPH_ADAPTIVE_EXPLORE=0` 禁用。

### 修复

- 索引仅包含配置式文件（YAML、Twig 或 `.properties`）的项目不再误导性地报告"未找到要索引的文件" — 这些文件在文件级别被跟踪，现在被计为已索引。感谢 @luojiyin1987 (#357)。

## [0.9.7] - 2026-05-28

### 新功能

- Go：gRPC 接口存根现在连接到手写的实现，因此调用者、被调用者、影响和追踪会定位到实际方法而不是空的生成存根。
- 生成的文件（protobuf、gRPC 存根、模拟对象、构建输出）现在在搜索、追踪和探索中排名最后，因此结果定位到您的实际实现而不是自动生成的占位符。
- 当 `codegraph_trace` 找不到静态路径（动态分发断点）时，它现在在一次响应中内联显示两个端点的源代码、调用者和被调用者，因此代理无需进行大量后续调用即可获得完整情况。
- Trace 现在通过优先选择共享目录的符号，在大型多模块仓库中选择正确的端点，而不是从无关模块中抓取任意同名符号。
- 测试文件现在在 `codegraph_explore` 中被降低优先级（Go、Ruby、JS/TS、Java/Kotlin/Scala），因此探索预算用于您的实际实现源代码。
- 小型项目（约 500 文件以下）现在在更少的 MCP 调用中解决流程问题，拥有更精简的工具表面和针对项目大小调整的上下文及探索输出。
- `codegraph_context` 现在自动追踪流程问题，如"X 如何到达 Y"或"追踪从 A 到 B 的路径"，将追踪结果拼接到响应中，这样您就不需要单独的 `codegraph_trace` 调用。
- `codegraph_context` 现在针对小型项目上的路由问题，会内联显示 URL 到处理程序的路由表以及主要路由文件的源代码，因此您不必自己去阅读 `routes.rb` 或 `web.php`。
- `codegraph_context` 搜索现在提升项目核心框架文件所在目录中的结果，因此一个同名的小型扩展文件不再排名高于实际框架核心。
- 接口到实现的链接现在适用于 C#、TypeScript、JavaScript、Swift 和 Scala（以前仅限 Java/Kotlin），因此调查接口方法会显示其具体实现。
- MCP 工具描述现在更短，减少了每次会话的开销，同时保留指导。
- Java 和 Kotlin 导入现在通过完全限定名称解析，因此不同包中的同名类在多模块 Spring 和 Android 代码库中被正确区分，包括跨 Java/Kotlin 互操作边界。
- Java 和 C# 匿名类（`new T() { ... }`）及其重写的方法现在被索引为真正的类节点，因此代理无需 Read 即可在追踪中看到这些隐藏的重写。
- 安装程序不再向代理的指令文件（`CLAUDE.md`、`AGENTS.md`、`GEMINI.md`、Cursor 的 `.cursor/rules/codegraph.mdc` 或 Kiro 的 steering 文档）写入重复的 `## CodeGraph` 指令块 — MCP 服务器现在是唯一的真实来源，重新运行 `codegraph install` 或 `codegraph uninstall` 会移除之前版本留下的块（#529）。如果您在 `CODEGRAPH_START`/`CODEGRAPH_END` 标记内添加了您自己的注释，请先将其移出标记，因为整个标记块都会被移除。

### 修复

- MCP 工具不再返回服务器未运行期间已删除文件的结果 — 会话的第一次查询现在等待追赶同步，因此您获得正确的索引而不是过时的行。
- Windows：黑色控制台窗口不再在每次文件保存或 MCP 重新连接时闪烁（#485, #510, #530）。
- `codegraph index` 和 `init -i` 现在在其摘要中报告真实的边数，而不是因遗漏解析和合成器边而少计。

## [0.9.6] - 2026-05-27

### 新功能

- 企业级 Spring 和 MyBatis 流程现在端到端追踪：MyBatis XML 映射器被索引并链接到其 Java 映射器接口，Spring `@Value` 和 `@ConfigurationProperties` 引用解析到您的 `application.yml`/`.properties` 配置中的匹配键（包括宽松的 kebab/camel/snake 绑定），字段注入的具体 bean 如 `this.field.method()` 解析到其实现（#389）。
- Gemini CLI（以及重新命名的 Antigravity CLI）加上 Antigravity IDE 现在受 `codegraph install` 支持，开箱即用地检测和配置，兄弟设置和 MCP 服务器在重新安装时保持不变（#399）。
- Kiro（CLI 和 IDE）现在在 macOS、Linux 和 Windows 上受 `codegraph install` 支持，拥有自己的 steering 文件以便自然地加载 CodeGraph 指南（#385）。

### 修复

- C/C++：裸 `#include "header.h"` 指令现在连接到真正的头文件而不是一个幻影导入，因此包含关系显示为真实的文件到文件边；系统和标准库头文件被过滤掉，因此它们不会虚假解析（#453）。
- Java/Kotlin：导入现在使用完全限定导入路径来区分不同模块中的同名类，因此调用者、被调用者和追踪在多模块项目中定位到正确的类，而不是按文件邻近度猜测（#314）。
- TypeScript：具有对象形状的 `type` 别名（包括函数类型成员和交叉类型）现在在图谱中暴露其成员，因此像 `handle.stop()` 这样的调用解析到别名成员而不是兄弟目录中无关的相似类（#359）。
- C#：参数、返回、属性和字段类型现在产生引用边，因此 DTO 或服务类型上的调用者和被调用者返回真实结果而不是空值（#381）。
- Go：像 `pkg.Func()` 这样的跨包限定调用现在通过读取您的 `go.mod` 解析到正确的包，因此 Go 单体仓库上的调用者、被调用者、影响和追踪返回完整结果而不是几乎为空（#388）。
- `codegraph_files` 现在在代理传递根级路径如 `/`、`.`、`./`、`""` 或 Windows 风格 `\` 时返回整个项目，子目录过滤器如 `/src`、`./src` 和 `src\components` 都正确解析而不是返回"未找到文件"（#426）。
- 当另一个进程持有索引锁时，文件监视器不再将编辑过的文件标记为新鲜，因此逐文件陈旧性信号保持准确，直到编辑实际被索引（#449）。
- TypeScript/JavaScript：顶层变量初始化器（`const token = getToken()`）中以及内联对象字面量方法中的调用不再被丢弃，因此它们按预期出现在调用者中，包括在 Vue 单文件组件中（#425）。
- 监视同步不再在长时间运行的守护进程中因 `FOREIGN KEY constraint failed` 错误而中止；陈旧的查找现在丢弃单个边而不是使整个同步失败（#455）。
- Hermes：`codegraph install --target hermes` 不再破坏 `~/.hermes/config.yaml`，正确处理 PyYAML 的块样式列表，即使在已损坏的文件上也能干净地重新安装（#456）。
- NestJS：来自 `RouterModule.register([...])` 的路由前缀（包括嵌套的 `children`）现在传播到控制器路由，因此路由显示在其完整路径如 `GET /admin/users` 而不是 `GET /`（#459）。
- C++：调用者现在通过类型化成员指针解析，如 `m_alg->Processing()`，包括外联方法定义和两个类共享方法名的常见情况（#445, #454）。

## [0.9.5] - 2026-05-25

### 新功能

- 在同一项目中运行多个 AI 代理不再使成本翻倍：两个 Claude Code 窗口、一个 worktree 代理或并行子代理现在共享每个项目的一个后台守护进程，具有单个文件监视器、SQLite 连接和 tree-sitter 预热，而不是 N 个独立副本（#411）。
- 守护进程以分离方式运行，因此比任何单个会话存活更久，这意味着关闭一个编辑器或终端永远不会断开其他连接；在最后一个客户端断开后它会短暂停留，以便连续的会话跳过启动成本，然后退出并自我清理。使用 `CODEGRAPH_DAEMON_IDLE_TIMEOUT_MS` 调整空闲等待时间（默认五分钟）。
- 设置 `CODEGRAPH_NO_DAEMON=1` 以选择退出，每个客户端获得一个独立的服务器，适用于调试或禁止本地套接字的沙盒环境；守护进程也是版本固定的，因此升级 CodeGraph 永远不会在连接上混合版本。
- CodeGraph 响应现在告诉代理哪些文件待重新索引：当监视器自上次同步以来发现了编辑时，工具响应会添加一个警告横幅，列出过时文件及其状态，以便代理直接读取这些文件，同时信任其余部分，并在没有待处理内容时零成本（#403）。
- `CODEGRAPH_WATCH_DEBOUNCE_MS` 允许您调整文件监视器的静默窗口（默认 2000ms），适用于具有爆发性写入的工作区，如保存时格式化链或大型生成输出，而无需触及代理的命令行（#403）。
- Objective-C 索引：`.m`、`.mm` 和内容嗅探的 `.h` 文件现在以完整的结构化提取进行解析，包括完整的多部分选择器、属性、导入和超类/协议关系，因此 trace、callers 和 callees 在 iOS 代码库中正常工作（#165）。
- 混合 iOS、React Native 和 Expo 项目现在端到端追踪跨越语言边界：Swift 到 Objective-C 自动桥接、React Native 传统桥和 TurboModules、原生到 JS 事件通道、Expo Modules 以及 Fabric/Codegen 视图组件都被桥接，因此流程通过静态解析单独无法跟进的间隙连接起来（#401）。

### 修复

- TypeScript：仅用于接口属性或方法签名的类型现在产生引用边，因此类型上的影响和调用者包括每个仅为接口形状导入它的消费者（#432）。
- Git worktrees 不再静默借用另一个树的索引；从主检出目录内部嵌套的 worktree 运行 CodeGraph 曾经返回错误分支的代码且没有警告，现在 status 命令和每个读取工具都会指出冲突并提示您在 worktree 中运行 `codegraph init -i`（#155）。
- 文件监视器不再在大型代码库上耗尽操作系统文件监视预算：它在注册监视之前排除索引器忽略的相同目录（默认值加上您的 `.gitignore`），因此 CodeGraph 可以与您的编辑器或开发服务器同时运行而不会达到每用户监视上限（#276）。
- 现在在 `git pull`、分支切换和编辑器外部进行的编辑后，索引保持同步；变更检测基于文件系统而非依赖 `git status`，因此拉取或检出的代码会被捕获而无需完整重新索引。
- MCP 服务器现在在连接时追赶同步，协调在其未运行期间发生的任何更改，因此您的第一次查询反映当前代码而不是过时的快照。
- 依赖、构建和缓存目录如 `node_modules`、`vendor`、`dist`、`build`、`target`、`.venv`、`__pycache__`、`Pods` 和 `.next` 现在默认排除，因此上下文和搜索反映您的代码而不是第三方噪音，即使在没有 `.gitignore` 的项目中也是如此；添加 `.gitignore` 否定规则以索引其中一个（#407）。

## [0.9.4] - 2026-05-24

### 新功能

- 请求到处理程序的流程现在在许多 Web 栈上端到端追踪，新增或改进了 Express、Rails、Spring（Java 和 Kotlin）、Django/DRF、Laravel、Flask、FastAPI、Gin、chi、ASP.NET、Drupal、Axum、actix、Vapor、Play、Vue/Nuxt、Svelte/SvelteKit 和 React Router 的路由解析。
- `codegraph_trace`、`codegraph_callees` 和 `codegraph_explore` 现在遵循没有静态调用边的流程 — 回调和观察者注册、EventEmitter、React 重渲染和 JSX 子组件、Flutter `setState` 到 `build`、C++ 虚拟重写以及 Java/Kotlin 接口到实现分发（如 Spring 的 `@Autowired` 服务调用）— 每个桥接的跳点在 trace 中内联标记其注册位置。
- `codegraph_trace` 现在返回自包含的流程档案：每个跳点显示其完整代码体以及目标的传出调用，因此单次 trace 通常可以回答"X 如何到达 Y"的问题，无需后续的 explore、node 或 Read。
- `codegraph_explore` 现在在您的查询命名了流程中的符号时以执行流程开头，在这些符号中查找调用路径（包括跨动态分发跳点），因此您无需切换工具即可获得 trace 质量的答案。
- `codegraph_node` 和 `codegraph_trace` 现在输出带行号的源代码（与 `codegraph_explore` 和 Read 一致），因此您无需重新读取文件以恢复行号即可引用或编辑精确的行。
- 新的 `CODEGRAPH_MCP_TOOLS` 环境变量允许您只暴露选定的 codegraph 工具子集（例如 `trace,search,node,context`），而无需编辑客户端的 MCP 配置；未设置时暴露所有工具。
- 发布归档现在附带 `SHA256SUMS` 文件，npm 启动器验证其下载的包与此文件比对，在不匹配时中止（在此更改之前发布的版本跳过验证而不是失败）。

### 修复

- 上述路由工作基础的若干静态提取和解析正确性修复：以前缺失的 C++ 继承边、仅提取签名的 Dart 方法、被静默丢弃的名为 `index`/`get`/`update` 的 Python 处理程序，以及在具有非常大文件的代码库上导致源代码返回不足的 explore 输出预算问题。
- `codegraph serve --mcp` 在其父代理被强制杀死（OOM、`kill -9` 或容器拆除）后不再继续在 Linux 上运行，以前它会无限期地持有 inotify 监视、文件描述符和 SQLite WAL；服务器现在在其父进程更改时立即关闭，可通过 `CODEGRAPH_PPID_POLL_MS` 调整（#277）。
- 通过尚未镜像匹配按平台包的注册表镜像安装 `@colbymchenry/codegraph` 不再以 `no prebuilt bundle for <platform>` 失败；启动器现在从 GitHub Releases 下载包并缓存，使用 `CODEGRAPH_NO_DOWNLOAD=1` 禁用回退，使用 `CODEGRAPH_DOWNLOAD_BASE` 指向您自己的镜像（#303）。
- `install.sh` 在耗尽 GitHub 未认证 API 速率限制的共享或云主机上不再以 `403` / "could not resolve latest version" 失败；它现在通过无限制的 releases 重定向解析版本，`CODEGRAPH_VERSION` 接受像 `0.9.4` 这样的裸版本以及 `v0.9.4`（#325）。

## [0.9.3] - 2026-05-22

### 新功能

- 新的 `codegraph uninstall` 命令一步到位地从每个已配置的代理 — Claude Code、Cursor、Codex CLI、opencode 和 Hermes Agent — 中干净地移除 CodeGraph，询问是清理您的全局配置还是此项目的本地配置，并报告精确触及了哪些代理；它接受 `--location`、`--target` 和 `--yes` 用于脚本化或非交互式使用，仅移除 `codegraph install` 写入的内容，并保留您的 `.codegraph/` 索引（#313）。

### 修复

- 索引大型多语言项目不再中途因 `Fatal process out of memory: Zone` 崩溃在 Node.js 22 和 24 上中止，即使有足够的空闲 RAM — CodeGraph 现在使用 V8 标志启动以保持语法编译远离优化层级，任何未直接获得标志的启动路径会自动重新执行一次以应用它（#298, #293）。Node 25 暂时仍然受阻，因为其此 bug 的变体无法通过相同标志修复。
- 从 Cursor 卸载现在直接删除遗留的 `.cursor/rules/codegraph.mdc` 文件，而不是留下一个孤立的空规则，同时保留您在 CodeGraph 标记之外添加的任何内容。

## [0.9.2] - 2026-05-21

### 破坏性变更

- CodeGraph 不再有配置文件：`.codegraph/config.json` 和整个配置表面已移除，用于它的库 API（配置类型、`init()` 上的 `config` 选项以及获取/更新配置导出）已被移除 — 现有配置文件现在被忽略，`.gitignore` 是索引内容的唯一真实来源。`.codegraphignore` 标记也不再受支持；请改用 `.gitignore`。

### 新功能

- `codegraph install` 现在支持 Hermes Agent（Nous Research），配置 CodeGraph MCP 服务器，以便 Hermes 像其他代理一样驱动知识图谱。
- Drupal 项目（8/9/10/11）现在被检测并用框架智能索引：来自 `*.routing.yml` 的路由链接到其控制器、表单或实体处理器，模块间的钩子实现连接到其规范钩子名称，因此询问钩子的调用者会返回每个实现（#268）。
- 索引现在零配置，并在各处尊重您的 `.gitignore` — 在 git 仓库中通过 git，在非 git 项目中通过直接读取 `.gitignore` 文件 — 因此要将某些内容排除在图谱之外，只需将其添加到 `.gitignore` 即可。行为变更：未 gitignore 的已提交文件现在即使在 `vendor/`、`Pods/` 或已提交的 `dist/` 下也会被索引；添加 `.gitignore` 否定规则以排除它们（#283）。

### 修复

- Windows：全局安装后运行任何 `codegraph` 命令不再失败 — 启动器现在直接调用捆绑的运行时而不是现代 Node 拒绝生成的 `.cmd` 文件，因此无论您的 Node 版本如何，`codegraph` 都能工作（#289）。

### 安全

- 每次 `codegraph_context` 调用时写入的临时目录标记现在安全地打开，因此它不能跟随符号链接，堵住了共享机器上的其他本地用户可能将该写入重定向到您可写文件上的漏洞（#280）。

## [0.9.1] - 2026-05-21

### 修复

- 独立安装程序（`curl … | sh` 和 `irm … | iex`）在没有安装 Node 的机器上不再启动失败。
- 在 Linux x64 上使用 `npm i -g` 安装现在能找到其包，解决了 0.9.0 版本静默发布时没有 linux-x64 包的问题；发布流程现在验证每个包已到达 npm 注册表，因此发布不能再以"绿色但已破损"通过。

## [0.9.0] - 2026-05-21

CodeGraph 现在自带自包含运行时，因此它可以在任何 Node 版本上安装 — 或者根本不安装 — 无需原生构建步骤，旧的间歇性 "database is locked" 错误已永久消失。

### 新功能

- 不需要 Node.js 的一行独立安装程序：macOS 和 Linux 上的 `install.sh`，Windows 上的 `install.ps1` 获取自包含包并将 `codegraph` 放入您的 PATH（您仍可在任何 Node 版本上使用 `npm`/`npx`）。
- CodeGraph 现在使用其捆绑运行时内置的真正 SQLite，支持完整的 WAL 和 FTS5，从根本上修复了并发读取的 "database is locked" 错误，完全移除了原生构建步骤，并且对之前被困在旧 WASM 回退上的任何人来说运行更快（#238）。
- Lua：CodeGraph 现在索引 `.lua` 项目（Neovim 插件、Kong、OpenResty、游戏代码），展示函数、表方法、局部变量、`require(...)` 导入以及它们之间的调用边。
- Luau：CodeGraph 现在索引 `.luau`，Roblox 的类型化 Lua 超集，在 Lua 提取的所有内容之上添加了类型和 `export type` 别名、类型化函数签名、泛型和 Roblox 实例路径 require（#232）。
- `codegraph status` 现在报告有效的日志模式，因此 "database is locked" 报告可以一目了然地轻松分类。

### 修复

- 重新运行 `codegraph install` 现在会移除 0.8 之前版本写入 Claude Code 设置中的损坏的自动同步钩子，这些钩子曾在每次轮次导致 "Stop hook error: unknown command 'sync-if-dirty'"。清理是精确的，不影响无关的钩子。在受影响的机器上重新运行 `codegraph install` 一次以清除错误。

## [0.8.0] - 2026-05-20

### 破坏性变更

- 最低支持的 Node.js 版本现在是 20（Node 18 已停止支持）；Node 22 LTS 和 Node 24 开箱即用地获得快速原生后端，其他 Node 版本仍通过较慢的 WASM 回退运行，Node 25+ 仍然受阻（#81）。如果您使用的是旧版 Node，请升级到 20 或更新版本。

### 新功能

- NestJS：CodeGraph 现在识别 NestJS 项目并展示绑定每个处理程序的路由，涵盖 HTTP 控制器、GraphQL 解析器、微服务处理器和 WebSocket 网关，自动从任何 `@nestjs/*` 依赖中检测（#220）。
- `codegraph_explore` 源代码现在包含行号，因此代理可以直接从结果中引用 `file:line`，而无需重新打开文件来查找行号；设置 `CODEGRAPH_EXPLORE_LINENUMS=0` 以禁用。
- 在 WSL2 `/mnt/*` 驱动器上，实时文件监视器太慢且可能破坏 MCP 启动，CodeGraph 现在跳过监视器并提供改用 git 钩子保持索引新鲜；新的 `CODEGRAPH_NO_WATCH=1`（或 `serve --mcp --no-watch`）在任何地方强制关闭监视器，`CODEGRAPH_FORCE_WATCH=1` 在您的设置实际快速时覆盖 WSL 自动检测。
- CodeGraph 现在引导代理直接使用几次 codegraph 调用来回答"X 如何工作"和架构问题，而不是委托给文件读取子代理或 grep 加 read 循环，这在中型和大型代码库上提供更快、更便宜、带有 `file:line` 引用的答案。
- 对类、接口、结构体或枚举启用代码的 `codegraph_node` 现在返回紧凑的成员大纲（字段加上带行号的方法签名），而不是整个类体；函数和方法仍返回其完整源代码。
- `codegraph_explore` 输出现在随项目大小扩展，因此小型项目获得比您的原生 grep 加 read 流程更紧凑的响应，而大型代码库保持其更充分的预算，每个文件的上限阻止单个密集文件沦为整个文件转储（#185）。感谢 @essopsp。
- 搜索排名现在正确降低驼峰式测试文件（`FooTest.kt`、`BarTests.swift`、`BazSpec.scala`、`QuxTestCase.cs`）和 Kotlin、Swift、Scala 和 C# 中的测试源集目录的优先级，因此真正的实现不再被测试超越。

### 修复

- `codegraph_explore` 输出现在被硬限制在其大小预算内，因此过大的响应不再超出上限并留在代理的上下文中，在每次轮次中被重新读取。
- 新创建的未跟踪文件不再被永远报告为待处理并在每次 `codegraph sync` 时从头重新索引；CodeGraph 现在像处理跟踪文件一样对它们进行哈希比较（#206）。感谢 @15290391025。
- `codegraph init -i` 现在在非子模块的嵌套独立 git 仓库中找到源代码（在 CMake 超仓库布局中常见），而不是报告"未找到要索引的文件"（#193）。感谢 @timxx。
- 在 Node 24 上，索引不再静默降至较慢的回退后端并带有无法清除的警告；在 Node 22 或 24 上的新安装现在无需编译器即可获得快速原生后端，`codegraph status` 应报告它（#203）。感谢 @Finndersen。
- 当索引实际存在时，MCP 工具不再因 "CodeGraph not initialized" 失败；当客户端未报告工作区根目录时，服务器现在通过标准 MCP `roots/list` 请求请求它，然后才回退，并且在项目仍然无法解析时错误消息是可操作的（#196）。感谢 @zhangyu1197。
- MCP 服务器在 WSL2 下当项目位于 NTFS `/mnt/*` 挂载点上时不再在启动时挂起，因此 codegraph 工具实际出现；CodeGraph 在那里自动跳过监视器，使用手动和 git 钩子同步回退（#199）。感谢 @mengfanbo123。
- Claude Code 项目本地安装现在将 MCP 服务器写入 `.mcp.json`（Claude Code 实际读取项目级服务器的文件）而不是它忽略的文件，重新运行 `codegraph install` 自动迁移受影响的项目（#207）。感谢 @Jhsmit。
- `codegraph_explore` 和 `codegraph_context` 输出中的源代码省略标记现在与语言无关而不是 C 风格注释，因此它们在 Python、Ruby 和其他非 C 源块中不再显得错误。

## [0.7.10] - 2026-05-19

### 修复

- CodeGraph 工具现在在慢速文件系统上（Docker Desktop VirtioFS 在 macOS 上、WSL2）可靠地出现在您的客户端中，以前启动握手可能会超时，使进程运行但没有可见的工具（#172）。感谢 @sashanclrp 和 @sgrimm。
- 在 Windows PowerShell 和 cmd.exe 上，`codegraph index` 和 `codegraph sync` 期间的终端输出不再变成乱码；CodeGraph 现在在 Windows 上默认使用纯 ASCII 字符，使用 `CODEGRAPH_UNICODE=1` 选择 Unicode 字符，或 `CODEGRAPH_ASCII=1` 在任何平台上强制使用 ASCII（#168）。感谢 @starkleek 和 @Bortlesboat。
- 模块限定符号查找现在在 codegraph 工具中解析，因此您可以传递像 `module::symbol`（Rust、C++、Ruby）、`Module.symbol`（TypeScript、JavaScript、Python）或 `module/symbol` 这样的名称，包括多级路径和 Rust 前缀如 `crate`、`super` 和 `self`（#173）。感谢 @joselhurtado。

## [0.7.9] - 2026-05-17

### 新功能

- opencode：安装程序现在写入带有 CodeGraph 使用指南的 `AGENTS.md` 文件，因此 opencode 会使用 `codegraph_*` 工具而不是回退到其原生搜索。
- opencode：您在 `opencode.jsonc` 中的注释和格式现在在安装、重新安装和卸载后保持不变，因为安装程序进行精确编辑而不是重写整个文件。

### 修复

- opencode：`codegraph install` 现在在 opencode 实际读取的文件中配置 MCP 服务器 — 以前它写入了一个 opencode 默认忽略的配置文件，因此 CodeGraph 条目从未出现在任何 opencode 会话中；升级后重新运行 `codegraph install --target=opencode` 以使条目落到正确位置。

## [0.7.7] - 2026-05-17

### 新功能

- `codegraph install` 现在从一次多选提示中设置 Claude Code、Cursor、Codex CLI 和 opencode，它检测到的任何代理都预先勾选，因此一次安装就配置好您使用的每个编辑器（#137）。
- 您可以使用 `--target`、`--location`、`--yes`、`--no-permissions` 和 `--print-config` 等标志为脚本和 CI 进行非交互式安装。
- `codegraph init` 现在为您全局安装的任何代理自动配置项目本地代理配置，因此一次全局 `codegraph install` 在您打开的每个项目中都能工作，无需为每个项目重新安装。
- 代理指令现在是代理无关的，告诉每个代理信任 codegraph 结果而不是用 grep 重新验证，修复了即使 codegraph 可用时 Cursor 和 Codex 也回退到原生搜索的情况。
- 安装提示更清晰：代理选择器放在首位，单独的"将 CLI 安装到您的 PATH"和"应用于所有项目还是仅此项目"问题不再都被理解为"全局"。

### 修复

- Cursor：全局安装的 codegraph 不再在每个工作区报告"not initialized"；安装程序现在将正确的项目路径传递到 Cursor 的 MCP 配置中，以解决 Cursor 使用错误工作目录启动 MCP 服务器的问题。

感谢 @andreinknv 为本版本提供的实质性草案。

## [0.7.6] - 2026-05-13

### 修复

- 修复了在全新全局安装后立即出现 `codegraph` 命令因 `permission denied` 失败的问题 — 0.7.5 包发布时 CLI 缺少可执行位，因此您的 shell 拒绝运行它。新安装开箱即用。如果您卡在 0.7.5 上，请升级到 0.7.6 或通过 `chmod +x` 使已安装的二进制文件可执行来解除阻塞。

[0.9.7]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.7
[0.9.6]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.6
[0.9.5]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.5
[0.9.4]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.4
[0.9.3]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.3
[0.9.2]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.2
[0.9.1]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.1
[0.9.0]: https://github.com/colbymchenry/codegraph/releases/tag/v0.9.0
[0.8.0]: https://github.com/colbymchenry/codegraph/releases/tag/v0.8.0
[0.7.10]: https://github.com/colbymchenry/codegraph/releases/tag/v0.7.10
[0.7.9]: https://github.com/colbymchenry/codegraph/releases/tag/v0.7.9
[0.7.7]: https://github.com/colbymchenry/codegraph/releases/tag/v0.7.7
[0.7.6]: https://github.com/colbymchenry/codegraph/releases/tag/v0.7.6
