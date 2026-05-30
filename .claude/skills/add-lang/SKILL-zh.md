---
name: add-lang
description: 为 codegraph 端到端添加 tree-sitter 语言支持——连接语法 + 提取器、编写测试，然后在 3 个流行的真实仓库上基准测试提取质量和检索价值。当用户运行 /add-lang <language> 或要求为 codegraph 添加/支持新语言（如 Lua、Elixir、Zig、OCaml）时使用。
---

# 为 CodeGraph 添加语言

将新的 tree-sitter 语言接入 codegraph 的提取流水线，证明它能在流行仓库上提取真实符号，并证明它在代理场景中优于无 codegraph 的情况。**完全自主运行**——选择仓库、基准测试、更新文档，然后报告。**绝不提交、推送、发布或打标签**（内部规则）；将所有更改留给用户审查。

参数是 `Language` 联合类型中使用的语言令牌，例如 `lua`、`elixir`、`zig`。如果未给出，询问是哪种语言。在各地使用小写的单令牌形式（`csharp`，不是 `c#`）。

## 先决条件
- 从 codegraph 仓库根目录运行。需要 `node`、`git`、`gh` 以及已登录的 `claude` CLI（基准测试会生成真实的 `claude -p` 运行）。
- 基准测试使用本地开发构建——第 8 步会构建并将其链接到 PATH 上。

## 工作流程

复制此清单并按顺序逐步完成：
```
- [ ] 1. 解析语言；如果已支持则提前退出（仅基准测试）
- [ ] 2. 查找语法 + 健康检查（ABI / 堆损坏）
- [ ] 3. 发现语法的 AST 节点类型（dump-ast.mjs）
- [ ] 4. 连接语言（4 个文件；有时需要第 5 个核心修改）
- [ ] 5. 构建 + 验证提取循环直至通过
- [ ] 6. 添加提取测试；使其变绿
- [ ] 7. 按大小层级自动选择 3 个流行仓库；添加到 corpus.json
- [ ] 8. 全部 3 个基准测试：提取 + 有/无 A/B 比较
- [ ] 9. 更新 README + CHANGELOG
- [ ] 10. 报告；不要提交
```

### 第 1 步 — 解析 + 短路

检查语言是否已连接：在 `LANGUAGES` 常量（`src/types.ts`）和 `EXTRACTORS` 映射（`src/extraction/languages/index.ts`）中查找令牌。如果已支持（如 `typescript`、`rust`），**跳过第 2–6 步**直接进入基准测试（第 7–8 步）以验证/测量——在报告中注明无需代码更改。

### 第 2 步 — 查找语法，然后健康检查

```bash
ls node_modules/tree-sitter-wasms/out/ | grep -i <lang>   # csharp -> c_sharp
```
- **存在** → 通常是现成的；`grammars.ts` 会自动从 `tree-sitter-wasms` 解析它。（许多语言：elixir、zig、ocaml、solidity、toml、yaml……）
- **不存在** → 将 `.wasm` 供应商到 `src/extraction/wasm/` 中（如 `pascal` / `scala` / `lua`）并在第 4 步中将令牌添加到供应商分支中。

**在编写提取器之前务必进行健康检查——存在语法仍可能不可用：**
```bash
node scripts/add-lang/check-grammar.mjs <lang> path/to/valid-sample.<ext>
```
它会打印语法的 ABI 版本并在多语法运行时中多次解析有效样本。如果**失败**（有效代码上出现 ERROR 树——旧 ABI 损坏共享 WASM 堆，会静默丢弃第一个文件之后的每个文件上的嵌套调用/导入；例如 tree-sitter-wasms 的 **Lua** 语法是 ABI 13 并会失败），不要使用该 wasm。**改为供应商更新的（ABI 14/15）构建：**
```bash
npm pack @tree-sitter-grammars/tree-sitter-<lang>   # 通常附带预构建的 *.wasm
# 或构建一个：npx tree-sitter build --wasm   （需要 Docker/emscripten）
cp <the>.wasm src/extraction/wasm/tree-sitter-<lang>.wasm
```
然后将令牌添加到第 4 步中的供应商分支并重新运行 check-grammar 直到通过。**如果无法获得健康的 wasm，停止并告知用户。**

