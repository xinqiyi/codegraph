---
title: 集成
description: 支持的代理以及手动 MCP 设置。
---

交互式安装程序会自动检测并配置每个支持的代理 — 配置 MCP 服务器并写入其指令文件。

## 支持的代理

- **Claude Code**
- **Cursor**
- **Codex CLI**
- **opencode**
- **Hermes Agent**
- **Gemini CLI**
- **Antigravity IDE**
- **Kiro**

运行 `npx @colbymchenry/codegraph` 并选择你的代理；有关非交互式标志，请参阅[安装](/codegraph/getting-started/installation/)。

## 手动设置

如果你希望自己配置，请全局安装：

```bash
npm install -g @colbymchenry/codegraph
```

将 MCP 服务器添加到 `~/.claude.json`：

```json
{
  "mcpServers": {
    "codegraph": {
      "type": "stdio",
      "command": "codegraph",
      "args": ["serve", "--mcp"]
    }
  }
}
```

可选地，在 `~/.claude/settings.json` 中自动允许只读工具：

```json
{
  "permissions": {
    "allow": [
      "mcp__codegraph__codegraph_search",
      "mcp__codegraph__codegraph_context",
      "mcp__codegraph__codegraph_callers",
      "mcp__codegraph__codegraph_callees",
      "mcp__codegraph__codegraph_impact",
      "mcp__codegraph__codegraph_node",
      "mcp__codegraph__codegraph_status",
      "mcp__codegraph__codegraph_files"
    ]
  }
}
```

:::tip
Cursor 以错误的工作目录启动 MCP 子进程。安装程序会通过注入 `--path` 参数为你处理此问题；如果你手动配置 Cursor，请显式传递项目路径。
:::
