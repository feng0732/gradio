# Gradio 嵌入 FastAPI 子路径桥接逻辑分析

本文档按代码路径逐步分析 Gradio 应用如何嵌入到 FastAPI 的子路径下。重点分析**静态资源前缀处理的三条差异路径**：
- **路径 A**：后端动态路由（FastAPI mount 负责剥离前缀）
- **路径 B**：前端根地址拼接（config.root 负责拼接所有出站请求 URL）
- **路径 C**：接口返回文件地址加前缀（`add_root_url` 给响应中的文件 URL 补前缀）

---

## 一、挂载入口：从 `mount_gradio_app` 到 FastAPI 的 `app.mount`

### 1.1 入口函数

用户调用的公开 API 是 `gr.mount_gradio_app(app, blocks, path=...)`，其导出链为：

- `gradio/__init__.py` → `from gradio.routes import mount_gradio_app`
- 实际定义在 `gradio/routes.py#L2429-L2596`

### 1.2 内部执行步骤

```
用户调用 mount_gradio_app(app, blocks, path="/gradio", root_path=...)
    │
    ├─ 1. 配置 blocks 对象
    │     ├─ blocks.custom_mount_path = path                 (routes.py#L2517)
    │     ├─ if root_path: blocks.root_path = root_path     (routes.py#L2549-L2550)
    │     └─ blocks.config = blocks.get_config_file()       (routes.py#L2515)
    │
    ├─ 2. 创建 Gradio 内部 FastAPI 子应用
    │     └─ gradio_app = App.create_app(blocks, ...)       (routes.py#L2575-L2580)
    │
    ├─ 3. 合并 lifespan 生命周期
    │     ├─ 包装: 先进入外层 app lifespan，再进入 gradio_app lifespan
    │     └─ app.router.lifespan_context = new_lifespan     (routes.py#L2593)
    │
    └─ 4. FastAPI 原生挂载（关键！这是路径 A 的基础）
          └─ app.mount(path, gradio_app)                    (routes.py#L2595)
```

### 1.3 `App` 类与 `create_app`

`App` 类定义在 `gradio/routes.py#L220`，继承自 `fastapi.FastAPI`。

`create_app` 静态方法定义在 `gradio/routes.py#L358`，核心步骤：

```python
@staticmethod
def create_app(blocks, ...):
    # 1. 创建 App（FastAPI 子类）实例
    app = App(auth_dependency=auth_dependency, ...)         # L376

    # 2. 绑定 blocks 到 app（此处传递 root_path）
    app.configure_app(blocks)                                # L385
    #    └─ self.root_path = blocks.root_path or ""          (routes.py#L278)

    # 3. 创建 API 路由（APIRouter，prefix=API_PREFIX="/gradio_api"）
    router = APIRouter(prefix=API_PREFIX)                    # route_utils.py#L74

    # 4. 注册大量路由（路径 A 的核心：全部挂在子应用根路径上）
    #    ├─ @app.get("/")                       ← 主页面
    #    ├─ @app.get("/static/{path:path}")     ← 静态资源
    #    ├─ @app.get("/assets/{path:path}")     ← 前端构建产物
    #    ├─ @app.get("/svelte/{path:path}")     ← Svelte 组件
    #    ├─ @router.get("/config")              ← /gradio_api/config
    #    ├─ @router.post("/call/{fn_index}")    ← /gradio_api/call/...
    #    ├─ @router.get("/file={path_or_url}")  ← 文件服务
    #    └─ ... (theme.css, favicon, robots.txt 等)

    app.include_router(router)
```

`configure_app` 方法 `gradio/routes.py#L265-L279`：

```python
def configure_app(self, blocks):
    self.blocks = blocks
    self.root_path = blocks.root_path or ""   # L278: 传递 root_path 到 app
    self.state_holder.set_blocks(blocks)
```

---

## 二、静态资源前缀处理的三条路径详解

以下以典型部署场景为例：

```
浏览器实际访问: https://example.com/myapp/gradio/
                    ↑        ↑      ↑
                 协议+主机  代理前缀  FastAPI mount path

部署结构:
  Nginx (反向代理) → 剥离 /myapp，转发 → uvicorn (--root-path=/myapp)
                        FastAPI 主应用
                          ├─ GET  /            → "Hello"
                          └─ mount /gradio → Gradio App（子应用）
```

### 路径 A：后端动态路由（FastAPI mount 自动剥离前缀）

**核心思想：路由注册时不加前缀，由 FastAPI mount 在请求分发时自动剥离前缀。**

#### A.1 路由注册代码位置

所有 Gradio 路由**全部**注册在子应用 `App` 的根路径上，注册时完全不考虑子路径：

