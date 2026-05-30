---
name: framework-coverage-sweep-2026-05-23
date: 2026-05-23 23:59
project: codegraph
branch: architectural-improvements
summary: 动态调度覆盖扫描完成——所有 14 个 README 框架 + 每个与流程相关的语言均已验证（测量→修复→验证→测试→手册→提交）。约 37 个提交已推送，套件绿色。发布准备（CHANGELOG + PR 到 main）是唯一剩余事项。
---

# 交接：动态调度框架/语言覆盖扫描（完成）

## 从这里继续——先阅读此处
**当前状态：** 覆盖扫描**已完成**，并且**前沿传递**关闭了可处理的局部项。README 14 行表格中的每个框架均为 ✅，每个与流程相关的语言均已验证（TS/JS、Python、Go、Java、C#、PHP、Ruby、Rust、Swift、Dart、Kotlin、Lua/Luau、Scala、C/C++），前沿传递新增了：React 对象数据路由器（字面量）、Next.js 误报修复、Flask-RESTful `add_resource`（redash 6→77）、Flask 元组方法 + 更广泛的检测（flask-realworld 0→19）、gorilla/mux 确认。全部已提交/推送到 `architectural-improvements`（除未跟踪的 `.claude/handoffs/` 外，树干净）。完整套件绿色（**809 通过**，2 跳过；flaky `watcher.test.ts > debounced sync` 重新运行后通过）。**尚无 CHANGELOG 条目，分支尚未合并到 main。**
**立即下一步：** 发布准备——编写 CHANGELOG 条目，对整个扫描进行分组（Flask/FastAPI/Drupal/Rust-Axum+actix/Vapor/Spring-Kotlin/Play + React Router 的路由解析；Python 内置名称保护、Dart 方法范围、C++ 继承的基础修复；flutter-build 和 cpp-override 合成器通道），提升 `package.json`，然后打开 PR 到 main。

> 建议下一条消息："进行发布准备：为此分支上的整个框架/语言覆盖扫描编写 CHANGELOG 条目，提升版本，并打开 PR 到 main"

## 目标
关闭 codegraph 支持的每种语言/框架的**动态调度**静态提取漏洞，使跨符号流（请求→路由→处理器→服务、状态→渲染、虚拟→覆盖）存在于图谱中，代理只需少数 codegraph 调用和 ~0 次 Read/Grep 即可回答流程问题。每种语言/框架：规范流程 `trace` 端到端连接，代理 A/B 显示更少的读取，无节点爆炸，记录在 `docs/design/dynamic-dispatch-coverage-playbook.md`（§6 矩阵 + §7 各项注释）。**此目标现已达成；剩余的是发布准备 + 已记录的前沿。**

## 关键发现（本次会话的工作，全部已提交）
- **路由约定是每个后端的漏洞**——每次都是相同的模式：解析器/提取器假设了一种语法。Flask（中间的 `@login_required`/叠加路由）、FastAPI（空 `""` 路径）、Drupal（FQCN `_form`/单冒号控制器的 `claimsReference` + 通过 composer 名称/类型/`.info.yml` 的 contrib `detect`）、Rust/Axum（链式 `get(h).post(h2)` + 命名空间 `mod::handler`）、actix（构建器 API `web::resource().route(web::get().to(h))`）、Vapor（分组 `routes.grouped("x"); x.get(use:h)`——在真实应用上均为 0）、Spring **Kotlin**（`fun` 处理器语法 + `.kt`）、Play（无扩展名 `conf/routes` → 控制器）、React Router（`<Route>` JSX）。
- **三个基础性修复（广泛受益，非框架特定）：**（1）Python **裸名内置保护**在 `src/resolution/index.ts` 中——名为 `index`/`get`/`update` 的处理器被过滤为内置方法；镜像了点分支 `knownNames` 保护。（2）**Dart 方法范围**在 `src/extraction/tree-sitter.ts` 的 `createNode` 中——Dart 的函数体是签名的兄弟节点，因此方法为 `end==start`（仅签名）；将 `endLine` 扩展到已解析的函数体（受保护，子函数体语法无害）。（3）**C++ 继承**——`extractInheritance` 处理了 `base_clause`（PHP）但未处理 C++ 的 `base_class_clause`；已添加（leveldb 从 219→298）。
- **两个新的合成器通道**在 `src/resolution/callback-synthesizer.ts` 中（Dart 类比 + C++ 类比 react-render）：`flutter-build`（State 方法调用 `setState(` → `build`）和 `cpp-override`（基类虚方法 → 同名子类覆盖，仅限 C++）。
- **先测量一再将"需修复"与"已有覆盖"分开：** Svelte、NestJS（之前），以及本次会话中的 **Lua/Luau**（模块调度已解析）+ **Compose**（组合是普通函数调用，已是静态）无需代码变更。假设的漏洞并不存在。
- **`claimsReference` 预过滤是反复出现的陷阱**（`src/resolution/index.ts:497-503`）：命名未声明符号的路由引用（FQCN、`Controller@method`、`controller#action`、`Class.method`）在 `framework.resolve()` 运行之前就被丢弃了。本次会话中为 Drupal 和 Play 添加了此功能。

