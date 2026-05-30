# CodeGraph A/B 基准测试——有 vs 无，每种语言 × 小/中/大

**日期：** 2026-05-24 · **分支：** `main` · **codegraph 0.9.4**

一个无头智能体（Claude Opus，`--permission-mode bypassPermissions`）回答每个仓库的一个**典型流程问题**——两次：**有** codegraph MCP 服务器，以及**没有**任何 MCP（仅内置 Read/Grep/Glob/Bash）。相同模型，相同提示；codegraph 是唯一的变量。每个单元格都**先重新索引**（针对当前 `main` HEAD 的 `dist/` 构建），因此"有"实验组反映的是已发布的 0.9.4 解析器。

## 概要

**跨 37 个单元格，codegraph 将总文件读取从 159 降至 38——减少了 76%。** 它在任何单元格中从未*增加*读取（0 回归）。其机制：几次亚毫秒级的 codegraph 调用取代了读取加 grep 的探索。

**成本基本持平——在此处"有"实验组略高**（37 个单元格汇总：有 $15.4 vs 无 $13.8）。在这些短小的单流程问题上，无实验组在 <10 次调用内解决且从未膨胀，因此它没有达到 codegraph 成本节省累积的区间，而"有"实验组则支付了固定的 MCP 开销（上下文中的工具定义 + 工具加载），短任务无法摊销这些开销。优势在于**更少的工具调用（189 vs 321，减少 41%）+ 更低的墙钟时间**（平均 **38s vs 48s**），这正是设计目标。在更难的多轮调查中，成本会转为净节省，因为无实验组的累计上下文会膨胀——见 `docs/benchmarks/call-sequence-analysis.md`。

差距随仓库大小和流程复杂度而扩大：在中/大型仓库上，无 codegraph 实验组经常**挣扎**——大量的 grep/glob、shell `find`/`grep`（Bash），偶尔生成一个**子智能体**——而有 codegraph 的实验组在 2–8 次调用内回答。在小型仓库上（几个文件），两个实验组持平或 codegraph 略慢（MCP/索引开销在流程仅涉及一两个文件时无法体现价值）——但读取仍然下降。

## 如何阅读表格

- **R / G / Gl / B / Ag** = Read / Grep / Glob / Bash / 子智能体（Task）工具调用次数。
- **cg-calls** = "有"实验组中的 codegraph MCP 调用次数（用以换取读取/grep）。
- **dur** = 墙钟秒数。**files** = 索引文件数（规模代理指标）。
- **reads saved** = 无实验组读取数 − 有实验组读取数。
- 每个实验组一次运行（一个**快照**——运行间差异是真实存在的；将 ±1–2 次读取和 ±10s 视为噪音，关注跨单元格的模式）。其中几个流程的 2 次运行/实验组概要数据见 `docs/design/dynamic-dispatch-coverage-playbook.md` §7。

## 结果

