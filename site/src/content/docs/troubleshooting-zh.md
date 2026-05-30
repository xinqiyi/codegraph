---
title: 故障排除
description: 最常见 CodeGraph 问题的修复方法。
---

## "CodeGraph 未初始化"

请先在项目目录中运行 `codegraph init`。

## 索引速度慢

请检查 `node_modules` 和其他大型目录是否已被排除（如果它们已被 gitignore，则默认会被排除）。使用 `--quiet` 参数可减少输出开销。

## MCP 提示 `database is locked`

当前版本不应出现此问题：CodeGraph 捆绑了自家的 Node 运行时，并使用 Node 内置的 `node:sqlite`（WAL 模式），其中并发读取永远不会阻塞写入。如果仍然遇到此问题：

- **你安装的是旧版本（0.9 之前）。** 重新安装以获取捆绑的运行时 — `curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh`（macOS/Linux）、`irm https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.ps1 | iex`（Windows）或 `npm i -g @colbymchenry/codegraph@latest`。
- **`codegraph status` 显示 `Journal:` 不是 `wal`** — 此文件系统上无法启用 WAL（在网络共享和 WSL2 `/mnt` 上很常见），因此读取可能会阻塞写入。将项目（及其 `.codegraph/` 文件夹）移动到本地磁盘上。

## MCP 服务器无法连接

确保项目已初始化/已索引，验证 MCP 配置中的路径是否正确，并检查 `codegraph serve --mcp` 是否能从命令行正常工作。

## 缺少符号

MCP 服务器会在保存时自动同步（等待几秒钟）。如有需要，可手动运行 `codegraph sync`。检查文件的语言是否为[支持的语言](/codegraph/reference/languages/)，并且没有被 `.gitignore` 排除。
