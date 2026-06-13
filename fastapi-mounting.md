# FastAPI 路由挂载机制分析

> 本文分析 Gradio 后端路由挂载机制横跨**服务（Server/App）**与**资源（Static/Assets）**的架构，
> 结合代码梳理**应用接入**、**接口生成**与**前端资源服务**三者之间的边界。

---

## 一、整体架构概览

Gradio 的路由体系基于 FastAPI 构建，分为三个核心层次：

```
┌───────────────────────────────────────────────────────────┐
│                   应用接入层 (Entry)                      │
│  Server(mode=server)  │  Blocks.launch  │ mount_gradio_app │
├───────────────────────────────────────────────────────────┤
│                   接口生成层 (API)                        │
│  App.create_app  →  APIRouter(/gradio_api)  →  业务路由   │
│           /run  /call  /queue/join  /upload  ...          │
├───────────────────────────────────────────────────────────┤
│                前端资源服务层 (Static/Assets)              │
│  /static  /assets  /favicon.ico  /file=  /svelte          │
│  ── 可剥离为独立 StaticWorkerPool 多进程服务（需 Node 代理分流）──│
└───────────────────────────────────────────────────────────┘
```

---

## 二、应用接入层：三种入口与挂载方式

### 2.1 `Server` 类 —— Server Mode 入口

