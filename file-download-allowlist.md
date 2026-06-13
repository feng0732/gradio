# 文件下载白名单与校验机制

## 1. 路径白名单核心机制

### 1.1 核心校验函数 `is_allowed_file`

**定义位置**: `gradio/utils.py` L1788-L1806

```python
def is_allowed_file(
    path: Path,
    blocked_paths: Sequence[str | Path],
    allowed_paths: Sequence[str | Path],
    created_paths: Sequence[str | Path],
) -> tuple[bool, Literal["in_blocklist", "allowed", "created", "not_created_or_allowed"]]:
```

**校验优先级（从高到低）**:

1. **黑名单检查** (`in_blocklist`): 如果路径在 `blocked_paths` 中，直接拒绝
   - 优先级最高，覆盖所有白名单规则
   - 使用 `is_in_or_equal` 判断，支持目录匹配（子目录/文件均命中）
   - 黑名单比较时路径转小写，不区分大小写

2. **显式白名单** (`allowed`): 如果路径在 `allowed_paths` 中，允许访问
   - 由 `launch(allowed_paths=[...])` 设置
   - 可通过环境变量 `GRADIO_ALLOWED_PATHS` 以逗号分隔设置
   - 下载路由中还额外包含 `_StaticFiles.all_paths`（通过 `set_static_paths()` 注册的静态文件路径）

3. **创建路径白名单** (`created`): 如果路径在 `created_paths` 中，允许访问

4. **默认拒绝** (`not_created_or_allowed`): 不在上述任一列表中的路径均被拒绝

### 1.2 路径包含判断 `is_in_or_equal`

**定义位置**: `gradio/utils.py` L1282-L1295

```python
def is_in_or_equal(path_1: str | Path, path_2: str | Path) -> bool:
```

**核心逻辑**:
- 使用 `abspath(path).resolve()` 规范化路径（解析符号链接、消除 `..`）
- 使用 `path_1.relative_to(path_2)` 判断包含关系
- 支持目录递归包含（子目录/文件均匹配父目录）

---

## 2. 两套路径检查机制对比

Gradio 在两个不同阶段调用 `is_allowed_file`，传入的参数差异极大，对应的防御目标也完全不同。

### 2.1 机制对比总览

| 维度 | `file_fetch` 下载路由 | `_check_allowed` 缓存前检查 |
|------|---------------------|---------------------------|
| **触发时机** | 外部 HTTP 请求 `/file=xxx` 到达时 | 开发者函数返回文件路径后、文件被移动到缓存之前 |
| **所在模块** | `gradio/route_utils.py` L1213-L1220 | `gradio/processing_utils.py` L505-L548 |
| **`allowed_paths`** | `blocks.allowed_paths + _StaticFiles.all_paths` | 后处理：`blocks.allowed_paths + [cwd, tempdir]`<br/>预处理：`[]`（空） |
| **`created_paths`** | `[upload_dir, cache_folder]` | `[upload_dir]` |
| **包含 `cwd`/`tempdir`？** | ❌ 不包含 | ✅ 后处理模式包含 |
| **包含 `cache_folder`？** | ✅ 包含 | ❌ 不包含 |
| **dotfile 特殊检查** | ❌ 无 | ✅ cwd 中的 dotfile 需显式允许 |
| **失败表现** | `HTTPException(403)` — 客户端收到 403 | 抛出 `InvalidPathError` — 开发者在函数调用链中可见 |
| **防御目标** | 阻止攻击者从下载接口读取任意服务器文件 | 约束开发者返回合法来源的文件，然后移动到缓存 |

### 2.2 下载路由 `file_fetch`

**定义位置**: `gradio/route_utils.py` L1190-L1255

```python
allowed, reason = utils.is_allowed_file(
    abs_path,
    blocked_paths=blocks_or_config.blocked_paths,
    allowed_paths=blocks_or_config.allowed_paths + _StaticFiles.all_paths,
    created_paths=[upload_dir, str(utils.get_cache_folder())],
)
if not allowed:
    raise HTTPException(403, f"File not allowed: {path_or_url}.")
```

