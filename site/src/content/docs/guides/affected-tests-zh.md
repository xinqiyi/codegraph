---
title: CI 中受影响的测试
description: 仅运行变更实际触及的测试。
---

`codegraph affected` 可传递地追踪导入依赖，以查找哪些测试文件受一组已更改源文件的影响 — 因此 CI 可以只运行相关的测试。

```bash
codegraph affected src/utils.ts src/api.ts          # 将文件作为参数传递
git diff --name-only | codegraph affected --stdin    # 从 git diff 管道输入
codegraph affected src/auth.ts --filter "e2e/*"      # 自定义测试文件模式
```

## 选项

| 选项 | 描述 | 默认值 |
|---|---|---|
| `--stdin` | 从标准输入读取文件列表 | `false` |
| `-d, --depth <n>` | 最大依赖遍历深度 | `5` |
| `-f, --filter <glob>` | 用于识别测试文件的自定义 glob | 自动检测 |
| `-j, --json` | 输出为 JSON | `false` |
| `-q, --quiet` | 仅输出文件路径 | `false` |

## CI / 钩子示例

```bash
#!/usr/bin/env bash
AFFECTED=$(git diff --name-only HEAD | codegraph affected --stdin --quiet)
if [ -n "$AFFECTED" ]; then
  npx vitest run $AFFECTED
fi
```