| 路由 URL（注册时） | 注册方式 | 代码位置 |
|-------------------|----------|----------|
| `GET /` | `@app.get("/")` | `gradio/routes.py#L603` |
| `GET /{page}` | `@app.get("/{page}")` | `gradio/routes.py#L695` |
| `GET /static/{path:path}` | `@app.get("/static/{path:path}")` | `gradio/routes.py#L977-L979` |
| `GET /assets/{path:path}` | `@app.get("/assets/{path:path}")` | `gradio/routes.py#L1046-L1048` |
| `GET /svelte/{path:path}` | `@app.get("/svelte/{path:path}")` | `gradio/routes.py#L550-L553` |
| `GET /gradio_api/config` | `router = APIRouter(prefix="/gradio_api")` + `@router.get("/config")` | `gradio/routes.py#L383` + `gradio/routes.py#L858` |
| `POST /gradio_api/call/{fn_index}` | `@router.post("/call/{fn_index}")` | `gradio/routes.py#L1128` |
| `GET /gradio_api/file={path}` | `@router.get("/file={path_or_url:path}")` | `gradio/routes.py#L1082-L1084` |
| `GET /theme.css` | `@app.get("/theme.css")` | `gradio/routes.py#L1783-L1786` |
| `GET /favicon.ico` | `@app.get("/favicon.ico")` | `gradio/routes.py#L1050-L1053` |

静态资源常量定义在 `gradio/route_utils.py#L1142-L1157`：

```python
STATIC_TEMPLATE_LIB = "gradio/templates/"            # Jinja2 模板目录
STATIC_PATH_LIB     = "gradio/templates/frontend/static/"  # logo.svg 等
BUILD_PATH_LIB      = "gradio/templates/frontend/assets/"  # JS/CSS 构建产物
```

#### A.2 请求匹配的路径剥离过程

这是 FastAPI/Starlette 的 `mount` 机制自动处理的。用户调用 `app.mount("/gradio", gradio_app)` 后，Starlette 在 `Mount` 中间件中自动做：

```
浏览器请求 URL:  https://example.com/myapp/gradio/assets/app.js
                                              ↓ (ASGI scope)
                scheme: https, host: example.com, path: /myapp/gradio/assets/app.js
                scope["root_path"] = "/myapp"   (由 uvicorn 设置)
                                              ↓
                FastAPI 主应用接收到请求
                匹配到: path 以 "/gradio" 开头
                → 进入 Gradio 子应用
                → scope["root_path"] += "/gradio"  (= "/myapp/gradio")
                → scope["path"]  = "/assets/app.js"   ← 前缀被剥离!
                                              ↓
                Gradio 子应用匹配路由:
                匹配成功: GET /assets/{path:path}
                → path = "app.js"
                → file_response(BUILD_PATH_LIB, "app.js")
```

**路径 A 的特点**：
- **谁来处理前缀**：FastAPI/Starlette 的 `Mount` 中间件（请求入站方向）
- **处理时机**：路由匹配之前（入站请求到达 Gradio 子应用时，前缀已经被剥离）
- **影响范围**：所有后端路由匹配（主页面、静态资源、API、文件服务）
- **开发者是否需要感知**：**不需要**。路由注册时完全写相对根的路径即可

---

### 路径 B：前端根地址拼接（config.root 拼接所有出站请求）

**核心思想：前端不知道自己被挂在子路径下，需要后端把「完整前缀」塞进 config.root，前端靠这个值拼所有出站请求 URL。**

#### B.1 后端注入 `config.root` 的流程

主路由 `/` 处理器 `gradio/routes.py#L603-L692`：

```python
@app.get("/")
@app.get("/{page}")
async def main(request, ...):
    blocks = app.get_blocks()

    # ==== 三重优先级确定 root (gradio/routes.py#L613-L619) ====
    root = route_utils.get_root_url(
        request=request,
        route_path=f"/{page}",
        root_path=(
            app.root_path                    # 优先级1: 用户 mount_gradio_app(root_path=...)
            or request.scope.get("root_path") # 优先级2: ASGI scope.root_path (uvicorn --root-path + mount)
            or blocks.custom_mount_path       # 优先级3: mount_gradio_app(path=...)
        ),
    )

    # 生成配置并更新 root
    config = blocks.get_config_file()
    config = route_utils.update_root_in_config(config, root)  # route_utils.py#L826-L836
    #   └─ 把 config["root"] 设为完整 URL
    #   └─ 同时递归给 config 中所有文件 URL 加前缀 (路径 C 的预演)

    # 注入到模板
    return templates.TemplateResponse("frontend/index.html", {
        "request": request,
        "config": config,  # ← 包含带前缀的 root
        ...
    })
```