**关键特性**:
- `cwd` 和 `tempdir` **不在**默认白名单中——外部用户即使知道服务器工作目录下存在某个文件，也无法直接通过 `/file=` 读取
- 所有拒绝一律返回 403，不区分"文件不存在"和"文件不允许"，避免探测文件存在性（测试 `test_do_not_expose_existence_of_files_outside_working_directory` 明确断言 403 而非 404）
- 没有 dotfile 专门检查（但 cwd 本身就不在白名单中，cwd 下的 dotfile 自然也无法访问）

### 2.3 缓存前检查 `_check_allowed`

**定义位置**: `gradio/processing_utils.py` L505-L548

**调用时机**: `async_move_files_to_cache` 内部的 `_move_to_cache` 函数中，开发者函数返回文件路径后、文件被复制到缓存之前。

```python
created_paths = [utils.get_upload_folder()]
if check_in_upload_folder:
    allowed_paths = []
else:
    allowed_paths = blocks.allowed_paths + [os.getcwd(), tempfile.gettempdir()]
allowed, reason = utils.is_allowed_file(
    abs_path,
    blocked_paths=blocks.blocked_paths,
    allowed_paths=allowed_paths,
    created_paths=created_paths,
)
if not allowed:
    raise InvalidPathError(msg)
```

**后处理模式额外检查** (`check_in_upload_folder=False`):

```python
if (
    utils.is_in_or_equal(abs_path, os.getcwd())
    and abs_path.name.startswith(".")
    and not any(
        is_in_or_equal(path, allowed_path) for allowed_path in blocks.allowed_paths
    )
):
    raise InvalidPathError(
        "Dotfiles located in the temporary directory cannot be moved to the cache..."
    )
```

**关键特性**:
- 允许开发者从 `cwd` 和 `tempdir` 返回文件（方便开发和临时文件使用）
- 但 cwd 下的 dotfile（如 `.env`、`.git/config`）必须显式加入 `allowed_paths` 才能返回，防止开发者疏忽泄漏敏感文件
- 通过此检查的文件会被 `async_move_resource_to_block_cache` 复制到 `cache_folder`，**最终客户端下载的是缓存副本**，其路径位于 `file_fetch` 的 `created_paths` 中

### 2.4 开发者返回文件的完整流转

```
开发者函数返回 cwd/data.csv
        ↓
[_check_allowed] cwd 在 allowed_paths → 允许
        ↓
[async_move_resource_to_block_cache] 复制到 cache_folder/hash/data.csv
        ↓
[payload.path 更新] 指向缓存副本
        ↓
[前端访问] /file=cache_folder/hash/data.csv
        ↓
[file_fetch] cache_folder 在 created_paths → 200 允许
```

---

## 3. 安全路径拼接 `safe_join`

**定义位置**:
- `gradio/utils.py` L1767-L1785（`safe_join`）
- `gradio/route_utils.py` L1179-L1180（`routes_safe_join` 包装）

> **重要**：`safe_join` **不用于** `/file=` 下载路由。`/file=` 使用 `abspath()` + `is_allowed_file()` 的包含关系判定。
>
> `safe_join` 仅用于以下已知根目录的静态资源路由：
> - `/static/{path:path}` — 库内置静态资源（根目录: `STATIC_PATH_LIB`）
> - `/assets/{path:path}` — 前端构建产物（根目录: `BUILD_PATH_LIB`）
> - `/favicon.ico` — favicon
> - `/custom_component/...` — 自定义组件资源
> - `deep_links` 状态文件读取

**防护措施**:
- 禁止包含操作系统分隔符（Windows 的 `\`）
- 禁止绝对路径（`/` 开头）
- 禁止 `..` 目录穿越（`filename == ".."` 或 `startswith("../")`）
- 违反任一条件抛出 `InvalidPathError`

**异常处理**:
- `file_response` 函数（`gradio/route_utils.py` L1133-L1134）捕获 `InvalidPathError` 并抛出 `HTTPException(403)`
- 自定义组件路由（`gradio/routes.py` L1015-L1018）单独捕获 `InvalidPathError` 并抛出 `HTTPException(404, detail="Component not found.")`

---

## 4. 下载路由完整流程 `file_fetch`

### 4.1 总体流程图

```
用户请求 /file=path_or_url
       ↓
