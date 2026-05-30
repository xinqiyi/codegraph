---
title: CLI
description: 每个 CodeGraph 命令及其接受的标志。
---

```bash
codegraph                         # 运行交互式安装程序
codegraph install                 # 运行安装程序（显式）
codegraph uninstall               # 从代理中移除 CodeGraph（安装的逆向操作）
codegraph init [path]             # 在项目中初始化（--index 同时进行索引）
codegraph uninit [path]           # 从项目中移除 CodeGraph（--force 跳过提示）
codegraph index [path]            # 完整索引（--force 重新索引，--quiet 减少输出）
codegraph sync [path]             # 增量更新
codegraph status [path]           # 显示统计信息
codegraph query <search>          # 搜索符号（--kind、--limit、--json）
codegraph files [path]            # 显示文件结构（--format、--filter、--max-depth、--json）
codegraph context <task>          # 为 AI 构建上下文（--format、--max-nodes）
codegraph callers <symbol>        # 查找哪些内容调用了某个函数/方法（--limit、--json）
codegraph callees <symbol>        # 查找某个函数/方法调用了哪些内容（--limit、--json）
codegraph impact <symbol>         # 分析更改某个符号会影响到哪些代码（--depth、--json）
codegraph affected [files...]     # 查找受更改影响的测试文件
codegraph serve --mcp             # 启动 MCP 服务器
```

## 查询命令

`query`、`callers`、`callees` 和 `impact` 都接受 `--json` 参数以输出机器可读的结果。

```bash
codegraph query UserService --kind class --limit 10
codegraph callers handleRequest --json
codegraph impact AuthMiddleware --depth 3
```

## affected

可传递地追踪导入依赖，以查找哪些测试文件受已更改源文件的影响。有关选项和 CI 示例，请参阅[CI 中受影响的测试](/codegraph/guides/affected-tests/)。