### 第 3 步 — 发现 AST 节点类型

获取一个代表性源文件（编写一个覆盖函数、类/结构体、导入、枚举的小样本；或从已知仓库 `curl` 一个原始文件），然后：
```bash
node scripts/add-lang/dump-ast.mjs <lang> path/to/sample.<ext>
# 供应商语法：传递 wasm 路径而非令牌
node scripts/add-lang/dump-ast.mjs src/extraction/wasm/tree-sitter-<lang>.wasm sample.<ext>
```
频率表 + 字段名（`name:`、`parameters:`、`body:`、`return_type:`）告诉你需要映射什么。打开最接近该语言范式的现有提取器作为模型：`rust.ts`/`scala.ts`（函数式、trait）、`java.ts`/`csharp.ts`（面向对象）、`python.ts`/`ruby.ts`（脚本）、`go.ts`（顶级方法 + 接收者）。

### 第 4 步 — 连接语言（4 个文件）

这是精确且脆弱的连接——精确匹配现有风格：

1. **`src/types.ts`** — 两处编辑：
   - 在 `LANGUAGES` 常量中添加 `'<lang>',`（在 `'unknown'` 之前）；
   - 在 `DEFAULT_CONFIG.include` 中添加 `'**/*.<ext>',`。**不要跳过此项**——这是文件扫描允许列表；没有通配符，`codegraph init` 会找到 **0 个文件**，即使检测/提取已连接。
2. **`src/extraction/grammars.ts`** — 三个映射：
   - `WASM_GRAMMAR_FILES`：`<lang>: 'tree-sitter-<lang>.wasm',`
   - `EXTENSION_MAP`：每个文件扩展名 → `'<lang>'`（例如 `'.lua': 'lua',`）
   - `getLanguageDisplayName`：`<lang>: '<显示名称>',`
   - **仅供应商**：将 `<lang>` 添加到 `(lang === 'pascal' || lang === 'scala' || …)` wasm 路径分支。
3. **`src/extraction/languages/<lang>.ts`** — 新文件，导出 `export const <lang>Extractor: LanguageExtractor = { … }`。映射第 3 步的节点类型。必需字段：`functionTypes`、`classTypes`、`methodTypes`、`interfaceTypes`、`structTypes`、`enumTypes`、`typeAliasTypes`、`importTypes`、`callTypes`、`variableTypes`、`nameField`、`bodyField`、`paramsField`。根据需要添加钩子（`getSignature`、`getVisibility`、`isExported`、`extractImport`、`visitNode`、`getReceiverType`、`interfaceKind`、`enumMemberTypes` 等——参见 `src/extraction/tree-sitter-types.ts`）。
4. **`src/extraction/languages/index.ts`** — `import { <lang>Extractor } from './<lang>';` 并将 `<lang>: <lang>Extractor,` 添加到 `EXTRACTORS`。

**有时需要在 `src/extraction/tree-sitter.ts` 中进行第 5 个核心修改**——变量提取在 `extractVariable` 中有按语言的分支（通用回退仅找到直接的 `identifier`/`variable_declarator` 子节点）。如果语法嵌套了声明名称（例如 Lua 的 `variable_declaration → variable_list`），在那里添加 `} else if (this.language === '<lang>')` 分支，镜像现有的 ts/python/go 分支。不是独特节点的导入形式（Lua/Ruby `require` 是一个*调用*）则在提取器的 `visitNode` 钩子中处理。

### 第 5 步 — 构建 + 验证循环

```bash
npm run build            # tsc + copy-assets（将任何供应商的 *.wasm 复制到 dist/）
```
索引一个小样本仓库并检查提取：
```bash
( cd <sample-repo> && codegraph init -i )
node scripts/add-lang/verify-extraction.mjs <sample-repo> <lang>
```
如果未检测到语言或仅生成 `file`/`import` 节点——节点类型名称错误的经典症状——`verify-extraction.mjs` 会失败（退出码 1）。在失败或弱警告时：在更丰富的文件上重新运行 `dump-ast.mjs`，修复 `<lang>.ts` 中的映射，`npm run build`，重建索引，重新验证。**重复直至通过。**

