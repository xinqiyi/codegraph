---
title: 工作原理
description: 提取、存储、解析和自动同步的流水线。
---

CodeGraph 通过四个阶段将源代码转换为可查询的图。

```
文件 → 提取（tree-sitter）→ 数据库（节点/边/文件）
            ↓
      解析（导入、名称匹配、框架模式）
            ↓
      图查询（调用者、被调用者、影响范围）
            ↓
      上下文构建（供 AI 消费的 markdown / JSON）
```

## 1. 提取

[tree-sitter](https://tree-sitter.github.io/) 将源代码解析为 AST。特定于语言的查询提取出**节点**（函数、类、方法、类型……）和**边**（调用、导入、继承、实现）。繁重的解析工作会在主线程之外进行。

## 2. 存储

所有内容都会存入本地的 SQLite 数据库（`.codegraph/codegraph.db`），并支持 FTS5 全文搜索。CodeGraph 在可用时使用原生的 `better-sqlite3`，并在背后透明地回退到 WASM 后端；通过 `codegraph status` 可以查看当前使用的是哪个后端。

## 3. 解析

提取之后，引用会被解析：函数调用 → 定义、导入 → 源文件、类继承以及特定于框架的模式。一些动态分发边界（回调、观察者、React 重新渲染、JSX 子组件）由合成器桥接，使流程能够端到端连接。请参阅[解析与框架](/codegraph/core-concepts/resolution/)。

## 4. 自动同步

MCP 服务器使用原生操作系统文件事件（FSEvents / inotify / ReadDirectoryChangesW）监视你的项目。变更会经过防抖处理、过滤到源文件，并进行增量同步 — 在你编码时，图始终保持最新，无需任何配置。
