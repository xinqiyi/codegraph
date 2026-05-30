---
title: API
description: 将 CodeGraph 作为 TypeScript 库使用。
---

CodeGraph 附带一个 TypeScript API。公共接口是 `CodeGraph` 类。

```typescript
import CodeGraph from '@colbymchenry/codegraph';

const cg = await CodeGraph.init('/path/to/project');
// 或者打开现有索引：
// const cg = await CodeGraph.open('/path/to/project');

await cg.indexAll({
  onProgress: (p) => console.log(`${p.phase}: ${p.current}/${p.total}`),
});

const results = cg.searchNodes('UserService');
const callers = cg.getCallers(results[0].node.id);
const context = await cg.buildContext('修复登录错误', {
  maxNodes: 20,
  includeCode: true,
  format: 'markdown',
});
const impact = cg.getImpactRadius(results[0].node.id, 2);

cg.watch();   // 文件更改时自动同步
cg.unwatch(); // 停止监视
cg.close();
```

## 主要方法

| 方法 | 用途 |
|---|---|
| `CodeGraph.init(path)` / `CodeGraph.open(path)` | 创建或打开项目索引 |
| `indexAll(opts)` | 完整索引，带进度回调 |
| `sync()` | 增量更新 |
| `searchNodes(query)` | 全文符号搜索 |
| `getCallers(id)` / `getCallees(id)` | 遍历调用图 |
| `getImpactRadius(id, depth)` | 变更的可传递影响 |
| `buildContext(task, opts)` | 供 AI 使用的 Markdown / JSON 上下文 |
| `watch()` / `unwatch()` | 启动/停止文件监视器 |
| `close()` | 关闭数据库连接 |