### 第 6 步 — 测试

添加到 `__tests__/extraction.test.ts`，以 `Rust Extraction` 块为模型：
- `describe('Language Detection')` 中的 `detectLanguage` 断言
- 断言从内联源字符串中提取函数/类/导入的 `describe('<Lang> Extraction')` 块
```bash
npx vitest run __tests__/extraction.test.ts
```
在继续之前变为绿色。

### 第 7 步 — 自动选择 3 个仓库 + 语料库

**不询问地选择**。找到候选，然后策划 3 个真正以 `<lang>` 为主的仓库，每个大小层级一个：
```bash
gh search repos --language=<lang> --sort=stars --limit 40 \
  --json fullName,stargazerCount,description
```
层级（匹配 `corpus.json`）：**小型** <~150 个文件 · **中型** ~150–1500 · **大型** >~1500。跳过标记为 `<lang>` 但主要是其他语言的仓库。为每个仓库编写一个跨文件架构**问题**（需要跨文件追踪的那种）。将 `"<Language>"` 块添加到 `.claude/skills/agent-eval/corpus.json`（字段：`name`、`repo`、`size`、`files`、`question`），以便 `/agent-eval` 可以重用它们。

### 第 8 步 — 全部 3 个基准测试（提取 + A/B）

让开发构建成为 PATH 上的 codegraph **一次**，然后循环：
```bash
npm run build && ./scripts/local-install.sh
scripts/add-lang/bench.sh <lang> <name> <url> "<question>" headless   # ×3
```
`bench.sh` 克隆（共享的 `/tmp/codegraph-corpus`）、擦除 + 索引、运行 `verify-extraction.mjs`，然后通过 `scripts/agent-eval/run-all.sh` 进行有/无检索 A/B（如果提取损坏则跳过付费 A/B）。阅读 `run-all.sh` 打印的每个 `parse-run.mjs` 摘要：工具调用、文件 `Read`、Grep/Bash、codegraph 工具调用、时长和**成本**——包括 `with` 和 `without` 两种情形。循环后，如果需要恢复开发链接：`./scripts/local-install.sh`。

### 第 9 步 — 文档 + CHANGELOG

- **README.md**：将 `<Lang>` 添加到"19+ 种语言"功能点中，并将一行添加到**支持的语言**表中：`| <Lang> | \`.ext\` | 完整支持（类、方法……）|`。
- **CHANGELOG.md**：在顶部（最新版本之上）添加 `## [Unreleased]` 部分，包含 `### Added` → 一个用户视角的条目，例如 *"CodeGraph 现在索引 **<Lang>**（`.ext`）——函数、类、导入和调用边。"* 如果 `## [Unreleased]` 已存在，在其下追加。（发布时会折叠到下一个版本块中。）

### 第 10 步 — 报告（不要提交）

总结供审查：
- **更改的文件**：4 个连接编辑 + 新提取器 + 测试 + README + CHANGELOG + corpus.json（+ 任何供应商的 `.wasm`）。
- **提取**每个仓库：文件 / 节点 / 边 / `verify-extraction` 结果。
- **A/B**每个仓库：`with` vs `without`（工具调用、文件 Read、成本）以及一行结论——codegraph 是否减少了工作量，两种方式是否都得到了正确答案？
- **空白 / 后续**（尚未映射的节点类型、缺失的解析边、框架路由等）。

将更改交给用户。**不要**运行 `git commit`/`push` 或发布——发布通过 GitHub Actions Release 工作流程进行。

## 注意事项
- A/B 会生成真实的**付费** `claude -p` 运行（opus，`--max-budget-usd`），2 种情形 × 3 个仓库。语料库目录 `/tmp/codegraph-corpus` 与 `/agent-eval` 共享，因此克隆可在运行间重用。
- 任何新的 `*.wasm` 必须放在 `src/extraction/wasm/` 中——`copy-assets`（由 `npm run build` 运行）会复制它；否则它不会出现在 `dist/` 中。
- 索引必须由**同一**构建它的二进制文件提供。第 8 步先构建并链接开发构建，因此这成立。
- 如果无法获得语法，或提取无法达到通过状态，**停止并报告**——不要交付半连接的语言。