#### B.2 `get_root_url` 核心推导逻辑

定义在 `gradio/route_utils.py#L488-L513`：

```python
def get_root_url(request, route_path, root_path):
    # Step 1: 如果 root_path 是完整 URL (http://xxx), 直接返回
    if root_path and client_utils.is_http_url_like(root_path):
        return root_path.rstrip("/")                   # L505-L506

    # Step 2: 从请求还原原始 URL（考虑反向代理头）
    root_url = get_request_origin(request, route_path)  # L508

    # Step 3: 如果 root_path 是相对路径，且不在 URL path 中，拼上去
    if root_path and root_url.path != root_path:
        root_url = root_url.copy_with(path=root_path)   # L510-L511

    return str(root_url).rstrip("/")
```

`get_request_origin` 定义在 `gradio/route_utils.py#L427-L458`：

```python
def get_request_origin(request, route_path):
    # 优先 x-forwarded-host (反向代理场景)
    x_forwarded_host = get_first_header_value(request, "x-forwarded-host")
    if x_forwarded_host:
        root_url = f"http://{x_forwarded_host}"
    else:
        root_url = str(request.url)

    root_url = httpx.URL(root_url).copy_with(query=None)

    # x-forwarded-proto 升级为 https
    if get_first_header_value(request, "x-forwarded-proto") == "https":
        root_url = str(root_url).replace("http://", "https://")

    # 剥离当前 route_path (只有非代理场景才需要)
    route_path = route_path.rstrip("/")
    if len(route_path) > 0 and not x_forwarded_host and str(root_url).endswith(route_path):
        root_url = str(root_url)[: -len(route_path)]

    return httpx.URL(root_url.rstrip("/"))
```

#### B.3 前端如何使用 `config.root`

后端把 config 注入 HTML 后，前端通过 `window.gradio_config` 读取。前端启动链路如下：

```
HTML 模板加载完成
    │
    ├─ window.gradio_config = { ..., root: "https://example.com/myapp/gradio", ... }
    │
    ├─ js/core/src/Blocks.svelte (L34-L90) 接收 props:
    │   {
    │      root, components, layout, dependencies,
    │      api_prefix, ...
    │   }
    │
    ├─ 1) 前端 Client 连接: new AppTree(components, layout, dependencies, config, ...)
    │   │   (init.svelte.ts#L96-L153)
    │   │
    │   └─ get_api_url(config) 计算 API 基础地址:
    │        (init.svelte.ts#L45-L57)
    │        function get_api_url(config) {
    │            const rootUrl = new URL(config.root);
    │            // rootUrl.pathname = "/myapp/gradio"
    │            return rootUrl.origin + rootUrl.pathname + config.api_prefix
    │            // → "https://example.com/myapp/gradio/gradio_api"
    │        }
    │
    ├─ 2) 组件动态加载: get_component(type, class_id, api_url)
    │   │   (init_utils.ts 或 init.svelte.ts#L380-L387)
    │   │
    │   └─ load_component({ api_url, name, id, variant })
    │        (js/build/out/component_loader.js#L8)
    │        // api_url = "https://example.com/myapp/gradio/gradio_api"
    │        // 内置组件直接从 component_map 读（已打包进 bundle，无需请求）
    │        // 自定义组件:
    │        //   GET {api_url}/custom_component/{id}/client/{variant}/index.js
    │        //   → https://example.com/myapp/gradio/gradio_api/custom_component/.../index.js
    │        //   GET {api_url}/custom_component/{id}/client/{variant}/style.css
    │        //   → https://example.com/myapp/gradio/gradio_api/custom_component/.../style.css
    │
    ├─ 3) 静态资源引用（HTML/CSS 中的 src/href）:
    │   │
    │   ├─ 构建产物: <script src="{root}/assets/index-abc123.js">
    │   │   // 注意: 构建产物的 URL 在 HTML 模板中直接引用
    │   │   // 由于 index.html 本身从 {root}/ 返回
    │   │   // 相对路径 "./assets/xxx.js" 实际等价于 "{root}/assets/xxx.js"
    │   │
    │   ├─ theme.css: <link rel="stylesheet" href="{root}/theme.css">
    │   │
    │   └─ favicon: <link rel="icon" href="{root}/favicon.ico">
    │
    └─ 4) Client 发送 API 请求:
         (client/js/src/helpers/init_helpers.ts#L24-L33)
         function resolve_root(base_url, root_path, prioritize_base) {
             // 对 API 请求: prioritize_base = true → 用 base_url
             // 对文件请求: prioritize_base = false → 用 root_path
             if (root_path.startsWith("http")) {
                 return prioritize_base ? base_url : root_path;
             }
             return base_url + root_path;
         }
         // POST {api_url}/call/{fn_index}
         // GET  {api_url}/config
         // POST {api_url}/upload
         // ...
```