| 语言 | 大小 | 仓库 | 文件数 | **有** R/G | cg 调用 | 时长 | **无** R/G | 时长 | 节省读取 |
|---|---|---|---|---|--:|--:|--:|---|--:|--:|
| C | 大 | `c-redis` | 884 | 0R / 2G | 4 | 42s | 5R / 6G | 51s | 5 |
| C# | 小 | `aspnet-realworld` | 78 | 0R / 0G | 2 | 27s | 5R / 3G / 2Gl | 54s | 5 |
| C# | 中 | `aspnet-eshop` | 262 | 0R / 1G | 5 | 39s | 9R / 2G / 5Gl | 58s | 9 |
| C# | 大 | `aspnet-jellyfin` | 2081 | 3R / 0G | 4 | 51s | 17R / 1G / 2Gl / 17B / 1Ag | 212s | 14 |
| C++ | 中 | `cpp-leveldb` | 134 | 0R / 0G | 3 | 26s | 4R / 2G | 37s | 4 |
| Dart | 小 | `flutter_module_books` | 6 | 1R / 0G | 2 | 24s | 2R / 0G / 1Gl | 29s | 1 |
| Dart | 中 | `compass_app` | 212 | 2R / 0G / 1Gl | 2 | 42s | 3R / 0G / 2Gl | 30s | 1 |
| Go | 小 | `gin-realworld` | 21 | 0R / 0G | 5 | 35s | 4R / 3G / 1Gl | 57s | 4 |
| Go | 中 | `gin-vueadmin` | 625 | 1R / 1G | 4 | 47s | 3R / 3G / 1Gl | 44s | 2 |
| Go | 大 | `gin-gitness` | 4438 | 4R / 3G | 4 | 64s | 8R / 7G / 2Gl | 57s | 4 |
| Java | 小 | `spring-realworld` | 117 | 2R / 0G | 3 | 35s | 8R / 1G / 5B | 57s | 6 |
| Java | 中 | `spring-mall` | 536 | 1R / 0G | 5 | 39s | 2R / 4G / 2Gl | 49s | 1 |
| Java | 大 | `spring-halo` | 2444 | 1R / 2G | 8 | 60s | 4R / 1G / 6B | 52s | 3 |
| Kotlin | 小 | `kotlin-petclinic` | 43 | 0R / 0G | 2 | 37s | 3R / 0G / 1Gl | 23s | 3 |
| Kotlin | 中 | `Jetcaster` | 166 | 1R / 0G | 3 | 36s | 1R / 0G / 2Gl | 46s | 0 |
| Lua | 小 | `lualine.nvim` | 123 | 1R / 1G | 4 | 48s | 4R / 0G / 2Gl | 49s | 3 |
| Lua | 中 | `telescope.nvim` | 84 | 0R / 0G | 1 | 15s | 1R / 0G / 1Gl | 20s | 1 |
| Luau | 小 | `Knit` | 11 | 0R / 0G | 2 | 30s | 5R / 0G / 2Gl | 37s | 5 |
| PHP | 小 | `laravel-realworld` | 114 | 1R / 0G | 6 | 40s | 5R / 1G / 3Gl | 39s | 4 |
| PHP | 中 | `laravel-firefly` | 2047 | 2R / 1G | 4 | 47s | 4R / 5G / 3Gl | 75s | 2 |
| PHP | 大 | `laravel-bookstack` | 2160 | 1R / 2G | 2 | 41s | 2R / 4G / 1Gl | 50s | 1 |
| Python | 小 | `django-realworld` | 44 | 2R / 1G | 2 | 47s | 9R / 0G / 1B | 38s | 7 |
| Python | 中 | `django-wagtail` | 1672 | 2R / 0G | 4 | 45s | 8R / 3G / 3Gl / 1B | 66s | 6 |
| Python | 大 | `django-saleor` | 4429 | 2R / 2G | 4 | 52s | 4R / 6G / 1Gl | 64s | 2 |
| Ruby | 小 | `rails-realworld` | 59 | 0R / 0G | 2 | 30s | 3R / 0G / 2B | 33s | 3 |
| Ruby | 中 | `rails-spree` | 2905 | 2R / 3G / 1Gl | 5 | 43s | 3R / 3G / 2Gl / 1B | 55s | 1 |
| Ruby | 大 | `rails-forem` | 4658 | 3R / 1G | 3 | 43s | 4R / 2G / 3Gl | 48s | 1 |
| Rust | 小 | `rust-axum-realworld` | 13 | 0R / 0G | 2 | 21s | 3R / 0G / 1Gl | 38s | 3 |
| Rust | 中 | `rust-actix-examples` | 176 | 0R / 1G | 3 | 42s | 3R / 0G / 3B | 36s | 3 |
| Rust | 大 | `rust-cratesio` | 1053 | 1R / 0G | 3 | 22s | 1R / 2G | 18s | 0 |
| Scala | 小 | `computer-database` | 10 | 1R / 0G | 2 | 27s | 3R / 0G / 1Gl | 25s | 2 |
| Swift | 小 | `vapor-template` | 14 | 0R / 0G | 2 | 21s | 2R / 0G / 2Gl | 22s | 2 |
| Swift | 中 | `vapor-steampress` | 100 | 0R / 0G | 5 | 49s | 3R / 1G / 2Gl | 39s | 3 |
| Swift | 大 | `vapor-spi` | 542 | 1R / 1G | 4 | 27s | 2R / 5G | 34s | 1 |
| TypeScript/JS | 小 | `express-realworld` | 39 | 1R / 0G | 1 | 25s | 2R / 2G | 19s | 1 |
| TypeScript/JS | 中 | `excalidraw` | 643 | 1R / 0G | 3 | 55s | 7R / 5G / 3Gl / 1B | 87s | 6 |
| TypeScript/JS | 大 | `nest-immich` | 2759 | 1R / 0G | 7 | 50s | 3R / 0G / 1Gl | 44s | 2 |

