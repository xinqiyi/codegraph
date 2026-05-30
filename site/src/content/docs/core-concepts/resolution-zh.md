---
title: 解析与框架
description: CodeGraph 如何连接引用并将路由链接到处理器。
---

提取产生节点和原始边；**解析**将名称转化为真正的连接。

## 引用解析

解析之后，CodeGraph 会解析以下内容：

- **导入** → 它们所指向的源文件（包括 tsconfig 路径别名和 cargo 工作空间成员）。
- **调用** → 通过导入解析和名称匹配找到它们的定义。
- **继承** → 类型之间的 `extends` / `implements` 关系。

## 框架感知

CodeGraph 能识别 Web 框架的路由文件，并发出通过 `references` 边链接到其处理器类或函数的 `route` 节点 — 因此查询视图或控制器的调用者时，会显示绑定它的 URL 模式。有关已识别框架的完整列表，请参阅[框架路由](/codegraph/guides/framework-routes/)。

## 动态分发覆盖

静态解析会遗漏计算调用和间接调用，因此流程可能在动态分发处中断。CodeGraph 使用合成器桥接了多个此类边界，使流程能够端到端连接：

- 回调 / 观察者注册
- `EventEmitter` 通道
- React 重新渲染（`setState` → `render`）
- JSX 子组件（`render` → 子组件）
- Django ORM 描述符

每个合成的边都标记有 `provenance: 'heuristic'` 以及接线位置，并在路径经过时以内联方式显示。
