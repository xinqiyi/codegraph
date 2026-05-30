---
name: agent-eval
description: 通过在真实代码库上比较使用和不使用 CodeGraph 时代理的行为，基准测试 CodeGraph 的检索质量。当用户运行 /agent-eval 或要求针对某个语言的仓库测试、基准测试、审计或验证 codegraph 版本（本地开发构建或已发布的 npm 版本）时使用。
---

# CodeGraph 质量审计

衡量 CodeGraph 在选定真实世界仓库上对特定 codegraph 版本相比纯 grep/read 有多大帮助。驱动 `scripts/agent-eval/` 中的测试框架。

## 先决条件
- `tmux` 3+、已登录的 `claude` CLI、`node`、`git`（macOS/Linux）。
- 从 codegraph 仓库根目录运行。

## 工作流程

复制此清单：
```
- [ ] 1. 选择版本（本地或 npm）
- [ ] 2. 选择语言
- [ ] 3. 按大小选择仓库
- [ ] 4. 选择测试框架（headless / tmux / both）
- [ ] 5. 在后台运行 audit.sh
- [ ] 6. 报告结果
```

**第 1 步 — 版本。** 使用 AskUserQuestion 询问：要测试哪个 codegraph 版本。提供"本地开发构建"和"最新发布版"；自由文本"其他"让用户输入特定版本（例如 `0.7.10`）。将答案映射到 VERSION 令牌：
- "本地开发构建"→ `local`
- "最新发布版"→ `latest`
- 输入的版本→该字符串（例如 `0.7.10`）

**第 2 步 — 语言。** 读取 `.claude/skills/agent-eval/corpus.json`。使用 AskUserQuestion 询问要测试哪种语言，列出有条目的语言。

**第 3 步 — 仓库。** 从所选语言的条目中，询问哪个仓库。用其大小和文件数标记每个选项，例如 `excalidraw — Medium (~600 files)`。每个条目包含 `repo` URL 和一个代表性 `question`。

**第 4 步 — 测试框架。** 使用 AskUserQuestion 询问要运行哪个测试框架，并将答案映射到 MODE 令牌：
- "Headless" → `headless` — 使用 stream-json 的 `claude -p`：精确的令牌数/成本和干净的工具序列（2 次运行，快速，无 TTY）。
- "Interactive (tmux)" → `tmux` — 在 tmux 中驱动真实 Claude TUI：忠实的 Explore 子代理行为、来自会话日志的指标（2 次运行，较慢）。
- "Both" → `all` — headless + interactive（4 次运行）。

**第 5 步 — 运行。** 在后台启动（设置版本，克隆（如缺失）、擦除 + 重建索引、运行所选测试——需几分钟）：
```bash
scripts/agent-eval/audit.sh <VERSION> <repo-name> <repo-url> "<question>" <MODE>
```

**第 6 步 — 报告。** 当任务完成时，读取日志并按测试报告：
- Headless（`parse-run.mjs`）：总工具调用、文件 `Read`、Grep/Bash、codegraph 工具调用、时长、**总成本**。
- Interactive（`parse-session.mjs`）：`VERDICT: codegraph_explore used Nx | Read N | Grep/Bash N` 和 `TOKENS:` 行。

以成本 + 工具/Read 计数开头——它们是可靠的信号；原始令牌输入/输出会被子代理委派和提示缓存混淆。说明 codegraph 是否减少了工作量，以及两种情形是否都得到了正确答案。

## 注意事项
- 索引在每次运行时重建（`audit.sh` 擦除 `.codegraph`）——不同版本提取不同，因此索引必须由**同一**构建它的二进制文件提供。
- `audit.sh` 临时修改全局 `codegraph` 安装以进行测试，然后通过 `local-install.sh` 恢复你的开发链接。
- 语料库仓库克隆到 `/tmp/codegraph-corpus`（如果已存在则重用）。
- 在 `corpus.json` 中添加或编辑仓库（字段：`name`、`repo`、`size`、`files`、`question`）。
