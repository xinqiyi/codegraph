---
title: MCP 服务器
description: CodeGraph 通过 MCP 向 AI 代理暴露的工具。
---

CodeGraph 作为[模型上下文协议](https://modelcontextprotocol.io/)服务器运行。使用以下命令启动：

```bash
codegraph serve --mcp
```

由安装程序配置的代理会自动启动此服务器。当存在 `.codegraph/` 索引时，代理会使用以下工具。

## 工具

| 工具 | 用途 |
|---|---|
| `codegraph_search` | 在整个代码库中按名称查找符号 |
| `codegraph_context` | 为任务构建相关的代码上下文 — 在一次调用中组合搜索、节点、调用者和被调用者 |
| `codegraph_trace` | 在一次调用中追踪两个符号之间的调用路径（"X 如何到达 Y"）— 每个跳转都内联其主体，追踪 grep 无法追踪的动态分发跳转（回调、React 重新渲染、接口→实现） |
| `codegraph_callers` | 查找哪些内容调用了某个函数 |
| `codegraph_callees` | 查找某个函数调用了哪些内容 |
| `codegraph_impact` | 分析更改某个符号会影响到哪些代码 |
| `codegraph_node` | 获取特定符号的详细信息（可选附带源代码） |
| `codegraph_explore` | 在一次调用中返回按文件分组的多个相关符号的源代码，以及关系图 |
| `codegraph_files` | 获取已索引的文件结构（比文件系统扫描更快） |
| `codegraph_status` | 检查索引健康状况和统计信息 |

## 代理应如何使用

CodeGraph **就是**预先构建的搜索索引。对于"X 如何工作？"、架构、追踪或"X 在哪"这类问题，代理应通过少量 CodeGraph 调用来回答并停止 — 通常**零文件读取** — 而不是通过 `grep` + `Read` 重新推导答案。一次直接的 CodeGraph 调用只需少量调用；而 grep/read 探索则需要数十次。

安装程序会自动将此指南写入每个代理的指令文件中。
