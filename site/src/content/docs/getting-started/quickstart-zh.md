---
title: 开始使用
description: 在几秒钟内启动并运行 CodeGraph。
---

在几秒钟内启动并运行 CodeGraph。

## 无需 Node.js — 一个命令即可获取适合你操作系统的构建版本

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh

# Windows（PowerShell）
irm https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.ps1 | iex
```

## 已有 Node？使用 npm 代替（适用于任何版本）

```bash
npx @colbymchenry/codegraph        # 零安装，或：
npm i -g @colbymchenry/codegraph
```

CodeGraph 捆绑了自己的运行时 — 无需编译，无需原生构建，在任何地方都能以相同方式工作。交互式安装程序会自动配置你的代理 — Claude Code、Cursor、Codex CLI、opencode、Hermes Agent、Gemini CLI、Antigravity IDE、Kiro。

## 初始化项目

```bash
cd your-project
codegraph init -i
```

仅此而已 — 当存在 `.codegraph/` 目录时，你的代理会自动使用 CodeGraph 工具。

下一步：构建[你的第一个图](/codegraph/getting-started/your-first-graph/)，或查看完整的[安装](/codegraph/getting-started/installation/)选项。
