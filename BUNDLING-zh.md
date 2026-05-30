# 分发：自包含包

CodeGraph 随应用程序一起提供了**供应商化的 Node 运行时**。由于 Node 22.5+ 内置了真正的 SQLite（`node:sqlite`，支持 WAL + FTS5），捆绑 Node 意味着：

- **无需原生构建** — `better-sqlite3` 已移除，因此零原生插件需要编译或重建。
- **无需 wasm 回退** — 因此不再出现 "database is locked" 问题（问题 #238）。
- **无 Node 版本依赖** — 应用程序始终在捆绑的 Node 上运行，无论用户安装了（或没有安装）什么版本。

## 包中包含的内容

由 [`scripts/build-bundle.sh`](scripts/build-bundle.sh) 构建 — 每个平台一个归档文件，配方相同（仅 Node 下载不同）：

```
codegraph-<target>/
  node | node.exe          # 适用于 <target> 的官方 Node 运行时
  lib/
    dist/                  # 编译后的应用程序（+ tree-sitter .wasm 语法、schema.sql）
    node_modules/          # 仅生产依赖（纯 JS / wasm — 可移植）
  bin/
    codegraph | codegraph.cmd   # 启动器 → 使用捆绑的 Node 运行应用程序
```

目标平台：`darwin-arm64`、`darwin-x64`、`linux-x64`、`linux-arm64`、`win32-x64`、`win32-arm64`。Unix 目标产生 `.tar.gz`（shell 启动器）；Windows 产生 `.zip`（`node.exe` + `.cmd` 启动器）。

```bash
scripts/build-bundle.sh linux-x64            # -> release/codegraph-linux-x64.tar.gz
scripts/build-bundle.sh win32-x64            # -> release/codegraph-win32-x64.zip
```

由于移除了 better-sqlite3 后**零原生插件**，构建包只是纯文件打包 — **任何**目标都可以在**任何**操作系统上构建（整个矩阵在一个 Linux runner 上构建）。无需考虑交叉编译；只有*运行测试*包才需要目标平台（或模拟，例如 `docker run --platform linux/amd64`）。

## 安装渠道（都提供相同的包）

1. **`curl | sh`**（[`install.sh`](install.sh)）— 无需 Node；非常适合通过 SSH 在全新的 Linux VPS 上使用。检测操作系统/架构，从 GitHub Releases 拉取归档文件，将 `codegraph` 符号链接到 PATH。重新运行以升级；`--uninstall` 以移除。
2. **npm**（[`scripts/npm-shim.js`](scripts/npm-shim.js)）— 保留 `npm i -g @colbymchenry/codegraph`。主包是一个小型 shim；包作为按平台的 `optionalDependencies`（`@colbymchenry/codegraph-<target>`，带有 `os`/`cpu` 条件）提供，因此 npm 只安装匹配的一个。shim — 由用户的 Node 运行 — 执行包，因此实际工作在捆绑的 Node 24 上运行。即使在旧版 Node 上也能工作。在 Windows 上，它直接调用捆绑的 `node.exe` 针对应用程序入口点（而不是 `.cmd` 启动器）— 现代 Node 在尝试生成 `.cmd`/`.bat` 时会抛出 `EINVAL`。
3. **Windows**（[`install.ps1`](install.ps1)）— `irm … | iex`；与 install.sh 流程相同（检测架构，从 Releases 拉取 `.zip`，添加到 PATH）。
4. **Homebrew / Scoop** — TODO（指向 Release 归档的 tap + cask）。

## 发布流程

[`.github/workflows/release.yml`](.github/workflows/release.yml) — 手动触发。从 `package.json` 读取版本，在一个 runner 上构建所有平台包，创建 GitHub Release（从 `CHANGELOG.md` 提取说明），并发布 npm shim + 按平台包。需要 `NPM_TOKEN` 仓库密钥。

仍然需要完成的：
- **代码签名** — "下载即运行"的主要缺口：macOS Gatekeeper 需要开发者 ID + 公证；Windows 需要 Authenticode。Homebrew 缓解了 macOS 的情况（处理隔离）。
- 淘汰现在仅存旧意义的 Node 版本门控（`src/bin/codegraph.ts`）— 包始终运行 Node 24，而 npm shim 不执行 tree-sitter 工作。
- 重新通过 shim 连接 `npm uninstall` 清理（代理配置 `preuninstall`）— 生成的主包不携带它。
