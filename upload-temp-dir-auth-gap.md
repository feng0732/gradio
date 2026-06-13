# Gradio 静态 Worker 认证缺口与跨分区上传状态不一致详解

本文深入分析两个问题：
1. 启用静态 Worker 后，上传和文件访问是否仍经过主服务的登录校验
2. 跨分区上传时，前端返回"成功路径"后，后续组件读取的实际行为

---

## 1. 代理分流与路由层级关系

启用 `StaticWorkerPool` 时，整个请求链路分为三层：

```
浏览器
   ↓
Node 代理 (js/app/proxy_index.js)   ← 用户面对的端口，按路径分类路由
   ↓                                 ↓
主服务 FastAPI (Python :7861)    静态 Worker (Python :7862, :7863, ...)
```

路由分类逻辑位于 [proxy_routes.js#L83-L116](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js#L83-L116)，上传与文件访问相关路径全部归类于 `STATIC_ROUTE_PREFIXES`：

```javascript
export const STATIC_ROUTE_PREFIXES = [
    "/gradio_api/upload",
    "/gradio_api/upload_progress",
    "/gradio_api/file=",
    "/gradio_api/file/",
    "/upload",
    "/upload_progress",
    "/file=",
    "/file/",
    ...
];
```

只要 `hasWorkers` 为真（即配置了静态 worker），这些请求就走 `route: "worker"`，由 Node 代理直接转发到某个静态 worker 进程，**不会到达主服务的 FastAPI App**。

代理转发代码位于 [proxy_index.js#L68-L82](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_index.js#L68-L82)：

```javascript
if (route === "worker") {
    // ... affinity hashing / round-robin 选择 worker
    proxy.web(req, res, { target: `http://${pythonHost}:${targetPort}` });
    return;
}
```

这里 `http-proxy` 的 `proxy.web` 只是原样转发 HTTP 请求（含 Cookie、Header），**不做任何认证检查或拦截**。

---

## 2. 认证缺口：静态 Worker 路由完全绕过 login_check

### 2.1 主服务路由：有认证依赖

主服务创建路由时，上传和文件访问都显式加上了 `dependencies=[Depends(login_check)]`：

- 上传路由：[routes.py#L1738](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1738-L1738)
  ```python
  @router.post("/upload", dependencies=[Depends(login_check)])
  ```

- 文件下载路由：[routes.py#L1082-L1084](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1082-L1084)
  ```python
  @router.head("/file={path_or_url:path}", dependencies=[Depends(login_check)])
  @router.get("/file={path_or_url:path}", dependencies=[Depends(login_check)])
  ```

`login_check` 的实现在 [routes.py#L408-L419](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L408-L419)，当应用配置了 `auth` 时会拒绝未登录用户。

### 2.2 静态 Worker 路由：无任何认证依赖

静态 Worker 由 `create_static_app` 创建，代码位于 [static_server.py#L52-L167](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L52-L167)。其上传和文件路由定义如下：

```python
@app.head("/gradio_api/file={path_or_url:path}")
@app.get("/gradio_api/file={path_or_url:path}")
@app.head("/file={path_or_url:path}")
@app.get("/file={path_or_url:path}")
async def file(path_or_url: str, request: fastapi.Request):
    return file_fetch(path_or_url, request, config, upload_dir)   # 注意第 3 个参数是 config，不是 blocks

@app.post("/gradio_api/upload")
@app.post("/upload")
async def upload_file(request: fastapi.Request, upload_id: str | None = None):
    ...
```

对比主服务：
- **没有 `dependencies=[Depends(login_check)]`**
- **函数签名里甚至没有 `Depends()` 参数**
- 整个 `create_static_app` 中不存在 `auth`、`login`、`Depends` 等关键词（Grep 验证：0 个匹配）

`file_fetch` 的第 3 个参数在主服务里是 `blocks` 对象（含 `allowed_paths`/`blocked_paths`），在静态 worker 里传入的是 `StaticServerConfig`（仅含路径列表，不含认证状态）。`file_fetch` 内部只做路径安全检查，不做登录状态校验。

### 2.3 结论：认证缺口

> **只要配置了静态 Worker（`num_workers >= 1`），上传和文件访问就完全绕过主服务的 `login_check`。即使应用配置了 `gr.Interface(auth=...)`，未登录的匿名用户也可以直接：**
> 1. POST 到 `/upload` 上传任意文件
> 2. GET 到 `/file=<path>` 下载任意已知路径的文件

这是一个架构级的认证缺口 — Node 代理把敏感路径分流到了一个没有部署认证中间件的独立服务上。

---

## 3. 跨分区上传：返回路径与实际文件位置不一致

### 3.1 upload_fn 的返回逻辑

`upload_fn` 核心的落盘分支位于 [route_utils.py#L1307-L1318](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1307-L1318)：

```python
temp_file.file.close()
try:
    os.rename(temp_file.file.name, dest)   # 同分区：原子重命名
except OSError:
    if force_move:
        shutil.move(temp_file.file.name, dest)
    else:
        files_to_copy.append(temp_file.file.name)  # 跨分区：记录待复制
        locations.append(dest)
output_files.append(dest)  # ← 无论 rename 是否成功，dest 始终加入返回值
```

关键点：
- `temp_file.file.name` 是 `NamedTemporaryFile` 在**系统临时目录**的实际路径（如 `/tmp/tmpa1b2c3` 或 Windows `%TEMP%\tmpXXXX`）
- `dest` 是计算出的最终目标路径 `GRADIO_TEMP_DIR/<sha256>/filename`
- **`output_files.append(dest)` 在 try/except 之外，无条件执行**

因此 `upload_fn` 的返回值语义是：*"这是文件应该在的位置"*，而不是 *"这是文件当前实际所在的位置"*。

### 3.2 主服务 vs 静态服务的兜底处理

| 处理 | 主服务 `routes.py` | 静态服务 `static_server.py` |
|------|-------------------|---------------------------|
| 接收 `files_to_copy` / `locations` | ✅ 完整接收 | ❌ `output_files, _, _ = await upload_fn(...)` 丢弃 |
| `BackgroundTasks` 异步 `shutil.move` | ✅ `bg_tasks.add_task(move_uploaded_files_to_cache, files_to_copy, locations)` | ❌ 函数签名没有 `bg_tasks` 参数，完全不处理 |

- 主服务兜底函数 `move_uploaded_files_to_cache`：[route_utils.py#L821-L823](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L821-L823)
- 静态服务上传路由：[static_server.py#L94-L112](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L94-L112)

**结论：静态 Worker 在跨分区场景下不兜底。** 返回给前端的 `dest` 路径实际是个空壳，真正的字节永远停留在系统临时目录的随机名文件里。

### 3.3 前端对返回路径的处理

前端 `@gradio/client` 的 upload 流程（[client/js/src/upload.ts#L3-L47](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/client/js/src/upload.ts#L3-L47)）：

```javascript
return response.files.map((f, i) => {
    const file = new FileData({
        ...file_data[i],
        path: f,                                         // 后端返回的 dest
        url: `${root_url}${this.api_prefix}/file=${f}`   // 构造下载 URL
    });
    return file;
});
```

前端直接相信后端返回的 `path`，将其同时作为文件系统路径（`path` 字段）和下载 URL 的组成部分。不会做 HEAD 请求或存在性校验。

---

## 4. 前端返回"成功"后，后续组件读取的完整链路

当用户上传文件并触发事件时，这个 `FileData{path, url, ...}` 会被传回主服务，进入事件处理管线。以下按调用顺序分析每个阶段的行为。

### 4.1 阶段一：check_all_files_in_cache（入口校验）

代码位置：[blocks.py#L1841](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1841-L1841) → [processing_utils.py#L416-L428](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L416-L428)

```python
def _in_cache(d: dict):
    if (
        (path := d.get("path", ""))
        and not client_utils.is_http_url_like(path)
        and not is_in_or_equal(path, get_upload_folder())   # 只检查前缀匹配
        and not utils.is_static_file(path)
    ):
        raise Error(f"File {path} is not in the cache folder...")
```

- **检查内容**：路径是否以 `get_upload_folder()` 为前缀
- **不检查**：文件是否实际存在、是否可读
- **跨分区场景**：`dest` 路径显然在上传目录内，✅ 通过

### 4.2 阶段二：async_move_files_to_cache（preprocess 前处理）

代码位置：[blocks.py#L1849-L1853](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1849-L1853)

```python
inputs_cached = await processing_utils.async_move_files_to_cache(
    value_to_process,
    block,
    check_in_upload_folder=not explicit_call,   # 用户触发事件 → True
)
```

内部先调用 `_check_allowed(payload.path, check_in_upload_folder=True)`：

```python
def _check_allowed(path: str | Path, check_in_upload_folder: bool):
    abs_path = utils.abspath(path)
    created_paths = [utils.get_upload_folder()]
    if check_in_upload_folder:
        allowed_paths = []                              # preprocess 阶段只接受上传目录
    ...
    allowed, reason = utils.is_allowed_file(
        abs_path,
        blocked_paths=blocks.blocked_paths,
        allowed_paths=allowed_paths,
        created_paths=created_paths,                    # 路径从属检查
    )
```

- **检查内容**：路径是否属于 `created_paths`（即上传目录）
- **不检查**：文件是否实际存在
- **跨分区场景**：`dest` 属于上传目录，✅ 通过

### 4.3 阶段三：async_move_resource_to_block_cache（文件落盘到组件缓存）

通过前两关后，进入 [blocks.py#L335-L373](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L335-L373)：

```python
else:
    url_or_file_path = str(utils.abspath(url_or_file_path))
    if not utils.is_in_or_equal(url_or_file_path, self.GRADIO_CACHE):
        # 不在缓存目录 → 需要复制进来
        temp_file_path = processing_utils.save_file_to_cache(...)
    else:
        # 已在缓存目录 → 直接复用，不做任何 I/O
        temp_file_path = url_or_file_path
    self.temp_files.add(temp_file_path)
```

这里的关键分支是 `is_in_or_equal(url_or_file_path, self.GRADIO_CACHE)`：
- **跨分区场景**：`dest` 已在 `GRADIO_CACHE`（上传目录）内，走 else 分支，直接赋值 `temp_file_path = url_or_file_path`
- **不执行** `save_file_to_cache`（也就不会触发 `FileNotFoundError`）
- ✅ "成功"加入组件的 `temp_files` 追踪集合

### 4.4 阶段四：组件 preprocess（真正读取文件）

代码位置：[blocks.py#L1872-L1874](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1872-L1874)

```python
processed_value = await anyio.to_thread.run_sync(
    block.preprocess, inputs_cached, limiter=self.limiter
)
```

不同组件的 preprocess 行为不同，但只要涉及文件读取就会崩溃：

| 组件 | preprocess 行为 | 跨分区时的结果 |
|------|----------------|-------------|
| `gr.File` | 读取文件内容或元数据 | ❌ `FileNotFoundError: [Errno 2] No such file or directory: 'GRADIO_TEMP_DIR/<hash>/filename'` |
| `gr.Image` | PIL.Image.open(path) | ❌ `FileNotFoundError` |
| `gr.Audio` | 读取音频为 (sample_rate, data) | ❌ `FileNotFoundError` |
| `gr.Video` | 读取视频文件 | ❌ `FileNotFoundError` |

异常被包装为 `ComponentProcessingError` 并返回给前端，用户看到的是组件处理错误，而不是"上传未完成"。

### 4.5 旁路：前端直接预览（请求 /file= URL）

如果用户不触发事件，只是前端组件内部尝试预览文件（例如 `<img src="/gradio_api/file=...">`），请求会到达静态 Worker 的 `file_fetch`：

```python
abs_path = utils.abspath(path_or_url)
try:
    if abs_path.is_dir() or not abs_path.exists():
        raise HTTPException(403, f"File not allowed: {path_or_url}.")
except Exception as e:
    raise HTTPException(403, f"File not allowed: {path_or_url}.") from e
```

- 代码位置：[route_utils.py#L1204-L1209](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1204-L1209)

`not abs_path.exists()` 为真，返回 **403 Forbidden**（注意不是 404），错误信息 `"File not allowed: ..."` 具有误导性，让开发者误以为是权限问题而非文件不存在。

---

## 5. 问题全景与影响矩阵

| # | 问题 | 触发条件 | 影响 |
|---|------|---------|------|
| 1 | 上传/下载绕过登录校验 | 启用了 `StaticWorkerPool` 且配置了 `auth` | 未登录用户可匿名上传、下载任意文件 |
| 2 | 跨分区上传文件永久留在系统临时目录 | 系统 tmp 与 `GRADIO_TEMP_DIR` 不在同一分区 + 使用静态 Worker | 磁盘空间随上传线性泄漏，文件名为 `tmpXXXXXX` 无后缀难以辨识 |
| 3 | 前端返回成功但组件读取时报错 | 上述跨分区条件 + 用户触发事件 | 用户看到 `FileNotFoundError` / `ComponentProcessingError`，难以排查根因 |
| 4 | 前端预览返回 403 "File not allowed" | 上述跨分区条件 + 前端直接加载预览 | 错误信息误导为权限问题，排查成本高 |
| 5 | 上传文件未被 temp_file_sets 追踪 | 任何使用静态 Worker 的场景（无论是否跨分区） | 即使配置了 `delete_cache` 也不会清理这些文件，永久占用磁盘 |

---

## 6. 相关代码索引

| 模块 | 文件 | 关键行号 |
|------|------|---------|
| Node 代理分类路由 | [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js) | L83-L116 |
| Node 代理转发 HTTP | [proxy_index.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_index.js) | L56-L104 |
| 主服务 login_check 定义 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L408-L419 |
| 主服务 /upload 带认证依赖 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L1738-L1772 |
| 主服务 /file= 带认证依赖 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L1082-L1087 |
| 静态 Worker /upload 无认证依赖 | [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py) | L94-L112 |
| 静态 Worker /file= 无认证依赖 | [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py) | L85-L90 |
| upload_fn rename 分支与 output_files 返回 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py) | L1307-L1318 |
| file_fetch 文件存在性检查 → 403 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py) | L1204-L1209 |
| check_all_files_in_cache 仅检查前缀 | [processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py) | L416-L428 |
| _check_allowed 仅检查路径从属 | [processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py) | L505-L548 |
| async_move_resource_to_block_cache 已在缓存内直接复用 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L335-L373 |
| 组件 preprocess 实际读取文件 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L1870-L1887 |
| 前端 Client 处理上传响应构造 FileData | [client/js/src/upload.ts](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/client/js/src/upload.ts) | L25-L46 |
