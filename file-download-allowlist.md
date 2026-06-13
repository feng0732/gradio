# 文件下载白名单与校验机制

## 1. 路径白名单核心机制

### 1.1 核心校验函数 `is_allowed_file`

**定义位置**: [utils.py#L1788-L1806](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/utils.py#L1788-L1806)

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
   - 检查使用 `is_in_or_equal` 函数，支持目录匹配

2. **显式白名单** (`allowed`): 如果路径在 `allowed_paths` 中，允许访问
   - 由 `launch(allowed_paths=[...])` 设置
   - 包括通过环境变量 `GRADIO_ALLOWED_PATHS` 设置的路径
   - 包括 `_StaticFiles.all_paths`（静态文件路径）

3. **创建路径白名单** (`created`): 如果路径在 `created_paths` 中，允许访问
   - 包括 `upload_dir`（用户上传目录）
   - 包括 `utils.get_cache_folder()`（应用缓存目录）

4. **默认拒绝** (`not_created_or_allowed`): 不在上述任一列表中的路径均被拒绝

### 1.2 路径包含判断 `is_in_or_equal`

**定义位置**: [utils.py#L1282-L1295](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/utils.py#L1282-L1295)

```python
def is_in_or_equal(path_1: str | Path, path_2: str | Path) -> bool:
```

**核心逻辑**:
- 使用 `abspath(path).resolve()` 规范化路径，解析符号链接
- 使用 `path_1.relative_to(path_2)` 判断包含关系
- 支持目录递归包含（子目录/文件均匹配）
- 黑名单检查时转小写进行不区分大小写匹配

### 1.3 安全路径拼接 `safe_join` / `routes_safe_join`

**定义位置**: 
- [utils.py#L1767-L1785](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/utils.py#L1767-L1785)
- [route_utils.py#L1179-L1180](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/route_utils.py#L1179-L1180)

**防护措施**:
- 禁止包含操作系统分隔符（Windows 的 `\`）
- 禁止绝对路径（`/` 开头）
- 禁止 `..` 目录穿越
- 禁止 `../` 开头的相对路径
- 违反任一条件抛出 `InvalidPathError`

### 1.4 应用层路径检查 `_check_allowed`

**定义位置**: [processing_utils.py#L505-L548](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/processing_utils.py#L505-L548)

**调用时机**: 文件移动到缓存前（`async_move_files_to_cache` 中调用）

**两种检查模式**:

| 模式 | `check_in_upload_folder` | 允许路径 | 场景 |
|------|-------------------------|---------|------|
| **预处理模式** | `True` | 仅 `upload_folder` | 用户上传文件（preprocess 阶段），确保文件是用户上传的 |
| **后处理模式** | `False` | `allowed_paths` + `cwd` + `tempdir` | 应用返回文件（postprocess 阶段） |

**额外安全检查**:
- CWD 中的 dotfile（`.` 开头）默认禁止，除非显式加入 `allowed_paths`

### 1.5 公共主机名白名单（SSRF 防护）

**定义位置**: [processing_utils.py#L258-L263](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/processing_utils.py#L258-L263)

```python
PUBLIC_HOSTNAME_WHITELIST = [
    "hf.co",
    "huggingface.co",
    "*.hf.co",
    "*.huggingface.co",
]
```

**用途**: SSRF 保护下载时，跳过这些域名的内部 IP 检查（因为 Hugging Face 使用 DNS 分割，可能解析到内部 IP）

---

## 2. 签名判断机制

### 2.1 Spaces 环境检测

**定义位置**: [utils.py#L563-L566](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/utils.py#L563-L566)

```python
def get_space() -> str | None:
    if os.getenv("SYSTEM") == "spaces":
        return os.getenv("SPACE_ID")
    return None
```

**签名参数 `__sign`**:
- 存在于 [deep_link_button.py#L102](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/components/deep_link_button.py#L102) 中
- 在 Hugging Face Spaces 环境下，分享链接时会删除 URL 中的 `__sign` 参数
- 签名验证主要在 Spaces 基础设施层面（Node 代理层）处理，Python 代码不直接进行 HMAC 验证

### 2.2 代理请求授权

**定义位置**: [routes.py#L638-L688](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/routes.py#L638-L688)

**代理安全规则**:
- 仅允许代理 `.hf.space` 域名的 URL
- 非 `.hf.space` URL（包括内网 IP、local 域名）即使在 `proxy_urls` 中也会被拒绝
- 对合法的 `.hf.space` 请求，自动附加 `Authorization: Bearer {HF_TOKEN}` 头部
- 外部域名不会泄漏 HF Token

---

## 3. 文件响应完整流程

### 3.1 总体流程图

```
用户请求 /file=path
       ↓
[路由层] routes.py#L1082-L1086
       ↓
[登录检查] login_check 依赖
       ↓
[核心处理] file_fetch() route_utils.py#L1190-L1255
       ├─ 1. HTTP URL 检查 → 302 重定向
       ├─ 2. 协议检查 → 403 拒绝非文件协议
       ├─ 3. 路径规范化 → abspath + 存在性检查
       ├─ 4. 白名单校验 → is_allowed_file()
       ├─ 5. MIME 类型判断 → XSS 安全检测
       └─ 6. 响应构建 → FileResponse / RangedFileResponse
       ↓
返回文件 / 403 拒绝
```

### 3.2 路由定义

**定义位置**: [routes.py#L1082-L1086](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/routes.py#L1082-L1086)

```python
@router.head("/file={path_or_url:path}", dependencies=[Depends(login_check)])
@router.get("/file={path_or_url:path}", dependencies=[Depends(login_check)])
async def file(path_or_url: str, request: fastapi.Request):
    blocks = app.get_blocks()
    return file_fetch(path_or_url, request, blocks, app.uploaded_file_dir)
```

### 3.3 文件获取核心 `file_fetch`

**定义位置**: [route_utils.py#L1190-L1255](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/route_utils.py#L1190-L1255)

#### 步骤 1: URL 类型判断

```python
if client_utils.is_http_url_like(path_or_url):
    return RedirectResponse(url=path_or_url, status_code=302)
if starts_with_protocol(path_or_url):
    raise HTTPException(403, f"File not allowed: {path_or_url}.")
```

#### 步骤 2: 路径规范化与存在性检查

```python
abs_path = utils.abspath(path_or_url)
if abs_path.is_dir() or not abs_path.exists():
    raise HTTPException(403, f"File not allowed: {path_or_url}.")
```

#### 步骤 3: 白名单校验

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

#### 步骤 4: MIME 类型与内容处置策略

```python
mime_type, _ = mimetypes.guess_type(abs_path)
if mime_type in XSS_SAFE_MIMETYPES or reason == "allowed":
    media_type = mime_type or "application/octet-stream"
    content_disposition_type = "inline"  # 浏览器内联显示
else:
    media_type = "application/octet-stream"
    content_disposition_type = "attachment"  # 强制下载
```

**XSS 安全 MIME 类型列表** ([route_utils.py#L1165-L1172](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/route_utils.py#L1165-L1172)):
- `image/*` (png, jpeg, gif, svg+xml, webp, etc.)
- `audio/*` (mpeg, wav, etc.)
- `video/*` (mp4, webm, etc.)
- `text/plain`, `application/json`, `text/css`

#### 步骤 5: 响应构建

- **普通响应**: `FileResponse` 带 `Accept-Ranges: bytes` 头
- **范围请求**: `RangedFileResponse` 支持断点续传
- **反向代理**: 外部 Gradio 应用文件走 `/proxy=` 路径

### 3.4 文件移动到缓存流程 `async_move_files_to_cache`

**定义位置**: [processing_utils.py#L551-L623](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/processing_utils.py#L551-L623)

**调用时机**: postprocess 之后、preprocess 之前

**核心逻辑**:
```python
def _move_to_cache(d: dict):
    payload = FileData(**d)
    
    # 1. HTTP URL 直通（postprocess 阶段）
    if payload.url and postprocess and client_utils.is_http_url_like(payload.url):
        payload.path = payload.url
    
    # 2. 静态文件跳过处理
    elif utils.is_static_file(payload):
        pass
    
    # 3. 非代理文件处理
    elif not block.proxy_url:
        if not client_utils.is_http_url_like(payload.path):
            _check_allowed(payload.path, check_in_upload_folder)  # 路径检查
        if not payload.is_stream:
            temp_file_path = await block.async_move_resource_to_block_cache(
                payload.path
            )
            payload.path = temp_file_path
    
    # 4. 构建访问 URL
    url_prefix = f"{API_PREFIX}/file="
    payload.url = f"{url_prefix}{payload.path}"
```

---

## 4. SSRF 保护下载机制

### 4.1 安全下载函数

**定义位置**: [processing_utils.py#L300-L344](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/processing_utils.py#L300-L344)

#### `async_ssrf_protected_get`

- 使用 `safehttpx` 进行 SSRF 防护请求
- 每一跳重定向都重新验证主机（最多 20 次）
- 支持公共主机名白名单跳过内部 IP 检查

#### `async_ssrf_protected_download`

- 基于 URL 哈希缓存文件，避免重复下载
- 调用 `async_ssrf_protected_get` 获取内容
- 异步流式写入临时文件

### 4.2 不安全下载（向后兼容）

**定义位置**: [processing_utils.py#L347-L368](file:///d:/fz/0601/solo-dogfeeding/code/261-gradio/gradio/processing_utils.py#L347-L368)

`unsafe_download` 直接使用 httpx 下载，不进行 SSRF 检查，仅用于已知安全的场景。

---

## 5. 边界场景与防御矩阵

| 场景 | 防御机制 | 判定结果 |
|------|---------|---------|
| 路径包含 `../` 穿越 | `safe_join` 检查 | 403 拒绝 |
| 请求绝对路径 `/etc/passwd` | `safe_join` 检查 | 403 拒绝 |
| 访问 blocked_paths 中的文件 | `is_allowed_file` 黑名单 | 403 拒绝 |
| 访问 allowed_paths 中的 HTML | reason="allowed" → inline | 200 内联显示 |
| 访问 cwd 中的 HTML（非 allowed） | XSS 检查 → attachment | 200 强制下载 |
| 访问 upload_dir 中的文件 | created_paths 匹配 | 200 允许（强制下载非安全类型） |
| 访问系统临时目录文件 | 默认 allowed_paths 包含 tempdir | 200 允许 |
| 请求内部 IP 地址 URL | `safehttpx` SSRF 检查 | 拒绝 |
| 请求 huggingface.co URL | 公共白名单跳过 IP 检查 | 允许 |
| CWD 中的 dotfile `.env` | dotfile 特殊检查 | 403 拒绝（除非显式允许） |
| 代理非 `.hf.space` 域名 | 代理域名白名单 | PermissionError |