**位置**：[server.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/server.py#L28-L338)

`Server` 继承自 `App`（而 `App` 继承自 `fastapi.FastAPI`），提供 **Server Mode**：
用户直接用 `Server()` 实例替代 FastAPI 应用，通过 `@server.api()` 装饰器注册 API。

```python
@document("api", "launch")
class Server(App):
    """Server is the Gradio API engine exposed on a FastAPI application (Server mode).
    It inherits from FastAPI, so all standard FastAPI methods work directly."""
```

**核心机制**：
- `_deferred_apis` 列表延迟注册 API 函数（[server.py:172](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/server.py#L172-L172)）
- `api()` 装饰器将函数及配置暂存到 `_deferred_apis`
- `launch()` 时创建内部 `Blocks`，通过 `gr_api()` 将延迟注册的函数转为 Blocks 事件函数，
  最终调用 `blocks.launch(_app=self)` 复用 Blocks 的启动流程（[server.py:281-290](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/server.py#L281-L290)）

```python
def launch(self, ...) -> tuple[App, str, str]:
    from gradio.blocks import Blocks
    from gradio.events import api as gr_api

    with Blocks(mode="server") as blocks:
        for fn, api_kwargs in self._deferred_apis:
            gr_api(fn=fn, **api_kwargs)

    return blocks.launch(_app=self, ...)
```

> **边界要点**：`Server` 是 FastAPI 的直接子类，因此 `.get()`/`.post()`/`.include_router()`
> 等原生方法全部可用。它通过「延迟注册 + Blocks 桥接」将 API 函数纳入 Gradio 的
> 队列、SSE 流式、并发控制体系。

---

### 2.2 `mount_gradio_app` —— 挂载到现有 FastAPI

**位置**：[routes.py:2429-2596](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2429-L2596)

`mount_gradio_app(app, blocks, path)` 是将 Gradio 应用**作为子应用**挂载到已有
FastAPI 实例的标准方式。它使用 FastAPI 原生的 `app.mount(path, app)` 机制。

```python
def mount_gradio_app(
    app: fastapi.FastAPI,
    blocks: gradio.Blocks,
    path: str,
    ...
) -> fastapi.FastAPI:
    blocks.custom_mount_path = path
    ...
    gradio_app = App.create_app(blocks, ...)
    ...
    app.mount(path, gradio_app)
    return app
```

**核心步骤**：

| 步骤 | 代码位置 | 说明 |
|------|---------|------|
| 配置 Blocks | [routes.py:2510-2574](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2510-L2574) | 设置 `custom_mount_path`、`footer_links`、`auth`、主题/CSS/JS 等 |
| 创建 Gradio 子 App | [routes.py:2575-2580](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2575-L2580) | 调用 `App.create_app()` 生成独立的 FastAPI 子应用 |
| 融合生命周期 | [routes.py:2581-2593](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2581-L2593) | 将子应用的 lifespan 嵌入父应用的 lifespan，确保 startup/shutdown 事件正确执行 |
| 执行挂载 | [routes.py:2595](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2595-L2595) | `app.mount(path, gradio_app)` —— FastAPI/Starlette 原生 ASGI 挂载 |

**生命周期融合代码**：

```python
old_lifespan = app.router.lifespan_context

@contextlib.asynccontextmanager
async def new_lifespan(app: FastAPI):
    async with old_lifespan(app) as state:
        async with gradio_app.router.lifespan_context(gradio_app):
            gradio_app.get_blocks().run_startup_events()
            await gradio_app.get_blocks().run_extra_startup_events()
            yield state

app.router.lifespan_context = new_lifespan
```

> **边界要点**：`mount_gradio_app` 是**应用层边界**。它将 Gradio 封装为一个
> 完全自包含的 ASGI 子应用，通过 FastAPI 的 `mount` 机制与主应用隔离。
> 父应用与子应用仅共享「生命周期事件」和「路径前缀」，路由、状态、中间件
> 各自独立。

---

### 2.3 `Blocks.launch` —— 独立启动入口

**位置**：[blocks.py:2602-3150](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2602-L3150)

`Blocks.launch()` 是最常用的启动方式，内部调用 `App.create_app()` 创建 FastAPI 应用，
再通过 `http_server.start_server()` 启动 uvicorn。

**启动流程**：

```
Blocks.launch()
    │
    ├─ 配置主题 / CSS / JS / 认证 / 队列
    ├─ 决定 SSR 模式 → 启动 Node 服务（可选）
    ├─ App.create_app(blocks, _app=...)  ← 创建/复用 FastAPI 应用
    ├─ http_server.start_server(app)     ← 启动 uvicorn
    ├─ 启动 StaticWorkerPool（可选）     ← 静态资源独立进程
    └─ 启动 Node 前端代理（可选）        ← SSR 模式
```

**关键代码片段**（[blocks.py:2922-2930](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2922-L2930)）：

```python
self.server_app = self.app = App.create_app(
    self,
    app=_app,
    auth_dependency=auth_dependency,
    app_kwargs=app_kwargs,
    strict_cors=strict_cors,
    mcp_server=mcp_server,
    debug=debug,
)
```

> **边界要点**：`Blocks.launch` 是**服务启动边界**。它组装所有配置、决定架构
> （单进程 / SSR / 静态工作池），然后调用 `App.create_app` 和 `http_server.start_server`
> 完成服务化。

---

## 三、接口生成层：`App.create_app` 与路由构造

### 3.1 `App.create_app` —— 路由总装工厂

**位置**：[routes.py:358-2328](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L358-L2328)

`App.create_app(blocks, ...)` 是路由挂载的核心工厂函数。它创建一个 `App` 实例
（FastAPI 子类），然后在其中注册所有 Gradio 路由。

```python
@staticmethod
def create_app(
    blocks: gradio.Blocks,
    app: App | None = None,
    app_kwargs: dict[str, Any] | None = None,
    auth_dependency: Callable[[fastapi.Request], str | None] | None = None,
    strict_cors: bool = True,
    mcp_server: bool | None = None,
    debug: bool = False,
) -> App:
```

**路由构造总览**：

```
App.create_app()
    │
    ├─ 设置 MCP 服务 / lifespan
    ├─ 创建 APIRouter(prefix="/gradio_api")  ← API 路由组
    ├─ app.configure_app(blocks)              ← 关联 Blocks
    ├─ 添加中间件 (CORS, Brotli)
    │
    ├─ ── 认证类路由 ──
    │   /login  /logout  /token  /user  /login_check
    │
    ├─ ── 页面类路由（挂载在 app 根上）──
    │   GET /               ← 主页面（HTML 模板渲染）
    │   GET /{page}         ← 多页面路由
    │   GET /config         ← 配置 JSON
    │
    ├─ ── 静态资源路由（挂载在 app 根上）──
    │   GET /static/{path}  GET /assets/{path}
    │   GET /favicon.ico    GET /svelte/{path}
    │
    ├─ ── API 路由（挂载在 router 上，前缀 /gradio_api）──
    │   /info              ← API 信息
    │   /openapi.json      ← OpenAPI schema
    │   /upload            ← 文件上传
    │   /run/{api_name}    ← 直接运行（非队列）
    │   /call/{api_name}   ← 简化调用（入队）
    │   /queue/join        ← 入队
    │   /queue/data        ← SSE 数据拉取
    │   /cancel            ← 取消
    │   /heartbeat/{hash}  ← 心跳保活
    │   /stream/...        ← 媒体流（HLS）
    │   /file={path}       ← 文件获取
    │   /proxy={url}       ← 反向代理
    │   ...
    │
    └─ app.include_router(router)  ← 将 API 路由组纳入 FastAPI
```

---

### 3.2 两类路由的边界：根路由 vs API 路由

Gradio 的路由分为两个**命名空间**，边界清晰：

| 路由类型 | 挂载位置 | 前缀 | 职责 | 示例 |
|---------|---------|------|------|------|
| **根路由** | `app` (FastAPI) | 无 | 页面、静态资源、登录 | `/`、`/config`、`/static/...`、`/login` |
| **API 路由** | `router` (APIRouter) | `/gradio_api` | 业务接口、队列、文件 | `/gradio_api/run/...`、`/gradio_api/queue/join` |

**关键代码**（[routes.py:383](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L383-L383) 和 [routes.py:2327](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2327-L2327)）：

```python
router = APIRouter(prefix=API_PREFIX)   # API_PREFIX = "/gradio_api"
...
app.include_router(router)              # 最后统一纳入
```

> **边界要点**：`API_PREFIX = "/gradio_api"` 是**接口层边界**。所有 Gradio 业务 API
> 都在此前缀下，与页面路由、静态资源路由物理隔离。这种设计便于：
> - 权限控制（`login_check` 依赖统一注入 API router）
> - 反向代理分流
> - 未来独立微服务化

---

### 3.3 接口生成机制：从 Blocks 配置到路由

Gradio 的 API 不是显式定义的，而是**从 Blocks 的依赖关系动态生成**。

**生成链路**：

```
Blocks 配置 (dependencies / fns)
    │
    └─ blocks.get_api_info()  ← 从函数签名推断输入输出 schema
           │
           └─ /gradio_api/info  ← 暴露 API 元数据
                  │
                  └─ /gradio_api/openapi.json  ← 动态生成 OpenAPI schema
```

**路由调用链路**（以 `/call/{api_name}` 为例）：

```python
# routes.py:1342-1355
@router.post("/call/{api_name}", dependencies=[Depends(login_check)])
async def simple_predict_post(api_name: str, body: SimplePredictBody, ...):
    full_body = PredictBody(**body.model_dump(), simple_format=True)
    fn = route_utils.get_fn(blocks=app.get_blocks(), api_name=api_name, body=full_body)
    full_body.fn_index = fn._id
    return await queue_join_helper(full_body, request, username)
```

`route_utils.get_fn()` 根据 `api_name` 从 `blocks.fns` 中查找对应的 `BlockFunction`，
然后通过队列系统执行。

> **边界要点**：接口层与应用层的边界是 `app.get_blocks()` 和 `blocks.fns`。
> 路由层只负责 HTTP 协议解析、认证、参数校验，**不持有业务逻辑**。
> 业务逻辑全部封装在 `Blocks.fns`（`BlockFunction` 对象）中，由 `route_utils.call_process_api`
> 统一调度。

---

## 四、前端资源服务层：静态资源与可剥离架构

### 4.1 三类静态资源路由

**位置**：[routes.py:977-1053](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L977-L1053)

| 路由 | 源路径 | 用途 |
|------|--------|------|
| `GET /static/{path:path}` | `STATIC_PATH_LIB` | Gradio 内置静态资源（图标、字体等） |
| `GET /assets/{path:path}` | `BUILD_PATH_LIB` | 前端构建产物（JS/CSS 等） |
| `GET /favicon.ico` | `blocks.favicon_path` | 网站图标 |
| `GET /svelte/{path:path}` | `BUILD_PATH_LIB/svelte` | Svelte 开发模式资源 |
| `GET /file={path}` | 上传目录 / allowed_paths | 用户文件服务 |

所有静态资源通过 `file_response()` 和 `file_fetch()` 统一处理，支持路径安全校验、
MIME 类型推断、缓存策略。

---

### 4.2 StaticWorkerPool：静态资源独立进程

**位置**：[static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py)

当 `num_workers > 0` 时，Gradio 会启动独立的多进程静态文件服务，将文件 I/O 和
上传下载从主 Gradio 服务器剥离。

```python
class StaticWorkerPool:
    """Manages N static file server processes."""
    def __init__(self, num_workers: int, config: StaticServerConfig, ports: list[int]):
        ...
    def start(self): ...       # spawn N 个子进程
    def get_next_url(self) -> str: ...  # 预留的轮询接口（详见下方说明）
    def shutdown(self): ...
```

> ⚠️ **关于 `get_next_url()` 的说明**：该方法在
> [static_server.py:229-233](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L229-L233)
> 定义，实现了 Round-Robin 端口选择逻辑，但在当前代码库中**零调用点**。
> 它是 Python 侧预留的分流接口，实际线上流量分发由 Node 代理侧的
> `classifyRoute` + `workerIndex` 完成（见第九章 9.2 节）。不应将此预留接口
> 等同于运行时的线上分流行为。

每个静态工作进程是一个**独立的 FastAPI 应用**（[static_server.py:52-167](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L52-L167)），
包含以下路由：

```
Static Worker FastAPI App
    ├─ GET /svelte/{path}
    ├─ GET /static/{path}
    ├─ GET /assets/{path}
    ├─ GET /favicon.ico
    ├─ GET/HEAD /gradio_api/file={path}
    ├─ GET/HEAD /file={path}
    ├─ POST /gradio_api/upload
    ├─ POST /upload
    ├─ GET /gradio_api/upload_progress
    ├─ GET /upload_progress
    └─ GET /health
```

**启动位置**：[blocks.py:2983-3020](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2983-L3020)

```python
if resolved_num_workers is not None and resolved_num_workers >= 1:
    static_config = StaticServerConfig(...)
    self._static_worker_pool = StaticWorkerPool(
        num_workers=resolved_num_workers,
        config=static_config,
        ports=worker_ports,
    )
    self._static_worker_pool.start()
```

> **边界要点**：静态资源服务是**可独立部署的资源层边界**——但「可独立部署」仅在实际有
> 流量分发器（即 Node 代理）时才生效。在拓扑 3（SSR + Node 代理）下，
> `StaticWorkerPool` 通过「多进程 + Node 侧轮询分发」实现水平扩展，
> 将 I/O 密集型的文件服务从 CPU/状态密集型的主服务中解耦。在拓扑 2（无 Node）
> 下，Worker 进程虽已启动，但无流量入口指向它（详见第十一章 11.1 节）。

---

### 4.3 Node SSR 代理：前端渲染的另一种边界

当 `ssr_mode=True` 且 Node 可用时，Gradio 会启动 Node 服务作为**前端代理**，
Python 退居为内部 API 服务。

**架构**（[blocks.py:2862-2906](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2862-L2906)）：

```
用户请求 → Node (前端代理, 用户端口)
              ├─ 静态资源 → 直接由 Node 提供 (SSR)
              └─ API 请求 → Python (内部端口)
                            └─ Static Workers (文件服务)
```

关键设计：**Node 代理延迟启动**，等 Python 和静态工作进程全部就绪后再启动 Node，
避免启动窗口期出现 502 错误（[blocks.py:3027-3030](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L3027-L3030)）。

---

## 五、三层边界总结

### 5.1 边界总表

| 边界 | 分隔的层次 | 关键接口 | 核心代码 |
|------|-----------|---------|---------|
| **应用接入边界** | 应用逻辑 ↔ 服务框架 | `Blocks` ↔ `App` ↔ `FastAPI` | `mount_gradio_app()`、`Server.launch()`、`Blocks.launch()` |
| **接口生成边界** | 业务逻辑 ↔ HTTP 接口 | `blocks.fns` ↔ APIRouter | `App.create_app()`、`route_utils.get_fn()`、`call_process_api()` |
| **资源服务边界** | API 服务 ↔ 静态资源 | `/static` `/assets` `/file=` | `file_response()`、`StaticWorkerPool`、`/gradio_api/upload` |

---

### 5.2 数据流向

```
        应用接入层                  接口生成层                  资源服务层
  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
  │  Server / Blocks │─────▶│  App (FastAPI)   │─────▶│  Static Workers  │
  │  .api() .launch()│      │  APIRouter       │      │  file_response   │
  │  mount_gradio_app│      │  /gradio_api/*   │      │  upload_fn       │
  └──────────────────┘      └──────────────────┘      └──────────────────┘
           │                        │                        │
           │  配置与生命周期        │  HTTP 协议与认证        │  文件 I/O
           │  注入与桥接            │  队列与流式            │  水平扩展
           ▼                        ▼                        ▼
      业务应用代码            Gradio 核心框架           静态文件/上传下载
```

---

### 5.3 设计洞察

1. **「App 是 FastAPI 子类」是架构基石**：`App` 继承 `FastAPI`，`Server` 继承 `App`，
   使得 Gradio 既可以作为独立服务，也可以作为子应用挂载，还可以直接使用 FastAPI
   原生能力。

2. **「延迟注册 + Blocks 桥接」实现了渐进式 API**：`Server.api()` 装饰器在定义时
   不立即生成路由，而是存入 `_deferred_apis`，到 `launch()` 时才通过 Blocks 体系
   统一生成。这保证所有 API 都走同一套队列/并发/SSE 管线。

3. **资源层可剥离是性能架构的关键（拓扑 3 下生效）**：在 Node 代理模式下，
   `StaticWorkerPool` 把文件 I/O 从主进程移出，Gradio 可以支撑高并发的静态资源访问
   而不阻塞核心 API。但在无 Node 代理的拓扑下，Worker 进程虽已启动却不接收线上流量，
   文件 I/O 仍在主进程中处理。

4. **`/gradio_api` 前缀是重要的隔离设计**：所有业务 API 统一前缀，便于鉴权、
   监控、代理分流，也为未来 API 服务独立部署预留了架构空间。

---

---

## 七、挂载路径、代理前缀与公开访问地址

本节顺着代码讲清**三个前缀概念**如何分层作用，并最终影响浏览器看到的公开访问地址。

### 7.1 三个前缀概念

| 前缀变量 | 设置方式 | 含义 | 典型值 |
|---------|---------|------|--------|
| `blocks.custom_mount_path` | `mount_gradio_app(app, blocks, path="/app")` 第 3 参数 | FastAPI `app.mount()` 时的 ASGI 子路径，**仅当作为子应用挂载时才有值** | `/app`、`/demo` |
| `blocks.root_path` / `app.root_path` | `mount_gradio_app(..., root_path=...)` 参数；或反向代理通过 ASGI `root_path` 注入 | **公开访问前缀**，可以是相对路径或完整 URL。优先级高于 `custom_mount_path` | `/myapp`、`https://example.com/myapp` |
| `request.scope["root_path"]` | 由 uvicorn 等 ASGI 服务器根据反向代理设置自动注入（如 nginx 的 `X-Forwarded-Prefix`） | 运行时动态前缀，代表反向代理剥离的路径段 | `/space/xxx` |

**关键代码**：

- 设置位置（[routes.py:2517](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2517-L2517) 和 [routes.py:2549-2550](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L2549-L2550)）：
```python
blocks.custom_mount_path = path          # 子应用挂载路径
if root_path is not None:
    blocks.root_path = root_path         # 手动指定公开前缀
```

- App 初始化时读取（[routes.py:278](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L278-L278)）：
```python
self.root_path = blocks.root_path or ""
```

---

### 7.2 `get_root_url()`：公开地址的最终解析器

**位置**：[route_utils.py:488-513](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/route_utils.py#L488-L513)

所有需要生成**公开 URL** 的路由（主页、/config、/info、文件响应等）都通过 `get_root_url()` 得到浏览器应使用的根地址。

**解析优先级链**：

```
get_root_url(request, route_path, root_path)
    │
    ├─ ① 如果 root_path 是完整 URL (http:// 或 https://)
    │      → 直接返回该 URL，最高优先级
    │
    ├─ ② 从请求中提取 origin (get_request_origin):
    │      ├─ 优先读 x-forwarded-host 头
    │      ├─ 其次读 x-gradio-server 头
    │      └─ 最后用 request.url 本身
    │      然后：如果有 x-forwarded-proto=https → 切换协议为 https
    │            如果 URL 以 route_path 结尾 → 截断尾部路由路径
    │
    └─ ③ 如果 root_path 是相对路径（如 "/myapp"）：
           → 将 origin 的 path 部分替换为 root_path
           （如果 path 已经等于 root_path 则不重复加）
```

**辅助函数 `get_request_origin`**（[route_utils.py:438-458](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/route_utils.py#L438-L458)）：

```python
x_forwarded_host = get_first_header_value(request, "x-forwarded-host")
x_gradio_server  = get_first_header_value(request, "x-gradio-server")
root_url = (
    f"http://{x_forwarded_host}"
    if x_forwarded_host
    else str(x_gradio_server or request.url)
)
# ... 处理 x-forwarded-proto=https 的切换
# ... 去除尾部的 route_path 段
```

**优先级总结**：
```
完整 URL root_path > x-forwarded-host + x-forwarded-proto > request.url
root_path (相对) 会强制替换 path 部分
```

> ⚠️ **验证标注**：`custom_mount_path` **并非** `get_root_url()` 的内部 fallback。
> 它只在 `/` 和 `/config` 两个路由的**调用侧**以 `or` 链形式传入：
> `root_path=app.root_path or request.scope.get("root_path") or blocks.custom_mount_path`
> （[routes.py:616-618](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L616-L618)、[routes.py:961-963](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L961-L963)）。
> 而 `/gradio_api/info` 等路由**只用 `app.root_path`**，不兜底 `custom_mount_path`
> （[routes.py:730](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L730-L730)）。
> 这意味着挂载子应用但未设 `root_path` 时，API 信息页中的示例 URL 可能缺少挂载前缀。

---

### 7.3 三种部署模式下的地址拼接示例

设 Gradio 内部端口 7860，`custom_mount_path = "/demo"`，分别看三种场景：

#### 场景 A：直接访问，无反向代理
```
浏览器 → http://localhost:7860/demo

对 / 和 /config 路由（custom_mount_path 参与兜底）：
  root_path=""，x-forwarded-host=空，request.url = http://localhost:7860/demo/
  → get_request_origin: origin = http://localhost:7860
  → root_path 参数 = app.root_path(空) or scope.root_path(空) or custom_mount_path("/demo")
  → get_root_url 内部：root_path="/demo" 替换 origin.path → root = "http://localhost:7860/demo"

对 /gradio_api/info 路由（custom_mount_path 不参与）：
  root_path 参数 = app.root_path(空)（无兜底）
  → get_root_url 内部：root_path 为空，不做 path 替换
  → root = "http://localhost:7860"（可能缺少 /demo 前缀）
```

#### 场景 B：nginx 反代 + root_path 手动指定
```
nginx https://example.com/myapp  →  http://127.0.0.1:7860/demo
设置 X-Forwarded-Host: example.com
设置 X-Forwarded-Proto: https
调用 mount_gradio_app(..., root_path="https://example.com/myapp")

各路由内 get_root_url() 解析：
  root_path = "https://example.com/myapp" (完整 URL)
  → 直接返回，最高优先级命中
  → 最终 root = "https://example.com/myapp"
```

#### 场景 C：挂载到子应用但未显式设置 root_path
```
FastAPI 主应用 localhost:8000
  └─ mount("/tools/gradio", gradio_app)  ← custom_mount_path="/tools/gradio"

浏览器访问 http://localhost:8000/tools/gradio

对 / 和 /config 路由：
  FastAPI mount 会自动将 scope["root_path"] 设为 "/tools/gradio"
  → root_path 参数 = app.root_path(空) or scope["root_path"]("/tools/gradio")
  → custom_mount_path 无需兜底（scope.root_path 已经命中）
  → root = "http://localhost:8000/tools/gradio"

对 /gradio_api/info 路由：
  root_path 参数 = app.root_path(空)（无兜底，无 scope["root_path"]）
  → get_request_origin: request.url = http://localhost:8000/tools/gradio/gradio_api/info
  → 截去 route_path 后 origin = http://localhost:8000
  → root_path 为空，不做 path 替换
  → root = "http://localhost:8000"（缺少 /tools/gradio 前缀）

  ⚠️ 此场景下 /info 路由的示例代码 URL 不正确是已知行为，
  需要用户显式设置 root_path="/tools/gradio" 来修正。
```

---

## 八、前缀如何影响页面、配置、文件地址

所有需要对外暴露的 URL 都**不直接硬编码路径**，而是通过统一的前缀注入流程。

### 8.1 页面路由 `/` 的前缀注入

**位置**：[routes.py:603-681](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L603-L681)

```python
@app.head("/", response_class=HTMLResponse)
@app.get("/", response_class=HTMLResponse)
def main(request, user, page, deep_link):
    # ① 计算公开 root
    root = route_utils.get_root_url(
        request=request,
        route_path=f"/{page}",
        root_path=app.root_path
                  or request.scope.get("root_path")
                  or blocks.custom_mount_path,   # custom_mount_path 仅在此兜底
    )
    # ② 构造 config（包含 components / dependencies / layout）
    config = utils.safe_deepcopy(blocks.config)
    config["username"] = user
    # ③ ★ 关键：把 root 注入 config，递归修正所有文件 URL
    config = route_utils.update_root_in_config(config, root)
    # ④ 渲染 HTML 模板
    return templates.TemplateResponse("frontend/index.html", {
        "config": config,      # 里面已带 root 和完整的文件 URL
        "gradio_api_info": ...
    })
```

**`update_root_in_config` 的作用**（[route_utils.py:826-836](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/route_utils.py#L826-L836)）：

```python
def update_root_in_config(config, root):
    previous_root = config.get("root")
    if previous_root is None or previous_root != root:
        config["root"] = root
        config = processing_utils.add_root_url(config, root, previous_root)
    return config
```

它做两件事：
1. 把新的 `root` 写入 `config["root"]` 字段（前端 JS 从这里读取 API 根地址）
2. 递归遍历 config 中所有「文件对象」（带 `url` 字段的 dict），给它们的 URL 拼接 root 前缀

---

### 8.2 文件地址的前缀拼接：`add_root_url`

**位置**：[processing_utils.py:626-635](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/processing_utils.py#L626-L635)

```python
def add_root_url(data, root_url, previous_root_url):
    def _add_root_url(file_dict):
        # ① 如果旧前缀存在，先剥离（避免重复拼接）
        if previous_root_url and file_dict["url"].startswith(previous_root_url):
            file_dict["url"] = file_dict["url"][len(previous_root_url):]
        # ② 如果已经是完整的外部 URL，直接跳过
        elif client_utils.is_http_url_like(file_dict["url"]):
            return file_dict
        # ③ 前缀拼接
        file_dict["url"] = f"{root_url}{file_dict['url']}"
        return file_dict

    return client_utils.traverse(data, _add_root_url, client_utils.is_file_obj_with_url)
```

**拼接效果示例**（挂载在 `/demo`，反向代理根 `https://ex.com/myapp`）：

| 内部 URL（config 中的初始值） | 拼接后的公开 URL |
|------------------------------|-----------------|
| `/file=/home/user/img.png` | `https://ex.com/myapp/file=/home/user/img.png` |
| `/gradio_api/proxy=https://xxx.hf.space/...` | `https://ex.com/myapp/gradio_api/proxy=https://xxx.hf.space/...` |
| `https://cdn.example.com/external.png` | 不变（已是完整 URL，跳过） |

---

### 8.3 `/config` 路由的前缀注入

**位置**：[routes.py:954-975](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L954-L975)

与主页面流程完全一致：

```python
@app.get("/config", ...)
def get_config(request, deep_link):
    config = utils.safe_deepcopy(app.get_blocks().config)
    root = route_utils.get_root_url(
        request=request,
        route_path="/config",
        root_path=app.root_path
                  or request.scope.get("root_path")
                  or blocks.custom_mount_path,
    )
    config["username"] = get_current_user(request)
    config = route_utils.update_root_in_config(config, root)  # ★ 同样的注入流程
    return ORJSONResponse(content=config)
```

前端 SPA 模式下切换页面时，会请求 `/config` 获取新的配置，走相同的前缀计算。

---

### 8.4 API 信息与代码片段的前缀注入

**位置**：[routes.py:715-747](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L715-L747)

`/gradio_api/info` 路由同样计算 `root`，并将其注入到生成的代码片段中：

```python
root = route_utils.get_root_url(
    request=request,
    route_path=f"{API_PREFIX}/info",
    root_path=app.root_path,
)
# ...
for ep_name, ep_info in api_info.get("named_endpoints", {}).items():
    ep_info["code_snippets"] = generate_code_snippets(
        ep_name, ep_info, str(root),   # ★ root 作为 base_url 传入
        ...
    )
```

这保证了用户点击「查看 API」时看到的 `curl` / Python 示例代码中的 URL
会正确带上反代或挂载路径前缀。

---

### 8.5 队列 SSE 回调中的前缀注入

文件在后台处理后通过队列返回给前端时，也要确保 URL 正确。
**位置**：[queueing.py:312-315](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/queueing.py#L312-L315)、[queueing.py:402-413](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/queueing.py#L402-L413)、[queueing.py:841-844](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/queueing.py#L841-L844)

```python
root_path = route_utils.get_root_url(
    request=request,
    route_path=API_PREFIX + "/queue/join",
    root_path=self.blocks.app.root_path,
)
# ... 然后用该 root_path 构造 SSE 返回的数据体
predict_body, _output = route_utils.call_process_api(
    ...,
    root_path=root_path,
    ...
)
```

`call_process_api` 内部同样会调用 `add_root_url` 把输出文件的 URL 拼上前缀。

---

### 8.6 前缀注入链路总览

```
浏览器请求
    │
    ├─ GET / (主页)
    │   └─ get_root_url(app.root_path or scope.root_path or custom_mount_path) → root
    │       └─ update_root_in_config()
    │           ├─ config["root"] = root
    │           └─ add_root_url() → 所有文件 URL 拼接前缀
    │               → HTML 模板 → 浏览器
    │
    ├─ GET /config
    │   └─ 同上流程 → JSON
    │
    ├─ GET /gradio_api/info
    │   └─ get_root_url(app.root_path 仅此，无兜底) → root
    │       └─ generate_code_snippets(root) → 代码片段中的 URL
    │           ⚠️ 若未设 root_path 且 custom_mount_path 有值，此处 root 可能缺前缀
    │
    ├─ POST /gradio_api/queue/join
    │   └─ 队列系统:
    │       └─ get_root_url(app.root_path) → root_path
    │           └─ call_process_api(root_path)
    │               └─ add_root_url() → 输出文件 URL 拼接前缀
    │                   → SSE 返回给浏览器
    │
    └─ 所有 /gradio_api/file=... / /file=... 请求
        → 浏览器基于 config.root 拼出完整 URL 发起请求
           后端收到后路径本身已带 /gradio_api/file=... 前缀，无需再加
```

> **核心设计洞察**：Gradio 采取「**后端算 root、后端拼 URL**」的策略，
> 所有对外暴露的 URL（文件、代理、API 示例代码）都在后端一次性拼接完成。
> `config["root"]` 是前后端的契约字段，前端基于它构造 API 请求的 base URL。
>
> 但这一策略存在**不一致**：`/` 和 `/config` 用 `or` 链兜底 `custom_mount_path`，
> 而 `/gradio_api/info` 和队列系统只用 `app.root_path`。
> 当 `mount_gradio_app(path="/demo")` 但未设 `root_path` 时，
> 页面和 config 中的 URL 正确，但 API 文档页的示例 URL 会缺少前缀。

---

## 九、请求分流架构：主服务和资源进程各接哪些请求

### 9.1 三种部署拓扑

Gradio 根据是否启用 `ssr_mode` 和 `num_workers`，存在三种请求分流拓扑。

---

#### 拓扑 1：单进程默认模式（最常见）
**条件**：`ssr_mode=False`，`num_workers=0`（或未设置）

```
浏览器
   │  所有请求（页面 / API / 静态 / 文件 / 上传）
   ▼
Python FastAPI (App) ── 端口 7860
   ├─ 根路由：GET /  GET /config  GET /static  GET /assets  /favicon.ico
   ├─ API 路由：/gradio_api/*  (run / call / queue/join / upload / file= / proxy=)
   └─ 文件 I/O 全部在主进程处理
```

**各路径归属**：

| 请求路径 | 归属进程 | 处理者 | 代码位置 |
|---------|---------|--------|---------|
| `GET /` `GET /{page}` | Python 主 | Jinja 模板渲染 config | [routes.py:603](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L603-L681) |
| `GET /config` | Python 主 | `get_config()` | [routes.py:954](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L954-L975) |
| `GET /static/*` | Python 主 | `file_response(STATIC_PATH_LIB)` | [routes.py:977](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L977-L979) |
| `GET /assets/*` | Python 主 | `file_response(BUILD_PATH_LIB)` | [routes.py:1046](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L1046-L1048) |
| `GET /favicon.ico` | Python 主 | `favicon()` | [routes.py:1050](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L1050-L1053) |
| `POST /gradio_api/upload` | Python 主 | `upload_fn()` 写文件到磁盘 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py) 上传路由 |
| `GET /gradio_api/file=*` | Python 主 | `file_fetch()` 从磁盘读 | [routes.py:1082](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L1082-L1086) |
| `POST /gradio_api/run/*` | Python 主 | 直接执行函数 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py) |
| `POST /gradio_api/call/*` | Python 主 | 入队 + SSE 推送 | [routes.py:1342](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L1342-L1355) |
| `POST /gradio_api/queue/join` | Python 主 | 队列系统 | queueing.py |

**特点**：简单、所有逻辑单进程内完成。适合大多数场景，但文件 I/O 会阻塞 API 事件循环。

---

#### 拓扑 2：多进程资源分离（Static Workers，无 Node 代理）
**条件**：`num_workers ≥ 1`，`ssr_mode=False`

端口分配逻辑（[blocks.py:3004-3008](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L3004-L3008)）：
```
Python 主端口 7860
Static Worker 1 端口 7861
Static Worker 2 端口 7862
...
```

**⚠️ 关键发现：Worker 进程已启动但无流量入口**

经过代码逐项验证（详见[第十一章 11.1 节](#111-无-node-时-staticworkerpool-是否接收流量)），在无 Node 代理的拓扑 2 下：

- ✅ Worker 进程确实通过 `multiprocessing.Process` + uvicorn 启动并监听端口
- ✅ Python 主服务**没有任何代码**将请求重定向或代理到 Worker 端口
- ✅ `StaticWorkerPool.get_next_url()` 在全库中**零调用点**
- ✅ `App._static_prefixes` 仅声明未使用，`enable_static_workers` 函数**从未实现**

**结论**：拓扑 2 下 StaticWorkerPool 是**已启动但闲置**的进程——正常使用流程中
没有任何流量会到达 Worker 端口。Worker 的实际价值仅在拓扑 3（Node 代理）中体现，
由 Node 侧的 `classifyRoute` 完成分流。

---

#### 拓扑 3：SSR 生产模式（Node 前端代理 + Python API + Static Workers）
**条件**：`ssr_mode=True`，Node 可用，`num_workers` 可 0 或 ≥ 1

**端口分配**（[blocks.py:2862-2906](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2862-L2906)）：

```
用户访问端口 (Node)       7860    ← 浏览器连这里
Python 内部端口           7861    ← Node 转发非静态请求到这里
Static Worker 1           7862    ← Node 轮询分发静态/文件请求
Static Worker 2           7863
...
```

**Node 启动时的上游配置**（[node_server.py:121-128](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/node_server.py#L121-L128)）：

```python
env["GRADIO_PYTHON_PORT"] = str(python_port)          # 7861
env["GRADIO_PYTHON_HOST"] = python_host               # 127.0.0.1
if static_worker_ports:
    env["GRADIO_STATIC_WORKER_PORTS"] = "7862,7863"   # Node 用逗号分隔的端口列表做 RR
```

Node 作为七层反向代理，路由逻辑定义在 [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js)，
由 [proxy_index.js](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js) 中的 `classifyRoute()` 函数调用。

**`classifyRoute` 的三级分流规则**（可直接验证，[proxy_routes.js:83-116](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L83-L116)）：

```
① 匹配 STATIC_ROUTE_PREFIXES？
   前缀列表（[proxy_routes.js:23-37](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L23-L37)）：
     /gradio_api/upload, /gradio_api/upload_progress,
     /gradio_api/file=, /gradio_api/file/,
     /upload, /upload_progress, /file=, /file/,
     /static/, /assets/, /svelte/, /favicon.ico, /custom_component/
   ├─ hasWorkers=true  → route="worker"（轮询分发到 Static Worker）
   └─ hasWorkers=false → route="python"（回退到 Python）

② 匹配 PYTHON_ROUTE_PREFIXES？
   前缀列表（[proxy_routes.js:8-18](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L8-L18)）：
     /gradio_api, /config, /login, /logout,
     /theme.css, /robots.txt, /pwa_icon, /manifest.json, /monitoring
   → route="python"（转发到 Python 内部端口）

③ 其余所有路径
   → route="sveltekit"（Node 自身 SSR 渲染）
```

**⚠️ 重要修正**：STATIC_ROUTE_PREFIXES 的匹配优先于 PYTHON_ROUTE_PREFIXES。
这意味着 `/gradio_api/upload` 和 `/gradio_api/file=` 虽然也以 `/gradio_api` 开头，
但会被 STATIC 规则**先命中**，转发到 Worker 而非 Python。
这与直觉不同——`/gradio_api` 前缀并不保证请求一定去 Python。

**上传亲和性路由**（可直接验证，[proxy_routes.js:41-46](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L41-L46)）：

对于 `/gradio_api/upload` 和 `/upload` 带有 `upload_id` 查询参数的请求，
Node 使用 `hashString(upload_id) % numWorkers` 确定目标 Worker，
保证同一上传的 upload + upload_progress 请求命中同一 Worker。

**实际分流图**：

```
浏览器 → Node (7860)
           │
           ├─ /static/*  /assets/*  /svelte/*  /favicon.ico  /custom_component/*
           │     └─ 有 Workers → 轮询分发 → Static Worker N
           │        无 Workers → 转发 → Python
           │
           ├─ /gradio_api/upload  /upload  (带 upload_id → 亲和性哈希)
           ├─ /gradio_api/upload_progress  /upload_progress
           ├─ /gradio_api/file=*  /file=*
           │     └─ 有 Workers → Static Worker N (轮询或亲和)
           │        无 Workers → Python
           │
           ├─ /gradio_api/*  /config  /login  /logout  /theme.css  ...
           │     └─ 转发 → Python 内部端口 (7861)
           │
           └─ 其余路径（/ /{page} 等）
                 └─ Node SvelteKit SSR 渲染
```

**各路径归属（拓扑 3，有 Workers）**：

| 请求路径 | classifyRoute 结果 | 最终处理者 |
|---------|-------------------|-----------|
| `GET /` `GET /{page}` | sveltekit | **Node SSR 引擎** |
| `GET /config` | python | Python |
| `GET /static/*` | worker | **Static Worker N**（轮询） |
| `GET /assets/*` | worker | **Static Worker N**（轮询） |
| `GET /svelte/*` | worker | **Static Worker N**（轮询） |
| `GET /favicon.ico` | worker | **Static Worker N**（轮询） |
| `GET /custom_component/*` | worker | **Static Worker N**（轮询） |
| `POST /gradio_api/upload` | worker (带亲和性) | **Static Worker N**（upload_id 哈希） |
| `GET /gradio_api/upload_progress` | worker (带亲和性) | **Static Worker N**（upload_id 哈希） |
| `GET /gradio_api/file=*` | worker | **Static Worker N**（轮询） |
| `POST /gradio_api/queue/join` | python | Python |
| `POST /gradio_api/call/*` | python | Python |
| `POST /gradio_api/run/*` | python | Python |
| `GET /gradio_api/info` | python | Python |
| `GET /gradio_api/proxy=*` | python | Python |
| `GET /theme.css` | python | Python |
| `GET /login` `GET /logout` | python | Python |

> ⚠️ **无 Workers 时的回退**（拓扑 3 但 `num_workers=0`）：
> `classifyRoute` 中 STATIC 匹配但 `hasWorkers=false` 时回退到 `route="python"`
> （[proxy_routes.js:92-95](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L92-L95)），
> 即所有 `/static/*`、`/file=`、`/upload` 等请求也由 Python 处理。

**关键代码佐证**：Static Worker 的路由表在 [static_server.py:68-165](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L68-L165)，
完全对应上表中「由 Static Worker 处理」的路径。Python 主服务在拓扑 3 中
已不在用户端口上监听，不需要再处理这些静态/文件路径。

---

### 9.2 Node 代理中的轮询分发机制

Node 代理的实际分发代码在 [proxy_index.js:68-81](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L68-L81)：

```javascript
if (affinityIndex !== undefined) {
    // 亲和性路由：upload_id 哈希到固定 Worker
    targetPort = staticWorkerPorts[affinityIndex];
} else {
    // 普通轮询：递增计数器取模
    targetPort = staticWorkerPorts[workerIndex % staticWorkerPorts.length];
    workerIndex = (workerIndex + 1) % staticWorkerPorts.length;
}
proxy.web(req, res, { target: `http://${pythonHost}:${targetPort}` });
```

**两种分发策略**：

| 策略 | 触发条件 | 算法 | 用途 |
|------|---------|------|------|
| 轮询（Round-Robin） | 普通静态路由（/static、/assets、/file= 等） | `workerIndex++ % N` | 均衡分发 |
| 亲和性哈希 | 上传路由带 `upload_id` 查询参数 | `hashString(upload_id) % N` | 保证 upload + upload_progress 命中同一 Worker |

**⚠️ Python 端的 `StaticWorkerPool.get_next_url()` 不参与实际分发**：
该函数在 [static_server.py:229-233](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L229-L233) 定义，
但全库搜索**零调用点**。它是 Python 端预留的轮询接口，实际流量分发完全由
Node 侧 `proxy_index.js` 完成。详见[第十一章 11.2 节](#112-node-代理如何选择-worker)。

---

### 9.3 三种分流模式对比

| 维度 | 拓扑 1 单进程 | 拓扑 2 Static Workers（无 Node） | 拓扑 3 SSR + Workers |
|------|-------------|--------------------------------|---------------------|
| 监听端口 | Python × 1 | Python × 1 + Workers × N（Worker 未被浏览器访问） | Node × 1 + Python × 1 + Workers × N |
| 页面渲染 | Python Jinja | Python Jinja | **Node SSR** |
| 静态资源 | Python | Python | **Static Workers RR** |
| 文件上传/下载 | Python | Python | **Static Workers RR** |
| 业务 API / 队列 | Python | Python | Python |
| 文件 I/O 阻塞 API | 是 | 是（实际未分流） | **否** |
| 适用场景 | 开发 / 中小流量 | 预留架构，配合前端反代使用 | 生产高流量 |

---

### 9.4 部署模式下的端口分配时序

以拓扑 3（SSR + 2 Workers）为例，`Blocks.launch()` 中的分配时序
（[blocks.py:2879-3030](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2879-L3030)）：

```
① 启动前：
   user_port = 7860 (用户指定或默认)
   python_internal_port = _find_free_port(7861)   = 7861
   static_worker_ports = [7862, 7863]

② http_server.start_server(server_port=7861) → Python 在 7861 开始监听

③ StaticWorkerPool(ports=[7862,7863]).start()
   → multiprocessing.Process × 2 分别在 7862 / 7863 启动 uvicorn
   → 启动阶段健康检查重试（httpx.get /health，最多 50 次/端口），确认每个 Worker 就绪
   → ⚠️ 这里的「重试」是启动阶段的内部验证，与线上流量的轮询分发是不同的概念

④ ★ 最后启动 Node 前端代理（避免 502 窗口期）：
   start_node_server(
       server_port=7860,          # 面向用户的端口
       python_port=7861,          # 非静态请求转到这里
       static_worker_ports=[7862,7863]  # 静态/文件请求 RR 到这里
   )
   → Node 通过 PORT=7860、GRADIO_PYTHON_PORT=7861、
         GRADIO_STATIC_WORKER_PORTS=7862,7863 启动
   → 重试 HEAD / 直到 Node 返回非 5xx
   → Node 在 7860 就绪，对外宣告完成

⑤ local_url 更新为 http://localhost:7860/（Node 端口，不再是 7861）
   local_api_url = http://localhost:7860/gradio_api/
```

> **架构要点**：Node 代理是**最后启动**的。这避免了浏览器到达用户端口
> 但上游 Python/Worker 尚未就绪时的 502 错误。只有当所有内部依赖
> （Python + Static Workers + Node SSR 预热）全部就绪后，才把用户端口
> 对外开放。

---

## 十、关键文件索引

| 文件 | 职责 |
|------|------|
| [server.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/server.py) | Server 类（Server Mode 入口） |
| [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py) | App 类、create_app、mount_gradio_app、所有路由定义、get_root_url 调用处 |
| [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py) | Blocks 类、launch 方法、端口分配时序、StaticWorkerPool 启动、Node 代理启动 |
| [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py) | StaticWorkerPool、StaticServerConfig、单 Worker FastAPI 路由表 |
| [node_server.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/node_server.py) | Node 进程启动、环境变量（PYTHON_PORT / STATIC_WORKER_PORTS）注入、健康检查 |
| [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/route_utils.py) | `get_root_url()` / `get_request_origin()` / `update_root_in_config()` / 文件响应工具 |
| [processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/processing_utils.py) | `add_root_url()` —— 递归给文件对象拼接公开前缀 |
| [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/queueing.py) | 队列回调中同样调用 `get_root_url()` 给输出文件拼前缀 |
| [proxy_index.js](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js) | Node 代理入口，读取环境变量、调用 classifyRoute、执行轮询/亲和性分发 |
| [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js) | Node 代理分流规则定义：STATIC_ROUTE_PREFIXES / PYTHON_ROUTE_PREFIXES / classifyRoute |
| [+page.server.ts](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.server.ts) | SvelteKit SSR 服务端加载，从请求头提取 x-gradio-* 构造 root_url |
| [+page.ts](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.ts) | SvelteKit 客户端加载，用 root_url 调用 Client.connect |
| [client.ts](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/client.ts) | @gradio/client 核心类，使用 config.root 拼接所有 API 请求 URL |
| [init_helpers.ts](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/helpers/init_helpers.ts) | resolve_config：从 /config 接口获取配置，保留后端提供的 root |

---

## 十一、代码验证核对：逐条区分依据与推断

> 本章对前文中的关键结论逐条标注验证状态，区分「代码直接可证」与「需要推断」的结论。

### 11.1 无 Node 时 StaticWorkerPool 是否接收流量

必须区分两个不同层次的问题：**Worker 进程自身是否具备路由能力** vs **Worker 进程是否在生产流量路径上被访问到**。
健康检查通过只能证明前者，不能等同于线上分流。

#### 层次一：Worker 进程的路由能力（进程内部视角）

Worker 进程是一个**功能完整的独立 FastAPI 应用**，拥有自己的路由表：
- `/static/{path}`、`/assets/{path}`、`/svelte/{path}`、`/favicon.ico`
- `/gradio_api/file={path}`、`/file={path}`
- `/gradio_api/upload`、`/upload`
- `/gradio_api/upload_progress`、`/upload_progress`
- `/health`

（[static_server.py:68-165](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L68-L165)）

只要请求到达 Worker 端口，它就能独立处理。这一点可通过直接访问
`http://127.0.0.1:{worker_port}/health` 验证——健康检查正是这样做的
（[static_server.py:220-227](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L220-L227)）。

**但健康检查是启动阶段的内部验证，不是线上流量的入口。** 它证明的是"Worker 已就绪"，
而非"Worker 已接入流量"。

#### 层次二：Worker 进程的流量接入（服务整体视角）

线上流量是否到达 Worker，取决于是否存在一个**流量分发器**将请求路由到 Worker 端口。
这需要两件事同时满足：(1) 分发器存在；(2) 分发器知道 Worker 端口。

| 拓扑 | 分发器是否存在 | Worker 端口是否被分发器感知 | Worker 是否接收线上流量 |
|------|--------------|-------------------------|---------------------|
| 拓扑 1（单进程） | 无 | N/A | ❌ |
| 拓扑 2（Workers 无 Node） | ❌ Python 主服务不做分流 | Worker 端口在 `StaticWorkerPool.ports` 中，仅被 `start()` 内部健康检查读取；无线上流量分发器读取 | ❌ |
| 拓扑 3（Node 代理 + Workers） | ✅ Node `classifyRoute` | ✅ 环境变量 `GRADIO_STATIC_WORKER_PORTS` | ✅ |

**拓扑 2 的详细证据**：

| 证据 | 验证状态 | 依据 |
|------|---------|------|
| Worker 进程启动并监听端口 | ✅ **直接可证** | [static_server.py:207-215](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L207-L215) `multiprocessing.Process(target=_run_static_worker)` |
| 健康检查通过仅证明进程就绪 | ✅ **直接可证** | [static_server.py:220-227](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L220-L227) `httpx.get(f"http://127.0.0.1:{port}/health")` 是启动阶段的内部检查 |
| Python 主服务不将请求代理到 Worker | ✅ **直接可证** | `get_next_url()` 在 [static_server.py:229](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L229-L233) 定义，全库零调用 |
| `_static_prefixes` 仅声明未使用 | ✅ **直接可证** | [routes.py:254-256](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L254-L256) 声明为 `()`，注释说 "Populated by enable_static_workers"，但该函数从未实现 |
| `local_url` 不指向 Worker | ✅ **直接可证** | [blocks.py:2958](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/blocks.py#L2958-L2958) `self.local_url = local_url` 始终为主端口 |
| 手动访问 Worker 端口可正常响应 | ✅ **直接可证** | Worker 是完整 FastAPI 应用，任何 HTTP 客户端访问都能获得正确响应 |

**结论**：Worker 的路由能力和流量接入是两个独立维度。拓扑 2 下 Worker **有能力**处理请求
（路由表完整、健康检查通过），但**没有流量入口**（无分发器指向它）。这不矛盾——
就像一台已启动的 Web 服务器，如果 DNS 和负载均衡器没指向它，它不会收到线上流量，
但直接访问它的 IP 仍然能正常工作。

`enable_static_workers` 和 `_static_prefixes` 的存在痕迹表明，Python 侧曾计划实现
分流机制（让主服务根据路径前缀将请求转发到 Worker），但该功能未完成，最终由
Node 代理侧的 `classifyRoute` 替代实现了同样的分流目标。

#### 术语澄清：「轮询」在本文档中的三种含义

文档中「轮询」一词出现在不同上下文中，含义不同，不可混用：

| 上下文 | 代码位置 | 含义 | 是否涉及线上流量 |
|--------|---------|------|----------------|
| 启动阶段健康检查 | [static_server.py:220-227](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L220-L227) `for _ in range(50): httpx.get(...)` | 重试请求直到 Worker 就绪 | ❌ 仅启动时执行一次 |
| Python 预留分流接口 | [static_server.py:229-233](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L229-L233) `get_next_url()` | Round-Robin 端口选择 | ❌ 零调用点，未参与运行时 |
| Node 线上流量分发 | [proxy_index.js:74-76](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L74-L76) `workerIndex++ % N` | 运行时请求均衡分发 | ✅ 拓扑 3 下实际生效 |

**常见混淆**：将「健康检查重试」或「`get_next_url` 预留接口」误认为线上分流证据。
前者仅证明 Worker 进程已就绪，后者仅证明 Python 侧曾计划实现分流。
只有 Node 侧的 `workerIndex` 递增逻辑才是运行时实际生效的流量分发。

### 11.2 Node 代理如何选择 Worker

| 结论 | 验证状态 | 依据 |
|------|---------|------|
| Node 从环境变量读取 Worker 端口列表 | ✅ **直接可证** | [proxy_index.js:14-18](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L14-L18) 中 `process.env.GRADIO_STATIC_WORKER_PORTS.split(",")` |
| 静态路由使用轮询（Round-Robin）分发 | ✅ **直接可证** | [proxy_index.js:74-76](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L74-L76) 中 `workerIndex % staticWorkerPorts.length` + `workerIndex++` |
| 上传路由使用亲和性哈希分发 | ✅ **直接可证** | [proxy_routes.js:97-108](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L97-L108) 中 `hashString(uploadId) % numWorkers` |
| 无 Workers 时静态路由回退到 Python | ✅ **直接可证** | [proxy_routes.js:92-95](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L92-L95) 中 `if (!hasWorkers) return { route: "python" }` |
| Python 端的 `get_next_url()` 不参与实际分发 | ✅ **直接可证** | 全库搜索 `get_next_url` 仅在 [static_server.py:229](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/static_server.py#L229-L233) 定义，无任何调用 |
| 轮询状态在 Node 进程内存中，重启后重置 | ⚠️ **推断** | `workerIndex` 是模块级变量（[proxy_index.js:20](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L20-L20)），进程重启归零。这是 JS 变量的自然行为，代码未做持久化 |

### 11.3 页面和文件地址如何最终形成

本节追踪从浏览器输入 URL 到前端发出 API 请求的完整链路，区分两种渲染模式。

#### 模式 A：Python 渲染（拓扑 1/2，无 Node）

```
1. 浏览器输入 http://localhost:7860/demo
      │
2. GET /demo → FastAPI app.mount("/demo", gradio_app)
      │ Starlette 自动设置 scope["root_path"]="/demo"
      │
3. 路由函数 main() 被调用（[routes.py:603](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L603-L681)）
      │
   3a. get_root_url() 计算 root：
       root_path = app.root_path("") or scope.root_path("/demo") or custom_mount_path("/demo")
       → root = "http://localhost:7860/demo"
       │
   3b. update_root_in_config(config, root)：
       config["root"] = "http://localhost:7860/demo"
       add_root_url() 递归拼接所有文件 URL：
         "/file=/tmp/img.png" → "http://localhost:7860/demo/file=/tmp/img.png"
       │
   3c. HTML 模板输出，内嵌 gradio_config：
       <script>window.gradio_config = {root: "http://localhost:7860/demo", ...}</script>
      │
4. 浏览器执行 JS，Client.connect("http://localhost:7860/demo")
      │
   4a. resolve_config() 发现 window.gradio_config 存在 → 直接使用
       （[init_helpers.ts:77-108](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/helpers/init_helpers.ts#L77-L108)）
       │
   4b. config.root = "http://localhost:7860/demo"
       api_prefix = config.api_prefix || ""
       │
5. 前端后续请求全部基于 config.root + api_prefix 拼接：
      │
   5a. SSE 流：new URL(`${config.root}${api_prefix}/queue/join?session_hash=xxx`)
       → http://localhost:7860/demo/gradio_api/queue/join?session_hash=xxx
       （[stream.ts:28](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/utils/stream.ts#L28-L28)）
       │
   5b. 提交请求：`${config.root}${api_prefix}/call/predict`
       → http://localhost:7860/demo/gradio_api/call/predict
       （[submit.ts:199](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/utils/submit.ts#L199-L199)）
       │
   5c. 文件上传：config.root + api_prefix + "/upload?upload_id=xxx"
       → http://localhost:7860/demo/gradio_api/upload?upload_id=xxx
       （[client.ts:448-451](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/client.ts#L448-L451)）
       │
   5d. 取消/重置：`${config.root}${api_prefix}/cancel` / `/reset`
       （[submit.ts:122,129](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/utils/submit.ts#L122-L129)）
       │
6. 浏览器发出的所有请求都经过同一个 http://localhost:7860/demo 前缀
   → Python FastAPI 正常路由匹配
```

**验证标注**：

| 步骤 | 结论 | 验证状态 |
|------|------|---------|
| `scope["root_path"]` 由 FastAPI mount 自动设置 | Starlette ASGI 规范行为 | ✅ 直接可证 |
| `config["root"]` 被写入 HTML 模板的 `window.gradio_config` | 模板渲染机制 | ✅ 直接可证 |
| 前端 `Client.connect` 优先使用 `window.gradio_config` | [init_helpers.ts:77-108](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/helpers/init_helpers.ts#L77-L108) | ✅ 直接可证 |
| 前端所有请求基于 `config.root + api_prefix` 拼接 | stream.ts / submit.ts / client.ts 中多处使用 | ✅ 直接可证 |
| 文件 URL 已在后端通过 `add_root_url` 拼好了前缀 | [processing_utils.py:626](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/processing_utils.py#L626-L635) | ✅ 直接可证 |

#### 模式 B：Node SSR 渲染（拓扑 3）

> ⚠️ 本节已根据代码实际行为重新梳理。核心修正：Python /config 返回的 `config.root`
> **不是**内部地址，而是由 `x-gradio-server` 请求头携带的公开地址。公开地址的传递
> 链路横跨 Node → SvelteKit → Client → Python，形成闭环。

**场景假设**：浏览器通过 `https://example.com` 访问，Node 前面有 nginx 设置了
`X-Forwarded-Proto: https` 和 `X-Forwarded-Host: example.com`。

```
1. 浏览器输入 https://example.com
      │
2. Node 代理接收请求，classifyRoute("/") → "sveltekit"
      │
   2a. Node 注入请求头给 SvelteKit（[proxy_index.js:94-102](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L94-L102)）：
       x-gradio-server      = "http://127.0.0.1:7861"  ← Python 内部地址，供 SvelteKit 内部 fetch 使用
       x-gradio-port         = "7861"
       x-gradio-mounted-path = "/"
       x-gradio-original-url = "https://example.com"    ← 从 x-forwarded-proto + x-forwarded-host 合成
       │
   ⚠️ 注意区分两组头：
       x-gradio-server 是 Node → SvelteKit 的内部通信头，值是 Python 内部地址
       x-gradio-original-url 是 Node → SvelteKit 的公开地址头，值是浏览器看到的地址
      │
3. +page.server.ts 执行（[+page.server.ts:3-53](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.server.ts#L3-L53)）
      │
   3a. server    = x-gradio-server      → "http://127.0.0.1:7861"  ← 内部地址
   3b. mount_path = x-gradio-mounted-path → "/"
   3c. real_url  = new URL(x-gradio-original-url).origin → "https://example.com"  ← 公开地址
   3d. root_url  = new URL(mount_path, real_url).href   → "https://example.com"  ← 公开地址
       （去掉尾部斜杠后）
   3e. 向 Python 内部端口 fetch /config（仅检查 401 状态码，不使用返回的 config 内容）
       此请求不携带 x-gradio-server 头，Python 看到的 origin 是内部地址
       但这无关紧要——仅做认证判断，config 内容不被使用
      │
4. +page.ts 执行（[+page.ts:12-163](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.ts#L12-L163)）
      │
   4a. 构造 api_url 和请求头：
       服务端(!browser)：api_url = server = "http://127.0.0.1:7861"
                         headers.x-gradio-server = root_url = "https://example.com"  ← ★ 关键：公开地址
       浏览器端(browser)：api_url = new URL(mount_path, root_url).href = "https://example.com/"
                          headers.x-gradio-server = new URL(mount_path, location.origin).href
       │
   4b. Client.connect(api_url, { headers })
       服务端：connect("http://127.0.0.1:7861", { headers: { x-gradio-server: "https://example.com" } })
       浏览器端：connect("https://example.com/", { headers: { x-gradio-server: "https://example.com/" } })
      │
5. resolve_config 获取 config（[init_helpers.ts:67-130](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/helpers/init_helpers.ts#L67-L130)）
      │
   5a. 服务端路径（!browser，走 else if 分支）：
       fetch("http://127.0.0.1:7861/config", {
           headers: {
               "Content-Type": "application/json",
               "x-gradio-server": "https://example.com"   ← ★ Client.options.headers 携带
           }
       })
       │
       Python /config 路由执行（[routes.py:956-975](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/routes.py#L956-L975)）：
         get_request_origin():
           x-forwarded-host = 空（Client.fetch 不转发此头）
           x-gradio-server  = "https://example.com"       ← ★ 来自 Client 请求头
           → origin = "https://example.com"
         get_root_url():
           root_path = app.root_path or scope.root_path or custom_mount_path
           → root = "https://example.com"                 ← ★ 公开地址，不是内部地址
         update_root_in_config():
           config["root"] = "https://example.com"
           add_root_url() 递归拼接所有文件 URL 前缀
       → 返回 { root: "https://example.com", components: [...], ... }
       │
       init_helpers.ts 处理：
         config.root = "https://example.com"（后端已提供，非空）
         if (!config.root) fallback 不触发
       → config.root = "https://example.com"  ✅ 正确的公开地址
       │
   5b. 浏览器端路径（browser，走 if 分支）：
       如果 window.gradio_config 存在（SSR 已设置）：
         生产模式(dev_mode=false)：直接使用 window.gradio_config，不再请求 /config
         开发模式(dev_mode=true)：重新 fetch /config，config.root = endpoint
       如果 window.gradio_config 不存在：
         走 else if 分支，fetch "https://example.com/config"
         → 请求经 Node 代理转发到 Python
         → Python 收到的 x-gradio-server 来自 Client.options.headers
         → 同样返回 config.root = "https://example.com"
      │
6. 前端使用 config.root 拼接所有后续请求：
       SSE 流：  https://example.com/gradio_api/queue/join?session_hash=xxx
       API 调用：https://example.com/gradio_api/call/predict
       文件上传：https://example.com/gradio_api/upload?upload_id=xxx
       取消/重置：https://example.com/gradio_api/cancel
       文件下载：https://example.com/gradio_api/file=/tmp/img.png
       │
       所有请求到达 Node 代理 → classifyRoute 分流 → Python 或 Static Worker
```

**公开地址传递的关键链路**：

```
nginx (x-forwarded-proto + x-forwarded-host)
   │
   ▼
Node 代理 (合成 x-gradio-original-url)
   │
   ▼
+page.server.ts (从 x-gradio-original-url 提取 real_url → 计算 root_url)
   │
   ▼
+page.ts (将 root_url 放入 Client.connect 的 headers.x-gradio-server)
   │
   ▼
Client.resolve_config (fetch /config 时携带 x-gradio-server=root_url)
   │
   ▼
Python /config (从 x-gradio-server 读取 origin → get_root_url → config.root)
   │
   ▼
前端 (从 config.root 获取公开地址，拼接所有请求)
```

**⚠️ 之前描述中的错误**：旧版文档步骤 4d 写道
"x-gradio-server 头存在 → origin = http://127.0.0.1:7861"，
这是**不正确的**。混淆了两组不同的 x-gradio-server：

| 上下文 | 谁设置 | 值 | 用途 |
|--------|-------|-----|------|
| Node → SvelteKit 请求 | [proxy_index.js:99](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L99-L99) | `http://127.0.0.1:7861`（内部地址） | SvelteKit 内部 fetch Python 用 |
| Client → Python 请求 | [+page.ts:39](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.ts#L39-L39) / [+page.ts:45-46](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.ts#L45-L46) | `https://example.com`（公开地址） | Python /config 计算 config.root 用 |

Python 的 `get_request_origin` 读到的是 **Client 发来的 x-gradio-server**（公开地址），
不是 Node 注入到 SvelteKit 的那个（内部地址）。两者虽然同名，但在不同的 HTTP 请求中。

**验证标注**：

| 步骤 | 结论 | 验证状态 |
|------|------|---------|
| Node 注入 x-gradio-original-url（公开地址）| [proxy_index.js:94-102](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L94-L102) | ✅ 直接可证 |
| +page.server.ts 从 x-gradio-original-url 构造 root_url | [+page.server.ts:20-22](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.server.ts#L20-L22) | ✅ 直接可证 |
| +page.ts 将 root_url 放入 Client 的 x-gradio-server 头 | [+page.ts:38-47](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/src/routes/[...catchall]/+page.ts#L38-L47) | ✅ 直接可证 |
| Client.fetch 自动附加 options.headers（含 x-gradio-server）| [client.ts:111-125](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/client.ts#L111-L125) | ✅ 直接可证 |
| Python get_request_origin 优先读 x-forwarded-host，其次读 x-gradio-server | [route_utils.py:436-442](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/gradio/route_utils.py#L436-L442) | ✅ 直接可证 |
| Client.fetch 不转发 x-forwarded-host | Client.fetch 仅附加 options.headers 和 cookies | ✅ 直接可证 |
| Python /config 返回的 config.root 是公开地址（来自 x-gradio-server） | get_request_origin → get_root_url → update_root_in_config | ✅ 直接可证 |
| init_helpers.ts 中 `if (!config.root) config.root = endpoint` 仅作为 fallback | [init_helpers.ts:123-125](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/client/js/src/helpers/init_helpers.ts#L123-L125) | ✅ 直接可证 |
| Node 中 x-gradio-mounted-path 硬编码为 "/" | [proxy_index.js:101](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_index.js#L101-L101) | ✅ 直接可证 |
| 子路径挂载场景下 root_url 可能不完整 | ⚠️ **推断** | `x-gradio-mounted-path` 硬编码 `"/"`，如果 Node 前面有 nginx 做子路径反代（如 `location /demo`），`x-gradio-original-url` 不包含 `/demo` 路径段，`root_url` 会缺少子路径前缀 |

### 11.4 关键修正与注意事项

1. **STATIC_ROUTE_PREFIXES 优先于 PYTHON_ROUTE_PREFIXES**：
   这不是直觉设计，而是代码中明确的前后顺序
   （[proxy_routes.js:92](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L92-L92) 先匹配 STATIC，
   [proxy_routes.js:112](file:///d:/fz/0601/solo-dogfeeding/code/242-gradio/js/app/proxy_routes.js#L112-L112) 后匹配 PYTHON）。
   因此 `/gradio_api/upload` 和 `/gradio_api/file=` **不会**去 Python，而是去 Worker。

2. **`custom_mount_path` 的兜底仅限 `/` 和 `/config`**：
   其他路由（`/gradio_api/info`、队列系统）只用 `app.root_path`，不兜底 `custom_mount_path`。
   这意味着挂载子应用但未设 `root_path` 时，API 文档页的示例 URL 可能缺前缀。

3. **`_static_prefixes` 和 `enable_static_workers` 是未完成的功能**：
   代码中声明了字段和注释，但函数从未实现。这是 Python 主服务侧原计划的分流机制，
   最终由 Node 代理侧的 `classifyRoute` 替代完成。

4. **Node 代理的 `x-gradio-mounted-path` 硬编码为 `"/"`**：
   当前代码中 Node 不感知 Gradio 的 `custom_mount_path`，
   子路径挂载场景下 Node 代理如何处理需视具体反代配置而定。