**路径 B 的特点**：

| 谁来处理前缀 | `config.root`（后端塞入，前端拼接） |
|---|---|
| 处理时机 | 前端发起出站请求之前 |
| 影响范围 | 前端发起的所有请求：静态资源 `<script>/<link>`、API 请求、自定义组件加载、文件上传 |
| 开发者是否需要感知 | **不需要直接感知**。但部署时必须保证 `get_root_url` 推导正确（通过设置 `root_path` 参数、或正确的 `x-forwarded-host` 头、或 `uvicorn --root-path`）|

---

### 路径 C：接口返回文件地址加前缀（`add_root_url`）

**核心思想：后端 API 返回的 JSON 中，文件 URL 最初是带 `/gradio_api/file=` 前缀的相对路径，必须在出站前给它加上 root 前缀，前端才能正确请求。**

#### C.0 文件 URL 的生成：`move_files_to_cache`

文件 URL 并非凭空出现，而是在 `move_files_to_cache` 中生成的。这个函数在后端 postprocess 之后、`add_root_url` 之前运行，负责把本地文件路径转为可访问的 URL。

**同步版本** `gradio/processing_utils.py#L431-L502`，**异步版本** `gradio/processing_utils.py#L551-L623`，核心逻辑相同：

```python
# processing_utils.py#L481-L492 (同步) / L603-L614 (异步)
url_prefix = (
    f"{API_PREFIX}/stream/" if payload.is_stream else f"{API_PREFIX}/file="
)
# API_PREFIX = "/gradio_api"  (route_utils.py#L74)

if block.proxy_url:
    proxy_url = block.proxy_url.rstrip("/")
    url = f"{API_PREFIX}/proxy={proxy_url}{url_prefix}{payload.path}"
elif client_utils.is_http_url_like(payload.path) or payload.path.startswith(url_prefix):
    url = payload.path          # 已经是 http URL 或已有前缀，原样保留
else:
    url = f"{url_prefix}{payload.path}"   # 拼接: /gradio_api/file= + 实际路径
payload.url = url
```

关键点：**文件 URL 在生成时就自带 `/gradio_api/file=` 前缀**，与路由注册位置完全对应。

三种 URL 前缀格式：

| 场景 | 生成的 `payload.url` | 对应路由 |
|------|---------------------|----------|
| 普通文件 | `/gradio_api/file=/tmp/gradio/abc.png` | `router.get("/file={path_or_url:path}")` → `/gradio_api/file=...` |
| 流式文件 | `/gradio_api/stream/abc123` | `router.get("/stream/{session_hash}/...")` → `/gradio_api/stream/...` |
| 代理文件 | `/gradio_api/proxy=https://other/gradio_api/file=/...` | `router.get("/proxy=...")` → `/gradio_api/proxy=...` |

对应的路由注册（均在 `router = APIRouter(prefix="/gradio_api")` 下）：

- **主文件路由**：`@router.get("/file={path_or_url:path}")` → 完整路径 `/gradio_api/file={path}` （`gradio/routes.py#L1082-L1084`）
- **废弃文件路由**：`@router.get("/file/{path:path}")` → 完整路径 `/gradio_api/file/{path}` （`gradio/routes.py#L1185-L1187`）
- **流式文件路由**：`@router.get("/stream/{session_hash}/{run}/{component_id}/playlist-file")` （`gradio/routes.py#L1159`）

> ⚠️ **易混淆点**：`/file=` 是路由路径的一部分（非查询参数），注册在 `router`（prefix=`/gradio_api`）下，因此完整匹配路径是 `/gradio_api/file=xxx`，而不是 `/file=xxx`。文档中的路由表「`GET /gradio_api/file={path}`」才是正确写法。

#### C.1 调用链全景图