**总计（37 个单元格）：** 使用 codegraph **38 次读取 / 22 次 greps**，未使用 **159 次读取 / 72 次 greps**——**读取减少 76%，grep 减少约 69%。** Codegraph 从未在任何单元格中增加读取，而无实验组还额外运行了 **52 次 glob + 37 次 shell `find`/`grep`（Bash）+ 1 个子智能体**，有实验组（**0 次 Bash，0 次子智能体**）从未需要这些。（74 次智能体运行，总计 $29.18。）

## 观察

- **最大的胜利是中/大型后端，具有真实的 route→handler→service 流程：** aspnet-jellyfin（3R / 51s vs **17R + 17 Bash + 一个生成的子智能体 / 212s**——差异最显著的一个单元格）、aspnet-eshop（0R vs 9R）、django-realworld（2R vs 9R）、spring-realworld（2R vs 8R + 5 Bash）、django-wagtail（2R vs 8R）、excalidraw（1R / 55s vs 7R / 87s）、Luau Knit（0R vs 5R）、aspnet-realworld（0R vs 5R）、c-redis（0R vs 5R）。
- **没有 codegraph 时，大型仓库使智能体挣扎：** 它回退到 shell `find`/`grep`（矩阵中 37 次 Bash 调用），在 jellyfin 上甚至生成了一个子智能体——这正是 codegraph 旨在阻止的行为。有实验组在 2–8 次 codegraph 调用中回答了这些问题，并且在任何地方使用了 **0 次 Bash 和 0 次子智能体**。
- **持平区域 = 极小型仓库**（Kotlin Jetcaster 1R/1R、Rust cratesio 1R/1R、express 1R/2R、Swift template 0R/2R）：整个流程在 1–2 个文件中，因此读取已经很廉价；codegraph 在读取上持平，有时慢几秒（MCP + 索引开销——Kotlin petclinic 37s vs 23s、cratesio 22s vs 18s）。这与设计说明一致，即 codegraph 的价值随仓库大小而增长。
- **时长在大型仓库上跟随读取**（jellyfin 51s vs 212s、excalidraw 55s vs 87s、aspnet-eshop 39s vs 58s、django-wagtail 45s vs 66s），在小型仓库上是噪音；平均墙钟时间有 codegraph 为 38s vs 无 codegraph 为 48s。
- 一些"有"实验组仍然读取了 2–4 个文件（jellyfin、gitness、forem、saleor、django）——残余部分是有文档记录的前沿（匿名处理程序、深层服务链、动态查找器）；codegraph 将智能体引导到正确的文件，然后它读取一个文件以确认细节。

## 覆盖范围说明

所有 14 个 README 框架和每个与流程相关的语言都已验证（见 playbook）。此处的大小按索引文件数划分；一些语言在语料库中缺少干净的第三大小（Dart/Kotlin = 小/中、Scala/Luau = 仅小、C = 仅大、C++ = 仅中）——这些单元格被省略而不是伪造。

## 复现

标准测试工具：`scripts/agent-eval/run-all.sh <repo> "<question>" headless`（有 = 仅 codegraph MCP，无 = 空 MCP），从 stream-json 日志解析。用于此表格的一次性矩阵驱动器和解析器位于 `/tmp/ab-matrix/`：`run.sh`（`lang|size|repo|question` 矩阵——每个单元格执行 `rm -rf .codegraph && codegraph init -i` 然后两个实验组）、`parse-matrix.mjs`（单元格 → 此表格）、`compare.mjs`（新旧差异 + 汇总）。首先从目标提交构建 `dist/`，以便 MCP 服务器加载被测代码（PATH 上的 `codegraph` 是通过 `npm link` 链接到开发 `dist/` 的）。