[路由层] gradio/routes.py L1082-L1086
       ↓
[登录检查] login_check 依赖项
       ↓
[核心处理] file_fetch()  gradio/route_utils.py L1190-L1255
       ├─ 1. HTTP URL → 302 RedirectResponse
       ├─ 2. 非 http(s) 协议前缀 → 403 拒绝
       ├─ 3. abspath 规范化 + 目录/不存在检查 → 403 拒绝（不暴露存在性）
       ├─ 4. is_allowed_file() 白名单校验 → 403 拒绝
       ├─ 5. MIME 判定 + Content-Disposition 策略
       └─ 6. Range 请求? → RangedFileResponse / FileResponse
       ↓
返回文件 / 403
```

### 4.2 路由定义

**定义位置**: `gradio/routes.py` L1082-L1086

```python
@router.head("/file={path_or_url:path}", dependencies=[Depends(login_check)])
@router.get("/file={path_or_url:path}", dependencies=[Depends(login_check)])
async def file(path_or_url: str, request: fastapi.Request):
    blocks = app.get_blocks()
    return file_fetch(path_or_url, request, blocks, app.uploaded_file_dir)
```

### 4.3 MIME 类型与内容处置策略

```python
mime_type, _ = mimetypes.guess_type(abs_path)
if mime_type in XSS_SAFE_MIMETYPES or reason == "allowed":
    media_type = mime_type or "application/octet-stream"
    content_disposition_type = "inline"       # 浏览器内联显示
else:
    media_type = "application/octet-stream"
    content_disposition_type = "attachment"   # 强制下载
```

**XSS 安全 MIME 类型实际列表**（`gradio/route_utils.py` L1159-L1172）:
- `image/jpeg`, `image/png`, `image/gif`, `image/webp`
- `audio/mpeg`, `audio/wav`, `audio/ogg`
- `video/mp4`, `video/webm`, `video/ogg`
- `text/plain`, `application/json`

> `image/svg+xml`、`text/css`、`text/html` 等均不在安全列表内，需依赖 `reason == "allowed"`（即显式加入 `allowed_paths`）才会以 `inline` 方式返回，否则强制 `attachment` 下载。

---

## 5. 签名判断与代理授权边界

### 5.1 Spaces 环境检测

**定义位置**: `gradio/utils.py` L563-L566

```python
def get_space() -> str | None:
    if os.getenv("SYSTEM") == "spaces":
        return os.getenv("SPACE_ID")
    return None