```
前端 POST https://example.com/myapp/gradio/gradio_api/call/0
    │
    ├─ 后端路由匹配（路径 A 已剥离前缀）
    │   → POST /gradio_api/call/0 路由匹配成功
    │
    ├─ 路由处理器: @router.post("/call/{fn_index}")    (routes.py#L1128)
    │   │
    │   ├─ 1) 确定 root_path （再次推导，和路径 B 相同逻辑）
    │   │     root_path = route_utils.get_root_url(
    │   │         request, request.url.path, app.root_path
    │   │     )                                                    (routes.py#L1208-L1213)
    │   │
    │   ├─ 2) 调用 route_utils.call_process_api(
    │   │         app, body, gr_request, fn, root_path
    │   │     )                                                    (route_utils.py#L362)
    │   │
    │   └─ 3) call_process_api → blocks.process_api(
    │            root_path=root_path                               (route_utils.py#L386-L398)
    │        )
    │
    └─ process_api 内部文件 URL 的两次变换:
        │
        ├─ 第一步: postprocess_data → move_files_to_cache
        │    本地路径 → 带 /gradio_api/file= 前缀的相对 URL
        │    "/tmp/gradio/abc.png" → "/gradio_api/file=/tmp/gradio/abc.png"
        │    (processing_utils.py#L481-L492 或 L603-L614)
        │
        ├─ 第二步: add_root_url (blocks.py#L2258-2350 中四次调用)
        │    相对 URL → 带 root 的绝对 URL
        │    "/gradio_api/file=/tmp/..." → "https://host/myapp/gradio/gradio_api/file=/tmp/..."
        │
        │    ① batch 分支
        │       data = add_root_url(data, root_path, None)        (blocks.py#L2258-L2259)
        │    ② 非 batch 分支
        │       data = add_root_url(data, root_path, None)        (blocks.py#L2302-L2303)
        │    ③ 流式输出的每一片段
        │       output_data = add_root_url(output_data, root_path, None)
        │                                                              (blocks.py#L2132-L2135)
        │    ④ gr.render() 的 render_config
        │       output["render_config"] = add_root_url(...)             (blocks.py#L2347-L2350)
```

其他 API 路由也有类似的 `get_root_url` → `call_process_api` 模式：
- `POST /gradio_api/run/{api_name}` → `routes.py#L1295-L1306`
- `POST /gradio_api/queue/join` → `routes.py#L1242-L1249`

#### C.2 `update_root_in_config`（初始化配置时的路径 C 应用）

在主页面加载时，后端也会对初始 config 做相同的文件 URL 加前缀：

`gradio/route_utils.py#L826-L836`:

```python
def update_root_in_config(config, root):
    previous_root = config.get("root")
    if previous_root is None or previous_root != root:
        config["root"] = root
        # 递归给 config 中所有组件的默认文件值加前缀
        config = processing_utils.add_root_url(config, root, previous_root)
    return config
```

#### C.3 `add_root_url` 的核心实现

定义在 `gradio/processing_utils.py#L626-L635`：

```python
def add_root_url(data: dict | list, root_url: str, previous_root_url: str | None):
    def _add_root_url(file_dict: dict):
        # 1) 如果有旧前缀，先剥离（处理 root 变更场景）
        if previous_root_url and file_dict["url"].startswith(previous_root_url):
            file_dict["url"] = file_dict["url"][len(previous_root_url):]

        # 2) 如果已经是 http(s) 绝对 URL（如外部 CDN），跳过
        elif client_utils.is_http_url_like(file_dict["url"]):
            return file_dict

        # 3) 加前缀
        file_dict["url"] = f"{root_url}{file_dict['url']}"
        return file_dict

    # 递归遍历 data 中所有含 "url" 字段的文件对象
    return client_utils.traverse(data, _add_root_url, client_utils.is_file_obj_with_url)
```

#### C.4 文件 URL 的三阶段变换

以一个图片文件为例，追踪 URL 从产生到前端使用的完整变化：

```
阶段 1: 用户函数返回本地路径
  value = "/tmp/gradio/abc123/image.png"

阶段 2: postprocess_data → move_files_to_cache (processing_utils.py#L481-L492)
  url_prefix = f"{API_PREFIX}/file="  →  "/gradio_api/file="
  payload.url = "/gradio_api/file=/tmp/gradio/abc123/image.png"
  ↑ 此时是相对 URL，以 /gradio_api 开头

阶段 3: add_root_url (processing_utils.py#L626-L635)
  root_url = "https://example.com/myapp/gradio"
  file_dict["url"] = "https://example.com/myapp/gradio" + "/gradio_api/file=/tmp/gradio/abc123/image.png"
                   = "https://example.com/myapp/gradio/gradio_api/file=/tmp/gradio/abc123/image.png"
  ↑ 此时是绝对 URL

阶段 4: 前端拿到 URL → <img src="https://example.com/myapp/gradio/gradio_api/file=/tmp/...">
  浏览器发起 GET 请求 → 路径 A 剥离 /myapp/gradio → 子应用看到 /gradio_api/file=/tmp/...
  → 匹配 router.get("/file={path_or_url:path}") → 返回文件二进制 ✅
```

**路径 C 的特点**：

