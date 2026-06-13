# Gradio 流量边界精确分析：Worker 启动条件 → 前端分流 → 登录校验缺口的因果链

本文从代码层面精确串联三者关系：静态 Worker 启动条件、Node 代理前端分流路径、登录校验缺口。核心论点：

> **真正决定风险边界的不是静态 Worker 进程有没有启动，而是请求最终被路由到哪一侧的服务**。

即使静态 Worker 进程在运行，如果请求因路由规则落到主服务，也会正常经过登录校验。反之，即使没有单独访问主服务，如果请求被分流到静态 Worker，也会绕过登录。

---

## 1. 从 launch() 到 Worker 启动的完整决策链

### 1.1 决策树全貌

```
Blocks.launch()
│
├─ 第 1 步：解析 ssr_mode
│   ├─ ssr_mode = _resolve_ssr_mode(ssr_mode)
│   └─ 来源：launch(ssr_mode=...) 或 GRADIO_SSR_MODE 环境变量
│
├─ 第 2 步：解析 num_workers
│   ├─ resolved_num_workers = num_workers
│   └─ 若 None，尝试 GRADIO_NUM_WORKERS 环境变量
│
├─ 第 3 步：ssr_mode 分支
│   ├─ ssr_mode = False → 走纯 Python 架构（无 Node 代理）
│   │   └─ 即使 num_workers >= 1，也不启动 StaticWorkerPool
│   │
│   └─ ssr_mode = True
│       ├─ is_dev_mode = (GRADIO_LOCAL_DEV_MODE 存在)
│       │   ├─ 是 → 开发模式：Python 代理 vite，不启动 Node 前端代理，不启动 StaticWorkerPool
│       │   └─ 否 → 检查 node_path 是否可用
│       │       ├─ node_path 不可用 → 降级为纯 Python，不启动 StaticWorkerPool
│       │       └─ node_path 可用 → 生产 SSR 模式
│       │           ├─ 保留 internal_port 给 Python（用于内部通信）
│       │           ├─ 若 resolved_num_workers >= 1 → 计算 worker_ports
│       │           ├─ 启动 Python 主服务（在 internal_port）
│       │           ├─ 若 resolved_num_workers >= 1 → 启动 StaticWorkerPool 子进程
│       │           └─ 启动 Node 前端代理（用户面对的端口）
│       │               └─ 向 Node 传递环境变量：
│       │                   ├─ GRADIO_PYTHON_PORT = python_internal_port
│       │                   └─ GRADIO_STATIC_WORKER_PORTS = "port1,port2,..."
```

### 1.2 关键代码位置

| 决策点 | 文件 | 行号 |
|--------|------|------|
| ssr_mode 解析 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2828 |
| num_workers 解析（含环境变量） | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2832-L2836 |
| SSR 模式三分支（dev/production/无 node） | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2847-L2915 |
| StaticWorkerPool 实际启动位置 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2982-L3015 |
| Node 代理启动 + 环境变量传递 | [node_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/node_server.py) | L122-L128 |

### 1.3 结论：Worker 启动的四重必要条件

**StaticWorkerPool 进程真正启动，当且仅当：

1. `ssr_mode=True`（或环境变量 `GRADIO_SSR_MODE=True`）
2. `is_dev_mode=False`（没有设置 `GRADIO_LOCAL_DEV_MODE`）
3. 系统有可用的 `node` 可执行文件
4. `resolved_num_workers >= 1`（launch 参数或 `GRADIO_NUM_WORKERS` 环境变量）

**不满足任何一条，都不会有静态 Worker 进程存在。

---

## 2. 前端分流：classifyRoute 的完整决策树

Node 代理在 `js/app/proxy_index.js` 中对每个请求调用 `classifyRoute`，按路径前缀分类到三个目的地之一。

### 2.1 三个目的地定义

```javascript
// 必须去 Python 主服务的路由
const PYTHON_ROUTE_PREFIXES = [
    "/gradio_api",        // 注意：这是一个 catch-all 前缀
    "/config", "/login", "/logout", "/theme.css", ...
];

// 可以分流到静态 Worker 的路由（检查顺序在 PYTHON_ROUTE_PREFIXES 之前！）
const STATIC_ROUTE_PREFIXES = [
    "/gradio_api/upload", "/gradio_api/upload_progress",
    "/gradio_api/file=", "/gradio_api/file/",
    "/upload", "/upload_progress", "/file=", "/file/",
    "/static/", "/assets/", "/svelte/", ...
];
```

