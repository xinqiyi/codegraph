---
title: 知识图谱
description: 构建图谱所使用的节点和边类型。
---

CodeGraph 存储三种内容：**节点**（符号和文件）、**边**（它们之间的关系）和**文件**。每个节点和边都带有精确的 `kind`，来自一个固定的词汇表，因此查询在所有语言中都是一致的。

## 节点类型

`file`、`module`、`class`、`struct`、`interface`、`trait`、`protocol`、`function`、`method`、`property`、`field`、`variable`、`constant`、`enum`、`enum_member`、`type_alias`、`namespace`、`parameter`、`import`、`export`、`route`、`component`。

## 边类型

`contains`、`calls`、`imports`、`exports`、`extends`、`implements`、`references`、`type_of`、`returns`、`instantiates`、`overrides`、`decorates`。

## 来源

大多数边直接来自 AST。少数边 — 在静态解析无法追踪的动态分发边界处 — 是**合成的**，并标记有 `provenance: 'heuristic'` 以及创建它们的接线位置。这些边会以内联方式显示在 `trace`、`node` 追踪和 `context` 调用路径中，使代理能够确切看到连接的来源。

## 查询图谱

- **搜索** — 按名称搜索符号（FTS5）。
- **调用者 / 被调用者** — 逐跳遍历调用图。
- **影响范围** — 计算变更所影响的可传递半径。
- **追踪** — 一次调用返回两个符号之间的完整调用路径。

有关如何运行这些查询，请参阅 [CLI](/codegraph/reference/cli/) 和 [MCP 服务器](/codegraph/reference/mcp-server/) 参考文档。