```

### 5.2 `__sign` 查询参数

**出现位置**: `gradio/components/deep_link_button.py` L102

```javascript
currentUrl.searchParams.delete('__sign');  // 仅当 utils.get_space() 为真时执行
```

**边界与职责划分**:
- `__sign` 由 Hugging Face Spaces 的前端基础设施（Node 代理层）注入到 URL 中，用于 Spaces 平台级别的请求鉴权
- **Python 后端代码不进行 `__sign` 的读取、校验或签发**，签名验证完全在 Spaces 基础设施层面完成
- Python 端唯一相关逻辑：在 Spaces 环境下分享 deep link 时，前端 JS 主动从 URL 中移除 `__sign`，避免签名泄漏到分享链接中

### 5.3 代理请求授权 `build_proxy_request`

**定义位置**: `gradio/routes.py` L286-L306

**代理路由**: `/proxy={url_path:path}`（`gradio/routes.py` L1055-L1080）

#### 三层安全边界

| 层级 | 检查逻辑 | 失败结果 | 目的 |
|------|---------|---------|------|
| 1 | `url.host` 必须精确等于 `blocks.proxy_urls` 中某个 URL 的 host | `PermissionError: This URL cannot be proxied.` | 只有通过 `gr.load()` 显式加载的外部 Space 才能被代理 |
| 2 | `url.host` 必须以 `.hf.space` 结尾 | `PermissionError: This URL cannot be proxied.` | 防御 `proxy_urls` 被恶意配置注入内网 IP 或非 HF 域名导致 SSRF |
| 3 | 仅当 `Context.token` 非空时附加 `Authorization: Bearer {token}` 头 | 无 Token 时不附加 | 合法 `.hf.space` 请求自动带上 HF Token，访问私有 Space；非 `.hf.space` 在层 2 已被拦截，不会泄漏 Token |

#### Cookie 隔离（GHSA-2mr9-9r47-px2g 修复）

**定义位置**: `gradio/routes.py` L204-L214

```python
_proxy_transport = httpx.AsyncHTTPTransport(
    limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
)
```

- 所有 `/proxy=` 请求共享一个底层 `AsyncHTTPTransport`（连接池），但**每次请求都创建新的 `httpx.AsyncClient`**
- 不共享 `AsyncClient` 意味着不共享 cookie jar，防止 proxied Space A 返回的 `Set-Cookie` 被自动重放到对 Space B 的请求中
- `reverse_proxy` 处理函数中也明确使用 `XSS_SAFE_MIMETYPES` 过滤响应，非安全 MIME 强制 `attachment` 下载

---

## 6. SSRF 保护下载机制

### 6.1 公共主机名白名单

**定义位置**: `gradio/processing_utils.py` L258-L263

```python
PUBLIC_HOSTNAME_WHITELIST = [
    "hf.co",
    "huggingface.co",
    "*.hf.co",
    "*.huggingface.co",
]
```

### 6.2 安全下载函数

**定义位置**: `gradio/processing_utils.py` L300-L344

#### `async_ssrf_protected_get`

- 调用 `safehttpx.get(..., domain_whitelist=PUBLIC_HOSTNAME_WHITELIST, _transport=async_transport)`
- 手动处理重定向（最多 20 次），每跳都用 `urljoin` 解析相对/协议相对跳转，然后重新走 safehttpx 校验
- 目的：防止 302/307 跳转到内网 IP

#### `async_ssrf_protected_download`

- 以 URL 的 `hash_url(url)` 作为缓存目录名，避免重复下载
- 从 URL path 提取文件名并剔除非法字符
- 如果本地缓存已存在直接返回路径
- 否则调用 `async_ssrf_protected_get` 拉取内容，异步流式写入 `aiofiles`

### 6.3 不安全下载（向后兼容）

**定义位置**: `gradio/processing_utils.py` L347-L368

`unsafe_download` 直接使用 `sync_client.stream("GET", url, follow_redirects=True)`，不做 SSRF 检查。仅用于调用方已确认安全的场景。别名 `save_url_to_cache`（兼容 Gradio < 5.0 的自定义组件）。

---

## 7. 文件移动到缓存流程

**定义位置**: `gradio/processing_utils.py` L551-L623

**调用时机**: postprocess 之后、preprocess 之前

**核心逻辑（`_move_to_cache` 内部函数）**:

```python
# 1. postprocess + 开发者返回 HTTP URL → 直接用 URL，不下载
if payload.url and postprocess and client_utils.is_http_url_like(payload.url):
    payload.path = payload.url

# 2. 已注册为静态文件 → 跳过处理，直接访问
elif utils.is_static_file(payload):
    pass

# 3. 非代理场景的本地文件
elif not block.proxy_url:
    if not client_utils.is_http_url_like(payload.path):
        _check_allowed(payload.path, check_in_upload_folder)  # 路径合法性检查
    if not payload.is_stream:
        temp_file_path = await block.async_move_resource_to_block_cache(payload.path)
        payload.path = temp_file_path