- 代码位置：[proxy_routes.js#L8-L37](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js#L8-L37)

### 2.2 classifyRoute 完整决策逻辑

```
classifyRoute(path, { hasWorkers, serverModeEnabled, numWorkers, queryString })
│
├─ if (matchesPrefix(path, STATIC_ROUTE_PREFIXES))
│   ├─ if (!hasWorkers) → { route: "python" }          ← 无 Worker → 回退到 Python
│   └─ else (有 Worker)
│       ├─ if (上传路由 + 有 upload_id)
│       │   └─ { route: "worker", workerIndex: hash(upload_id) % numWorkers }
│       └─ else
│           └─ { route: "worker" }                    ← 轮询分发
│
├─ else if (matchesPrefix(path, PYTHON_ROUTE_PREFIXES) || serverModeEnabled)
│   └─ { route: "python" }
│
└─ else
    └─ { route: "sveltekit" }                          ← SvelteKit SSR 页面
```

- 代码位置：[proxy_routes.js#L83-L116](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js#L83-L116)

### 2.3 关键顺序：STATIC_ROUTE_PREFIXES 优先于 PYTHON_ROUTE_PREFIXES

注意代码顺序！`STATIC_ROUTE_PREFIXES` 检查在 `PYTHON_ROUTE_PREFIXES` 之前。这意味着：

- `/gradio_api/upload` → 匹配 STATIC → 走 Worker
- `/gradio_api/call/...` → 不匹配 STATIC → 匹配 PYTHON → 走主服务

这是有意设计的：`/gradio_api` 是一个宽前缀，其中一部分（上传/文件）需要 offload 到静态 Worker，其余（call/queue 等）必须到主服务。

### 2.4 hasWorkers 的来源

`hasWorkers` 在 Node 代理启动时从环境变量 `GRADIO_STATIC_WORKER_PORTS` 解析：

```javascript
const staticWorkerPorts = process.env.GRADIO_STATIC_WORKER_PORTS
    ? process.env.GRADIO_STATIC_WORKER_PORTS.split(",").map(p => parseInt(p.trim(), 10))
    : [];
const hasWorkers = staticWorkerPorts.length > 0;
```

- 代码位置：[proxy_index.js#L14-L18](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_index.js#L14-L18)

---

## 3. 两条最终链路上的 login_check 实际位置

### 3.1 主服务侧：几乎所有路由挂了 login_check 依赖

主服务在 `App.create_app` 中定义的路由，33 处显式挂了 `dependencies=[Depends(login_check)]`：

| 路由 | 行号 |
|------|------|
| `/config` | [routes.py#L954-L955](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L954-L955) |
| `/file={path}` | [routes.py#L1082-L1083](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1082-L1083) |
| `/file/{path}` | [routes.py#L1185](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1185-L1185) |
| `/upload` | [routes.py#L1738](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1738-L1738) |
| `/call/{api_name}` | [routes.py#L1342-L1343](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1342-L1343) |
| `/queue/join` | [routes.py#L1357](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1357-L1357) |

`login_check` 本身的逻辑：

```python
def login_check(user: str = Depends(get_current_user)):
    if (app.auth is None and app.auth_dependency is None) or user is not None:
        return  # ✅ 通过
    raise HTTPException(401, ...)  # ❌ 未认证
```

- 代码位置：[routes.py#L408-L419](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L408-L419)

`get_current_user` 从 Cookie 中读取 token，查询 `app.tokens` 字典验证。

### 3.2 静态 Worker 侧：完全没有 login_check 依赖

静态 Worker 的 FastAPI 应用由 `create_static_app` 创建，所有相关路由：

```python
# 文件路由：85-90 行
@app.head("/gradio_api/file={path_or_url:path}")
@app.get("/gradio_api/file={path_or_url:path}")
@app.head("/file={path_or_url:path}")
@app.get("/file={path_or_url:path}")
async def file(path_or_url: str, request: fastapi.Request):
    return file_fetch(path_or_url, request, config, upload_dir)

# 上传路由：94-112 行
@app.post("/gradio_api/upload")
@app.post("/upload")
async def upload_file(request: fastapi.Request, upload_id: str | None = None):
    output_files, _, _ = await upload_fn(...)
    return output_files
```

- 代码位置：[static_server.py#L85-L112](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L85-L112)

Grep 验证：整个 `static_server.py` 中 **0 个匹配** `Depends`、`auth`、`login`。

静态 Worker 的 `file_fetch` 只做路径安全检查（blocked/allowed/created 四层），不做登录状态校验。

### 3.3 Node 代理侧：不做任何认证检查

Node 代理（`proxy_index.js`）只是用 `http-proxy` 原样转发 HTTP 请求：

```javascript
proxy.web(req, res, { target: `http://${pythonHost}:${targetPort}` });
```

- 代码位置：[proxy_index.js#L78-L80](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_index.js#L78-L80)

不检查 Cookie、不检查 Token、不做拦截。

---

## 4. 三种部署架构下的分流与认证对比

### 架构一：无 SSR（默认模式）

```
用户 → Python FastAPI (:7860)
    └─ 所有请求 → 主服务 → 所有路由有 login_check
```

| 项目 | 值 |
|------|----|
| Node 代理 | 不存在 |
| 静态 Worker | 不存在 |
| 上传路由位置 | 主服务，有 login_check |
| 文件访问路由位置 | 主服务，有 login_check |
| 认证缺口 | 不存在 |
| 典型场景 | `demo.launch()` 默认 |

### 架构二：SSR 开发模式（`GRADIO_LOCAL_DEV_MODE=1`）

```
用户 → Python FastAPI (:7860) → 代理 vite dev server (:9876)
    └─ 所有 API 请求 → 主服务 → 所有路由有 login_check
```

| 项目 | 值 |
|------|----|
| Node 角色 | vite dev server，Python 是前端 |
| 静态 Worker | 不启动 |
| 上传路由位置 | 主服务，有 login_check |
| 文件访问路由位置 | 主服务，有 login_check |
| 认证缺口 | 不存在 |
| 典型场景 | `gradio dev` 本地开发 |

### 架构三：SSR 生产模式 + num_workers >= 1

```
用户 → Node 代理 (:7860)
    ├─ /upload, /file=, /gradio_api/upload, ... → 静态 Worker (:7862, :7863, ...)
    │   └─ 无 login_check，只路径安全检查
    └─ /gradio_api/call, /config, /queue, ... → Python 主服务 (:7861)
        └─ 有 login_check
```

| 项目 | 值 |
|------|----|
| Node 角色 | 前端代理，用户面对的端口 |
| 静态 Worker | 启动 N 个进程 |
| 上传路由位置 | 静态 Worker，**无 login_check** |
| 文件访问路由位置 | 静态 Worker，**无 login_check** |
| 认证缺口 | 存在（需同时配置了 auth=...） |
| 典型场景 | `demo.launch(ssr_mode=True, num_workers=2)` |

---

## 5. 风险边界精确图

```
                          ┌──────────────────────────────────────────────────────────┐
                          │           风险边界 = 路由目的地              │
                          └──────────────────────────────────────────────────────────┘
                                                     ↓
                    ┌─────────────────────────────────────────┐
                    │  请求路径                           │
                    │  ┌──────────────────────────┐  │
                    │  │ /upload, /file=, ...   │  │
                    │  │ /gradio_api/upload, ...  │  │   → 静态 Worker  → 无 login_check → ❌ 缺口
                    │  └──────────────────────────┘  │
                    │                                   │
                    │  ┌──────────────────────────┐  │
                    │  │ /gradio_api/call, ...  │  │
                    │  │ /config, /queue, ...       │  │   → 主服务        → 有 login_check → ✅ 安全
                    │  └──────────────────────────┘  │
                    └──────────────────────────────────┘
                                                     ↑
                          与 "Worker 进程是否存在" 是必要不充分条件
```

**关键结论**：

1. 静态 Worker 进程存在 ≠ 所有请求都不安全。只有匹配 `STATIC_ROUTE_PREFIXES` 的请求才会走 Worker。
2. 即使 Worker 进程不存在（如 `ssr_mode=False`），也可能因为其他配置错误导致请求走错地方（但这是另一个问题）。
3. 认证缺口的实质是：**同一组路径前缀在两条不同的服务上有不同的认证策略**。主服务侧有 `login_check`，静态 Worker 侧没有。
4. 从攻击者视角：只要目标应用配置了 `auth=...` + `ssr_mode=True` + `num_workers >= 1`，就可以直接 POST `/upload` 和 GET `/file=...` 绕过认证。

---

## 6. 相关代码索引

| 模块 | 文件 | 关键行号 |
|------|------|---------|
| ssr_mode 解析 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2828 |
| num_workers 解析（含环境变量） | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2832-L2836 |
| SSR 三分支决策 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2847-L2915 |
| StaticWorkerPool 启动 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2982-L3015 |
| Node 代理环境变量传递 | [node_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/node_server.py) | L122-L128 |
| STATIC_ROUTE_PREFIXES 定义（优先检查） | [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js) | L23-L37 |
| PYTHON_ROUTE_PREFIXES 定义 | [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js) | L8-L18 |
| classifyRoute 完整决策 | [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js) | L83-L116 |
| Node 代理读取 GRADIO_STATIC_WORKER_PORTS | [proxy_index.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_index.js) | L14-L18 |
| Node 代理原样转发（无认证） | [proxy_index.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_index.js) | L78-L80 |
| 主服务 33 处 login_check 依赖 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L408-L419, L954, L1082-L1083, L1738, ... |
| login_check 实现 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L408-L419 |
| 静态 Worker 路由（无 Depends） | [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py) | L85-L112 |
| 静态 Worker Grep 0 个 auth/login/Depends | [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py) | (全文搜索） |
