# Starlight 入门套件：基础

[![基于 Starlight 构建](https://astro.badg.es/v2/built-with-starlight/tiny.svg)](https://starlight.astro.build)

```
npm create astro@latest -- --template starlight
```

> 🧑‍🚀 **经验丰富的宇航员？** 删除此文件。尽情探索吧！

## 🚀 项目结构

在你的 Astro + Starlight 项目中，你会看到以下文件夹和文件：

```
.
├── public/
├── src/
│   ├── assets/
│   ├── content/
│   │   └── docs/
│   └── content.config.ts
├── astro.config.mjs
├── package.json
└── tsconfig.json
```

Starlight 会在 `src/content/docs/` 目录中查找 `.md` 或 `.mdx` 文件。每个文件会根据其文件名被暴露为一个路由。

图片可以添加到 `src/assets/` 目录中，并通过相对链接嵌入到 Markdown 中。

静态资源（如 favicon）可以放在 `public/` 目录中。

## 🧞 命令

所有命令均从项目根目录的终端运行：

| 命令 | 操作 |
| :------------------------ | :----------------------------------------------- |
| `npm install` | 安装依赖 |
| `npm run dev` | 在 `localhost:4321` 启动本地开发服务器 |
| `npm run build` | 构建生产站点到 `./dist/` |
| `npm run preview` | 在部署前本地预览构建结果 |
| `npm run astro ...` | 运行 CLI 命令，如 `astro add`、`astro check` |
| `npm run astro -- --help` | 获取 Astro CLI 帮助 |

## 👀 想了解更多？

查看 [Starlight 文档](https://starlight.astro.build/)，阅读 [Astro 文档](https://docs.astro.build)，或加入 [Astro Discord 服务器](https://astro.build/chat)。