## 注意事项
- **`claimsReference`：** 如果新框架的路由引用尽管有正确的 `resolve()` 仍无法解析，则是预过滤的问题——添加 `claimsReference`。
- **重建索引仅在干净索引上拾取解析器更改：** `codegraph index` 是增量的（跳过未更改的文件）；在 `npm run build` 之后，执行 `rm -rf .codegraph && codegraph init -i` 以重新提取。init 消息的边计数仅包含 contains 类型（具误导性）；查询数据库获取真实计数。
- **提取更改影响范围大**（共享的 `createNode`/`extractInheritance`）：在控制仓库上重新检查节点计数（excalidraw 9,290 / django 302）——Dart/C++ 修复已受保护，仅扩展/仅 C++，控制仓库不变。
- **Play `conf/routes` 无扩展名**→需要在 `grammars.ts` 中添加 `isPlayRoutesFile` 选择加入（isSourceFile + detectLanguage→'yaml' 无语法路径）。窄匹配，仅**添加** Play 文件。
- **不稳定：** `watcher.test.ts > debounced sync > should trigger sync after file change`——基于时间，重新运行后通过；与此工作无关。
- **前台的 `sleep` 在 Bash 中被阻塞**→后台 A/B 批次（`run_in_background: true`），读取任务输出文件。zsh 特性：引用通配符（`'*.vue'`）；SQL `count(*)` 在 `$(...)` 中需注意引号。
- 全局 `codegraph` 通过 npm link 链接到此仓库的 `dist/`；`npm run build` 然后重建索引。A/B 测试框架：`scripts/agent-eval/run-all.sh <repo> "<Q>" headless`（有 vs 无 MCP），通过 `node scripts/agent-eval/parse-run.mjs` 解析。

## 如何测试与验证（每种框架的循环）
- 语料库在 `/tmp/codegraph-corpus/<name>`（克隆 S/M/L，`git clone --depth 1`）。索引：`rm -rf .codegraph && codegraph init -i`。
- 测量漏洞：`sqlite3 .codegraph/codegraph.db "select count(*) from nodes where kind='route'"` + 路由→处理器边（`join edges on source where kind='references'`）。前后节点计数（无爆炸）。
- 流程：`node scripts/agent-eval/probe-node.mjs <repo> <symbol>`（显示被调用者/调用者追踪）/ `probe-trace.mjs <repo> <from> <to>`。
- 代理 A/B（≥2 次运行/组，方差真实存在）：`run-all.sh` headless，记录 Read/Grep/时长/codegraph。通过标准 = 使用 codegraph 时读取更少。
- 测试：`npm test`（vitest）。解析器提取测试在 `__tests__/frameworks.test.ts` 中；端到端测试在 `__tests__/frameworks-integration.test.ts` 中（真实 CodeGraph + indexAll）；Dart 范围在 `__tests__/extraction.test.ts` 中；Drupal 在 `__tests__/drupal.test.ts` 中。

