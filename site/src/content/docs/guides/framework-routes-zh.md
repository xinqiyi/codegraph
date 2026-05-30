---
title: 框架路由
description: CodeGraph 将 URL 模式链接到处理它们的处理器。
---

CodeGraph 检测 Web 框架的路由文件，并通过 `references` 边发出链接到其处理器类或函数的 `route` 节点。查询视图或控制器的调用者时，会显示绑定它的 URL 模式。

| 框架 | 识别的形式 |
|---|---|
| **Django** | `path()`、`re_path()`、`url()`、`include()` 在 `urls.py` 中（CBV `.as_view()`、点号路径） |
| **Flask** | `@app.route('/path', methods=[…])`、蓝图路由 |
| **FastAPI** | `@app.get(…)`、`@router.post(…)`、所有标准方法 |
| **Express** | `app.get(…)`、`router.post(…)` 含中间件链 |
| **NestJS** | `@Controller` + `@Get/@Post/…`、GraphQL 解析器、消息/事件模式、WebSocket 订阅 |
| **Laravel** | `Route::get()`、`Route::resource()`、`Controller@action`、元组语法 |
| **Drupal** | `*.routing.yml` 路由；`.module`/`.theme`/`.install`/`.inc` 中的 `hook_*` 实现 |
| **Rails** | `get '/x', to: 'users#index'`、哈希火箭语法 |
| **Spring** | 方法上的 `@GetMapping`、`@PostMapping`、`@RequestMapping` |
| **Gin / chi / gorilla / mux** | `r.GET(…)`、`router.HandleFunc(…)` |
| **Axum / actix / Rocket** | `.route("/x", get(handler))` |
| **ASP.NET** | 操作方法上的 `[HttpGet("/x")]` 属性 |
| **Vapor** | `app.get("x", use: handler)` |
| **React Router** / **SvelteKit** | 路由组件节点 |

路由解析是自动的 — 无需任何配置。如果某个框架文件被识别，其路由会在下一次索引或同步后出现在图中。
