---
title: 安装
description: 安装 CodeGraph 并配置你的 AI 编码代理。
---

## 1. 运行安装程序

```bash
npx @colbymchenry/codegraph
```

安装程序将执行以下操作：

- 询问要配置哪些代理 — 自动检测已安装的 **Claude Code**、**Cursor**、**Codex CLI**、**opencode**、**Hermes Agent**、**Gemini CLI**、**Antigravity IDE** 和 **Kiro**。
- 提示是否将 `codegraph` 安装到 `PATH` 中（以便代理可以启动 MCP 服务器）。
- 询问配置是应用于所有项目还是仅当前项目。
- 为每个选中的代理写入 MCP 服务器配置及指令文件（例如 `CLAUDE.md`、`.cursor/rules/codegraph.mdc`、`~/.codex/AGENTS.md`）。
- 当 Claude Code 是目标之一时，设置自动允许权限。
- 初始化当前项目（仅限本地安装）。

## 非交互式（脚本 / CI）

```bash
codegraph install --yes                              # 自动检测代理，全局安装
codegraph install --target=cursor,claude --yes       # 显式指定目标列表
codegraph install --target=auto --location=local     # 检测到的代理，项目本地安装
codegraph install --print-config codex               # 打印代码片段，不写入文件
```

| 标志 | 值 | 默认值 |
|---|---|---|
| `--target` | `auto`、`all`、`none` 或 csv 格式（`claude,cursor,…`） | 提示选择 |
| `--location` | `global`、`local` | 提示选择 |
| `--yes` | （布尔值） | 每一步都提示 |
| `--no-permissions` | （布尔值）跳过 Claude 自动允许列表 | 启用权限 |
| `--print-config <id>` | 转储单个代理的配置片段并退出 | — |

## 2. 重启你的代理

重启你的代理（Claude Code / Cursor / Codex CLI / opencode / Hermes Agent / Gemini CLI / Antigravity IDE / Kiro）以使 MCP 服务器加载。

## 3. 初始化项目

```bash
cd your-project
codegraph init -i
```

这会构建每个项目的知识图谱索引，并配置项目本地的代理表面，因此一次全局的 `codegraph install` 即可在你打开的每个项目中使用。

## 支持的操作系统

每个版本都会为所有三种桌面操作系统（x64 和 arm64）发布一个自包含的构建（捆绑了 Node 运行时 — 无需编译）：

| 平台 | 架构 | 安装方式 |
|---|---|---|
| Windows | x64、arm64 | PowerShell 安装程序或 npm |
| macOS | x64、arm64 | Shell 安装程序或 npm |
| Linux | x64、arm64 | Shell 安装程序或 npm |

## 卸载

改变主意了？一个命令即可从所有已配置的代理中移除 CodeGraph：

```bash
codegraph uninstall
```

这会逆向安装过程 — 从每个已配置的代理中移除 CodeGraph 的 MCP 服务器配置、指令和权限。你的项目索引（`.codegraph/`）将保持不变；使用 `codegraph uninit` 逐个项目移除。使用 `--target` 从特定代理中移除，或使用 `--yes` 以非交互模式运行。
