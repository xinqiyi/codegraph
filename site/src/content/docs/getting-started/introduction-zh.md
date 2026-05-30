---
title: 简介
description: CodeGraph 是什么，以及它为何能让 AI 编码代理更快、更省钱。
---

CodeGraph 是一个**本地优先的代码智能工具**。它使用 [tree-sitter](https://tree-sitter.github.io/) 解析你的代码库，将每个符号、边和文件存储在本地 SQLite 数据库中，并将结果作为一个可查询的**知识图谱**暴露出来 — 通过[模型上下文协议（MCP）](/codegraph/reference/mcp-server/)、CLI 和 TypeScript 库。

它的存在是为了让 AI 编码代理 — Claude Code、Cursor、Codex CLI、opencode、Hermes Agent、Gemini CLI、Antigravity IDE 和 Kiro — **回答结构性问题时无需扫描文件**。代理无需通过 `grep`、`glob` 和 `Read` 来重建代码的组织方式，而是查询预先构建的索引，只需少量调用就能获得答案。

## 为什么这很重要

当代理探索代码库时，它的大部分预算都花在了*发现*上 — 在读取正确文件之前先要找到它们。CodeGraph 消除了这一步骤：符号关系、调用图和结构都已经索引好了。

在 7 个真实世界的开源代码库上进行的测试（每个臂中位数 4 次运行）表明，为代理提供 CodeGraph 平均可以：

- **节省 35% 的成本**
- **减少 57% 的令牌数**
- **快 46%**
- **减少 71% 的工具调用**

收益随代码库规模的增长而增长 — 在大型仓库上，代理可以从索引中直接获取答案，**零文件读取**。

## 图中包含什么

- **符号** — 函数、类、方法、类型、路由、组件等。
- **边** — 调用、导入、继承、引用和特定于框架的关系。
- **文件** — 结构信息加上全文搜索（FTS5）。

提取是**确定性的** — 源自 AST，绝不经过 LLM 总结。

## 100% 本地化

没有数据会离开你的机器。无需 API 密钥，无需外部服务 — 只需一个位于 `.codegraph/` 中的 SQLite 数据库。

准备好尝试了吗？前往[快速入门](/codegraph/getting-started/quickstart/)。