| 谁来处理前缀 | `processing_utils.add_root_url`（后端在 JSON 序列化前修改） |
|---|---|
| 处理时机 | 后端 API 响应出站之前（postprocess + move_files_to_cache 后、return 前） |
| 影响范围 | 组件的 `value` 中文件对象（图片、视频、文件等），初始 config 中文件默认值，render_config 中文件 |
| 开发者是否需要感知 | **不需要**。只要调用 `process_api` 时传入 `root_path`，自动处理 |
| 核心约定 | `move_files_to_cache` 生成 `/gradio_api/file=xxx` 相对 URL → `add_root_url` 加 root 变为 `https://host/.../gradio_api/file=xxx` 绝对 URL |

---

## 三、三条路径的差异对比

### 3.1 核心差异对照表

| 维度 | 路径 A<br>后端动态路由 | 路径 B<br>前端根地址拼接 | 路径 C<br>接口返回文件加前缀 |
|------|----------------------|------------------------|---------------------------|
| **方向** | 入站请求（浏览器→后端） | 出站请求（前端→后端） | 出站响应（后端→前端） |
| **前缀处理者** | FastAPI `Mount` 中间件 | 前端 JS 读取 `config.root` 拼接 | `add_root_url` 后端递归遍历 |
| **处理阶段** | 请求到达 Gradio 子应用 **之前** | 前端发起 fetch/axios **之前** | 后端 JSON 序列化 **之前** |
| **影响的 URL 类型** | 所有后端路由匹配（/、/assets/*、/gradio_api/*、/gradio_api/file=*） | 静态资源 `<script>/<link>`、API 请求、自定义组件加载 | 响应 JSON 中的文件对象 `.url` 字段 |
| **开发者配置点** | `mount_gradio_app(path="/gradio")` | `root_path` 参数 / `x-forwarded-host` / `uvicorn --root-path` | 无需配置（依赖路径 B 推导出的 root_path） |
| **不配置的后果** | 路由根本匹配不到，返回 404 | 前端请求少了前缀，发到主应用上，404 | 组件显示不出图片，因为 URL 是相对路径 |
| **代码分布** | `gradio/routes.py` 路由注册 | `gradio/routes.py` 模板注入<br>`js/core/src/init.svelte.ts`<br>`client/js/src/helpers/init_helpers.ts` | `gradio/blocks.py`（process_api 内四处）<br>`gradio/processing_utils.py` |

### 3.2 一次完整请求中的三条路径流转

以「用户点击按钮生成图片」场景为例，看三条路径如何协作：

```
┌───────────────────────────────────────────────────────────────────────┐
│  阶段 1: 加载主页面                                                    │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  浏览器 GET https://example.com/myapp/gradio/  ← 完整 URL             │
│                           ↓                                           │
│  ╔══ 路径 A (入站剥离) ═══════════════════════════════════════╗        │
│  ║ FastAPI Mount:                                            ║        │
│  ║   scope.root_path = "/myapp" + "/gradio" = "/myapp/gradio"║        │
│  ║   scope.path      = "/"  (前缀已剥离)                     ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  Gradio 路由匹配 GET / → 进入 main() 处理器                           │
│                           ↓                                           │
│  ╔══ 路径 B 推导 (config.root 注入) ══════════════════════════╗        │
│  ║ get_root_url() → "https://example.com/myapp/gradio"        ║        │
│  ║ update_root_in_config(config, root) →                      ║        │
│  ║   1) config["root"] = 完整 URL                              ║        │
│  ║   2) (路径 C 的预演) 初始组件值中的文件 URL 加前缀           ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  返回 HTML + window.gradio_config = { root: "https://..." }           │
│                                                                       │
├───────────────────────────────────────────────────────────────────────┤
│  阶段 2: 前端发起 API 调用（用户点击按钮）                              │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ╔══ 路径 B (前端拼接出站 URL) ═══════════════════════════════╗        │
│  ║ Client 读取 config.root + api_prefix                       ║        │
│  ║   api_url = "https://example.com/myapp/gradio/gradio_api" ║        │
│  ║ POST {api_url}/call/0 → (带完整前缀的 URL)                 ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  浏览器 POST https://example.com/myapp/gradio/gradio_api/call/0       │
│                           ↓                                           │
│  ╔══ 路径 A (入站剥离) ═══════════════════════════════════════╗        │
│  ║ FastAPI Mount:                                            ║        │
│  ║   scope.root_path = "/myapp/gradio"                       ║        │
│  ║   scope.path      = "/gradio_api/call/0"                  ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  Gradio 路由匹配 POST /gradio_api/call/0 → 进入处理器                │
│                           ↓                                           │
│  ╔══ 路径 B 推导 (本次请求的 root_path) ══════════════════════╗        │
│  ║ get_root_url() → 再次得到 "https://example.com/myapp/gradio" ║      │
│  ║ 传入 process_api(root_path=...)                             ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  执行用户函数 → 生成图片文件: /tmp/gradio/abc123/generated.png        │
│                                                                       │
├───────────────────────────────────────────────────────────────────────┤
│  阶段 3: 返回响应（包含图片文件 URL）                                   │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  第一步: postprocess_data → move_files_to_cache                       │
│    url_prefix = "/gradio_api/file="                                   │
│    payload.url = "/gradio_api/file=/tmp/gradio/abc123/generated.png" │
│    ↑ 此时是相对 URL，以 /gradio_api 开头                              │
│                           ↓                                           │
│  ╔══ 路径 C (响应文件 URL 加前缀) ═════════════════════════════╗        │
│  ║ add_root_url(data, "https://example.com/myapp/gradio", None) ║      │
│  ║                                                             ║        │
│  ║ 遍历到文件对象:                                             ║        │
│  ║   1) 检查是否已 http → 否 (/gradio_api/file=...)            ║        │
│  ║   2) 拼前缀:                                                ║        │
│  ║      "https://example.com/myapp/gradio"                     ║        │
│  ║      + "/gradio_api/file=/tmp/gradio/abc123/generated.png"  ║        │
│  ║                                                             ║        │
│  ║ 结果:                                                       ║        │
│  ║   data = [{"url":                                          ║        │
│  ║     "https://example.com/myapp/gradio/gradio_api/file=/tmp/ ║        │
│  ║      gradio/abc123/generated.png",                          ║        │
│  ║     ...}]                                                  ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  JSON 序列化返回给前端                                                  │
│                                                                       │
├───────────────────────────────────────────────────────────────────────┤
│  阶段 4: 前端展示图片（再次发起文件请求）                                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  前端 Image 组件拿到 url →                                             │
│  <img src="https://example.com/myapp/gradio/gradio_api/file=/tmp/     │
│            gradio/abc123/generated.png">                              │
│                           ↓                                           │
│  浏览器 GET https://example.com/myapp/gradio/gradio_api/file=/tmp/    │
│              gradio/abc123/generated.png                              │
│                           ↓                                           │
│  ╔══ 路径 A (入站剥离) ═══════════════════════════════════════╗        │
│  ║ FastAPI Mount:                                            ║        │
│  ║   scope.root_path = "/myapp/gradio"                       ║        │
│  ║   scope.path = "/gradio_api/file=/tmp/gradio/abc123/      ║        │
│  ║                generated.png                               ║        │
│  ╚════════════════════════════════════════════════════════════╝        │
│                           ↓                                           │
│  Gradio 路由匹配:                                                     │
│    router(prefix="/gradio_api") + get("/file={path_or_url:path}")    │
│    → 匹配 /gradio_api/file=/tmp/... → file_fetch() 返回图片二进制 ✅  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 四、常见部署场景的 root_path 推导

### 场景 1：无反向代理，直接 mount

```python
# app.py
app = FastAPI()
demo = gr.Interface(lambda x: x, "text", "text")
gr.mount_gradio_app(app, demo, path="/gradio")
```

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

```
浏览器访问: http://localhost:8000/gradio

推导过程:
  app.root_path = "" (未显式设置)
  request.scope.root_path = "" (uvicorn 没设 --root-path)
  custom_mount_path = "/gradio"
  → root_path 参数 = "/gradio"
  → get_root_url:
      get_request_origin → "http://localhost:8000/gradio" (剥离 route_path="/")
      root_url.path = "/gradio"，等于 root_path
      → root = "http://localhost:8000/gradio" ✅
```

### 场景 2：Nginx 反向代理，代理设置了前缀但不剥离

```nginx
# nginx:
location /myapp/ {
    proxy_pass http://127.0.0.1:8000/;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

```bash
uvicorn app:app --host 0.0.0.0 --port 8000 --root-path /myapp
```

```
浏览器访问: https://example.com/myapp/gradio

推导过程:
  1) Nginx 剥离了 /myapp 前缀后转发到 http://127.0.0.1:8000/gradio
  2) uvicorn --root-path /myapp → scope.root_path = "/myapp"
  3) FastAPI mount /gradio → scope.root_path = "/myapp/gradio", scope.path = "/"
  4) main() 处理器:
     app.root_path = ""
     request.scope.root_path = "/myapp/gradio"
     → root_path 参数 = "/myapp/gradio"
  5) get_root_url:
     x-forwarded-host 存在 → "https://example.com"
     root_url = "https://example.com"
     root_path = "/myapp/gradio"，不等于 path "/"
     → root_url.copy_with(path="/myapp/gradio")
     → root = "https://example.com/myapp/gradio" ✅
```

### 场景 3：多层 Starlette Mount + 显式 root_path

```python
# demo/custom_path/run.py
CUSTOM_PATH = "/gradio"
PROXY_PREFIX = "/myapp"

app = FastAPI()
gr.mount_gradio_app(app, demo,
    path=CUSTOM_PATH,
    root_path=f"{PROXY_PREFIX}{CUSTOM_PATH}"  # 显式设置为 "/myapp/gradio"
)
# 外层再包一层 Starlette Mount:
outer_app = Starlette(routes=[Mount(PROXY_PREFIX, app=app)])
```

```
推导过程:
  main() 处理器:
    app.root_path = "/myapp/gradio"   ← 优先级最高，直接使用
    is_http_url_like → 否
    get_request_origin → "http://localhost:8000/myapp/gradio"
    root_url.path = "/myapp/gradio" == root_path
    → root = "http://localhost:8000/myapp/gradio" ✅
```

---

## 五、关键代码位置索引表（仓库相对路径）

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| `mount_gradio_app` 入口函数 | `gradio/routes.py` | L2429-L2596 |
| 挂载时设置 `custom_mount_path` | `gradio/routes.py` | L2517 |
| 挂载时设置 `blocks.root_path` | `gradio/routes.py` | L2549-L2550 |
| `app.mount(path, gradio_app)` 原生挂载 | `gradio/routes.py` | L2595 |
| `App` 类定义 (FastAPI 子类) | `gradio/routes.py` | L220 |
| `App.configure_app` 传递 root_path | `gradio/routes.py` | L265-L279 |
| `App.create_app` 构建子应用 + 注册路由 | `gradio/routes.py` | L358 |
| 主路由 `/` + root 三层解析 | `gradio/routes.py` | L603-L692, L613-L619 |
| 路由 `GET /static/{path}` | `gradio/routes.py` | L977-L979 |
| 路由 `GET /assets/{path}` | `gradio/routes.py` | L1046-L1048 |
| 路由 `GET /svelte/{path}` | `gradio/routes.py` | L550-L553 |
| 路由 `GET /gradio_api/file={path}` | `gradio/routes.py` | L1082-L1084 |
| 路由 `GET /gradio_api/file/{path}` (废弃) | `gradio/routes.py` | L1185-L1187 |
| 路由 `POST /gradio_api/call/{fn_index}` | `gradio/routes.py` | L1128 |
| call 路由中推导 root_path | `gradio/routes.py` | L1208-L1213 |
| `POST /gradio_api/run/{api_name}` 中推导 | `gradio/routes.py` | L1295-L1306 |
| `call_process_api` 传递 root_path 到 process_api | `gradio/route_utils.py` | L362-L417 |
| `get_root_url` 三层 root 推导 | `gradio/route_utils.py` | L488-L513 |
| `get_request_origin` 还原原始请求地址 | `gradio/route_utils.py` | L427-L458 |
| `update_root_in_config` 初始配置加前缀 | `gradio/route_utils.py` | L826-L836 |
| `API_PREFIX = "/gradio_api"` 常量 | `gradio/route_utils.py` | L74 |
| 静态资源目录常量 | `gradio/route_utils.py` | L1142-L1157 |
| `Blocks.process_api` 四次调用 add_root_url | `gradio/blocks.py` | L2258-L2350 |
| `add_root_url` 文件 URL 加前缀核心实现 | `gradio/processing_utils.py` | L626-L635 |
| `move_files_to_cache` 文件 URL 生成（同步） | `gradio/processing_utils.py` | L431-L502 |
| `async_move_files_to_cache` 文件 URL 生成（异步） | `gradio/processing_utils.py` | L551-L623 |
| `url_prefix = f"{API_PREFIX}/file="` 生成文件 URL 前缀 | `gradio/processing_utils.py` | L481-L482, L603-L604 |
| 前端 `get_api_url` 拼接 API 基础地址 | `js/core/src/init.svelte.ts` | L45-L57 |
| 前端 `resolve_root` 区别 API/文件的前缀策略 | `client/js/src/helpers/init_helpers.ts` | L24-L33 |
| 前端 `load_component` 自定义组件加载 URL 拼接 | `js/build/out/component_loader.js` | L8-L120 |
| 前端 `AppTree` 构造函数（接收 config） | `js/core/src/init.svelte.ts` | L96-L153 |
| 前端 `Blocks.svelte` props 接收 root | `js/core/src/Blocks.svelte` | L34-L90, L161-L177 |
| 静态工作进程路由注册 | `gradio/static_server.py` | L68-L79 |
| `custom_path` 演示 demo | `demo/custom_path/run.py` | 全文 |
| Docker 测试：简单 mount | `test/test_docker/test_reverse_proxy_fastapi_mount/app.py` | 全文 |
| Docker 测试：--root-path 场景 | `test/test_docker/test_reverse_proxy_fastapi_mount_root_path/app.py` | 全文 |