## 仓库状态
- 分支 `architectural-improvements`，最后提交 `42a0178 docs(playbook): record frontier pass; test(go): gorilla/mux`。
- 未提交：干净（仅有未跟踪的 `.claude/handoffs/`）。
- 分支上共约 37 个提交（交接的原始 11 个框架 + 本次会话的：Flask/FastAPI、Drupal、Rust/Axum、Vapor、React Router、actix、Dart、Kotlin、Lua、Scala/Play、C/C++——每个一个 feat + 一个 docs(playbook) 提交；Lua 仅为文档）。

## 未解决的线程 / TODO
- [ ] **发布准备（合并的唯一障碍）：** 整个扫描的 CHANGELOG 条目、`package.json` 版本提升、PR 到 main。发布仅通过 `.github/workflows/release.yml`——不要 `npm publish`（见 CLAUDE.md）。
- [x] **前沿传递完成（提交 0456915、03e49ab、42a0178）：** React 对象数据路由器（字面量）、Next.js 误报修复、Flask-RESTful `add_resource`、Flask 元组方法 + 检测、gorilla/mux 确认。
- [ ] **剩余前沿（有意保留，理由见手册 §7 "前沿传递"）：** 匿名/内联闭包（def-use 前沿）、元编程查找器（AR/Eloquent/JPA/EF）、响应式运行时（Vue Proxy / Compose 重组）、Akka 演员、C 回调结构体 422 路扇出、C++ 纯虚基方法、React 惰性数据路由器（变量路径 + 惰性导入）、Play SIRD、Nuxt 特定。强制添加这些会增加噪声。
- [ ] 预先存在、无关：`pages/` 目录中的 Next.js `*.config.mjs` 被视为路由（在 bulletproof-react 中发现的误报）。

## 近期记录（从旧到新，本次会话）
### 第 N 轮 — "覆盖范围还有什么/下一步是什么" → 做了 Flask/FastAPI
- 3 个漏洞：Flask 中间/叠加装饰器、FastAPI 空路径、**Python 裸名内置保护**（名为 `index`/`get` 的处理器被过滤）。microblog 6→27、realworld 12→20、dispatch 290/290。同时修复了 6 个过时的 Laravel/Rails 测试。已提交并推送。
### 第 N 轮 — "下一个是 Drupal"
- FQCN/_form/单冒号控制器的 `claimsReference` + contrib `detect`（composer 类型/名称 + `.info.yml`）。core 536→731（87%）、admin_toolbar 0→14。OOP `#[Hook]` = 前沿。已提交。
### 第 N 轮 — "Rust：Axum/actix/Rocket"
- Axum 链式方法 + 命名空间处理器（realworld 12→19，19/19）；Rocket 已 99%；**actix 构建器 API** `web::resource().route(web::get().to())`（示例 51→128）。已提交（2 个提交：axum，然后 actix）。
### 第 N 轮 — "Vapor（Swift）"
- 解析器在每个真实应用上都是 0 路由；重写以支持任何接收者 + 可选非字符串路径 + `.grouped` 前缀跟踪 + `use:` 鉴别器。template 0→3、SteamPress 0→27、SPI 0→14。已提交。
### 第 N 轮 — "2、3、4"（React Router、actix [已在上方完成]、Dart/Flutter）
- React Router `<Route>` JSX（react-realworld 0→10）。Dart/Flutter：**方法范围修复**（基础性）+ `flutter-build` setState→build 合成器。已提交。
### 第 N 轮 — "下一个是 Kotlin"
- Spring 解析器 `['java']`→`['java','kotlin']` + `fun` 处理器正则表达式（petclinic-kotlin 0→18，18/18；Java 不变 19/19）。Compose 组合已是静态。已提交。
### 第 N 轮 — "Lua/Luau、Scala、C/C++（先 Lua，但做全部三个）"
- **Lua：** 先测量→模块调度已有覆盖（telescope 335 个跨文件调用）；无代码变更，已验证。**Scala/Play：** `conf/routes` 文件遍历选择加入 + Play 解析器（computer-database 0→8）。**C/C++：** 通用调度强（redis 29k）；修复了 C++ `base_class_clause` 继承 + `cpp-override` 合成器（leveldb 12 条精确）。全部已提交并推送。
### 第 N 轮 — "收尾 + 刷新交接"
- 本交接。扫描完成；发布准备（CHANGELOG + PR）是剩余工作。