# 4. 构建访问 URL
url_prefix = f"{API_PREFIX}/file="   # 流文件用 /stream/
payload.url = f"{url_prefix}{payload.path}"
```

---

## 8. 边界场景防御矩阵

| 场景 | 触发机制 | 防御逻辑 | 判定结果 | 测试依据 |
|------|---------|---------|---------|---------|
| `/file=../../../etc/passwd` | 下载路由 | `abspath()` 规范化后不在 `allowed_paths` / `created_paths` | 403 拒绝 | `test_do_not_expose_existence_of_files_outside_working_directory` |
| `/file=/etc/passwd` | 下载路由 | 不在 `allowed_paths` / `created_paths` | 403 拒绝 | 同上 |
| `/file=` + cwd 下的任意文件 | 下载路由 | `cwd` **不在** `file_fetch` 的 `allowed_paths` 中 | 403 拒绝 | `test_cannot_access_files_in_working_directory` |
| `/file=subdir/.env`（cwd 下的 dotfile） | 下载路由 | cwd 不在白名单，403（无需走到 dotfile 检查） | 403 拒绝 | `test_cannot_access_files_in_working_directory` L617-L618 |
| `/file=` + 系统临时目录下的文件（未显式允许） | 下载路由 | `tempdir` **不在** `file_fetch` 的 `allowed_paths` 中 | 403 拒绝 | `test_get_allowed_paths` L261-L262 |
| 开发者函数返回 cwd 下的普通文件 | 缓存前检查 | 后处理模式 `allowed_paths` 含 `cwd` → 通过 → 复制到缓存 | 最终可通过缓存路径 200 下载 | — |
| 开发者函数返回 tempdir 下的文件 | 缓存前检查 | 后处理模式 `allowed_paths` 含 `tempdir` → 通过 → 复制到缓存 | 最终可通过缓存路径 200 下载 | — |
| 开发者函数返回 cwd 下的 `.env` | 缓存前检查 | dotfile 检查触发，且不在 `blocks.allowed_paths` | 抛出 `InvalidPathError`（500 级别） | 代码 L538-L548 |
| `/file=` + 已加入 `blocked_paths` 的目录下文件 | 下载路由 | `is_allowed_file` 黑名单优先 | 403 拒绝 | `test_allowed_and_blocked_paths` |
| `/file=` + `allowed_paths` 中的 HTML 文件 | 下载路由 | `reason == "allowed"` → `inline` | 200 内联显示（`text/html`） | `test_response_attachment_format` L318-L320 |
| `/file=` + `upload_dir` 下的 HTML 文件 | 下载路由 | `reason == "created"` → 不在 `XSS_SAFE_MIMETYPES` → `attachment` | 200 强制下载（`application/octet-stream`） | `test_response_attachment_format` L322-L324 |
| `/file=` + `upload_dir` 下的上传文件（非安全 MIME） | 下载路由 | `created_paths` 匹配 → 但不在安全 MIME 列表 | 200 强制下载 | 同上 |
| `/static/..%2findex.html` | 静态资源路由 | `safe_join` 检测 `..` 穿越 → `InvalidPathError` → `file_response` 捕获转 403 | 403 拒绝 | `test_static_files_served_safely` L62-L67 |
| `/custom_component/../../etc/passwd` | 自定义组件路由 | `safe_join` 异常被单独捕获 | 404 Not Found | `routes.py` L1015-L1018 |
| `ssrf_protected_download("http://169.254.169.254/...")` | SSRF 保护下载 | `safehttpx` 内部 IP 拦截 | 抛出异常拒绝下载 | — |
| `ssrf_protected_download("https://huggingface.co/...")` | SSRF 保护下载 | `PUBLIC_HOSTNAME_WHITELIST` 跳过内网 IP 检查 | 正常下载 | — |
| `/proxy=http://169.254.169.254/...` | 代理授权 | 第 2 层：非 `.hf.space` | `PermissionError` | `test_proxy_route_is_restricted_to_load_urls` |
| `/proxy=https://evil-space.hf.space/file=x` | 代理授权 | 第 1 层：不在 `proxy_urls` | `PermissionError` | `test_proxy_route_is_restricted_to_load_urls` |
| `/proxy=https://a.hf.space/` 响应 `Set-Cookie`，再请求 `/proxy=https://b.hf.space/` | 代理授权 | 每次请求新建 `AsyncClient`，不共享 cookie jar | Cookie 不泄漏 | GHSA-2mr9-9r47-px2g |
