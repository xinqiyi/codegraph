---
title: 你的第一个图
description: 构建索引并对其运行第一批查询。
---

CodeGraph 安装完成后，构建和探索图只需三个命令。

## 索引一个项目

```bash
cd your-project
codegraph init -i      # 初始化 + 索引一步完成
```

`init` 创建 `.codegraph/` 目录；`-i`（或 `--index`）立即构建完整索引。对于已有项目，你可以随时重新索引：

```bash
codegraph index          # 完整索引
codegraph sync           # 增量更新已更改的文件
```

## 检查是否成功

```bash
codegraph status
```

这会报告节点/边/文件的数量、活动的 SQLite 后端以及日志模式 — 快速健康检查，确认索引已就绪。

## 运行查询

```bash
codegraph query UserService          # 按名称查找符号
codegraph callers handleRequest      # 哪些内容调用了一个函数
codegraph callees handleRequest      # 一个函数调用了哪些内容
codegraph impact AuthMiddleware      # 变更会影响到什么
codegraph context "修复登录流程"     # 构建任务相关的上下文
```

每个命令都接受 `--json` 参数以输出机器可读的结果。请参阅完整的 [CLI 参考](/codegraph/reference/cli/)。

## 交给你的代理

当存在 `.codegraph/` 目录并且代理已配置（请参阅[安装](/codegraph/getting-started/installation/)），你的代理会自动使用 [MCP 工具](/codegraph/reference/mcp-server/) — 无需额外步骤。
