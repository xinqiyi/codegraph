---
title: 配置
description: CodeGraph 是零配置的 — 没有配置文件。
---

没有任何配置 — CodeGraph 是**零配置**的，**没有需要编写或维护的配置文件**。语言支持通过文件扩展名自动识别，无需为每种语言进行任何配置。

## 默认跳过的内容

- **依赖、构建和缓存目录** — `node_modules`、`vendor`、`dist`、`build`、`target`、`.venv`、`Pods`、`.next` 以及所有[支持的技术栈](/codegraph/reference/languages/)中的类似目录 — 因此图中只包含你的代码，没有第三方噪音。即使没有 `.gitignore`，这一规则同样适用。
- **`.gitignore` 中的任何内容** — 在 git 仓库中通过 git 机制生效，在非 git 项目中通过直接读取 `.gitignore` 文件（根目录及嵌套目录）来生效。
- **超过 1 MB 的文件** — 生成的 bundle、压缩后的 JS、供应商 blob 文件。

## 排除或包含更多内容

要排除其他内容，将其添加到 `.gitignore` 中。要将默认排除的目录**重新包含**进来（例如你真的希望将某个供应商依赖纳入索引），添加一个取反规则 — `!vendor/`。

默认规则统一适用，因此提交依赖或构建目录不会强制将其纳入图中 — `.gitignore` 取反规则是显式的选择加入机制。

## 数据存储位置

每个项目的数据保存在项目根目录的 `.codegraph/` 目录中，包含 SQLite 数据库（`codegraph.db`）。没有任何数据会离开你的机器。
