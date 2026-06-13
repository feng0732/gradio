# Gradio 文件上传、临时目录、缓存与下载校验机制

## 一、核心目录结构

### 1.1 上传目录（GRADIO_TEMP_DIR）
- **定义位置**: [route_utils.py#L1174-L1176](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1174-L1176) 及 [utils.py#L1489-L1492](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1489-L1492)
- **默认值**: 环境变量 `GRADIO_TEMP_DIR` 未设置时，默认为 `<系统临时目录>/gradio`（例如 Linux/macOS 下为 `/tmp/gradio`，Windows 下为 `%TEMP%/gradio`）
- **作用**: 所有用户上传文件的最终落盘位置，同时也是组件级缓存目录（Block.GRADIO_CACHE）

### 1.2 示例缓存目录（GRADIO_EXAMPLES_CACHE）
- **定义位置**: [utils.py#L1434-L1435](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1434-L1435)
- **默认值**: 环境变量 `GRADIO_EXAMPLES_CACHE` 未设置时，默认为 `.gradio/cached_examples`
- **作用**: 存储 `gr.Examples` 的预计算结果，与上传目录互相独立

### 1.3 系统临时目录（tempfile.gettempdir()）
- **用途**: 上传过程中临时存放文件（NamedTemporaryFile），上传完成后会被移动/重命名到上传目录

---

## 二、文件上传流程（从前端到后端落盘）

### 2.1 前端发起上传
- **前端组件**: [js/upload/](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/upload/)
  - 拖拽/点击选择文件：[utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/upload/src/utils.ts) 中的 `create_drag()`
  - MIME 类型校验：`is_valid_mimetype()` 基于 accept 属性检查扩展名和 MIME 类型
- **上传请求**: 以 `multipart/form-data` POST 到 `/gradio_api/upload` 或 `/upload`，可携带 `upload_id` 参数用于追踪进度

### 2.2 后端接收入口
有两种服务模式都实现了上传路由：

**模式 A：主 App 路由**（[routes.py#L1738-L1770](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1738-L1770)）
```
POST /gradio_api/upload  →  upload_fn()  →  返回文件路径列表
```

**模式 B：静态文件服务器进程**（[static_server.py#L94-L112](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L94-L112)）
```
POST /gradio_api/upload  →  upload_fn()  →  返回文件路径列表
POST /upload             →  upload_fn()  →  返回文件路径列表
```

### 2.3 核心上传函数：upload_fn()
**定义位置**: [route_utils.py#L1258-L1320](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1258-L1320)

执行步骤：
1. **解析 multipart**: 使用自定义的 `GradioMultiPartParser`（而非 Starlette 默认的解析器）
2. **逐块流式处理**: 每个文件块被写入 `NamedTemporaryFile(delete=False)`（在系统临时目录中），同时计算 SHA-256 哈希
3. **哈希去重**: 文件内容哈希相同的文件会落到同一个子目录 `<upload_dir>/<sha256_hash>/<filename>`
4. **从临时目录移动到上传目录**:
   - 优先用 `os.rename()`（原子操作，同分区内瞬间完成）
   - 若跨分区失败：
     - `force_move=True` 时，用 `shutil.move()`（复制+删除）
     - `force_move=False` 时，返回 `(files_to_copy, locations)`，调用方通过 BackgroundTasks 异步调用 `move_uploaded_files_to_cache()` 完成移动
5. **返回结果**: `(output_files, files_to_copy, locations)`

### 2.4 自定义 Multipart 解析器：GradioMultiPartParser
**定义位置**: [route_utils.py#L614-L818](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L614-L818)

相比 Starlette 默认实现的关键差异：
- 使用 `NamedTemporaryFile(delete=False)` 而非 `SpooledTemporaryFile`：文件始终落盘，便于后续重命名移动
- 内置 `GradioUploadFile`：持有 `sha` 属性，在流式写入时实时更新内容哈希
- 支持上传进度追踪：`FileUploadProgress` + SSE 端点 `/upload_progress`
- 支持 `max_file_size` 限制：超过直接抛出 `MultiPartException`

### 2.5 上传进度追踪
- **核心类**: `FileUploadProgress`（[route_utils.py#L559-L611](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L559-L611)）
- **SSE 端点**: `/gradio_api/upload_progress`（[routes.py#L1676-L1722](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1676-L1722) 和 [static_server.py#L114-L161](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L114-L161)）
- **机制**: 解析器每接收一个数据块就 `append(upload_id, filename, chunk_size)`，前端通过 SSE 轮询 `pop(upload_id)` 获取进度

---

## 三、文件缓存机制（组件级缓存）

### 3.1 两层 "Cache" 概念
Gradio 中存在两种完全不同的缓存：

| 缓存类型 | 存储介质 | 用途 | 生命周期 |
|---|---|---|---|
| **文件缓存**（Block.GRADIO_CACHE） | 磁盘（上传目录） | 组件输出文件、URL 下载文件、用户上传文件 | 由定时清理器管理 |
| **内存缓存**（gr.cache / gr.Cache） | 内存（OrderedDict LRU） | 函数返回值缓存、KV 手动缓存 | 由 LRU 策略或会话断开清理 |

### 3.2 文件缓存落盘流程
核心函数：`move_files_to_cache()` 与 `async_move_files_to_cache()`
**定义位置**: [processing_utils.py#L431-L502](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L431-L502) 和 [processing_utils.py#L551-L603](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L551-L603)

调用时机：
- **postprocess 之后**：组件 `.postprocess()` 返回文件路径 → 移到缓存并加上 `/file=` URL 前缀
- **preprocess 之前**：前端传回文件路径 → 校验文件是否在上传目录内（安全检查）

处理逻辑（`_move_to_cache` 内部函数）：
1. 若 `payload.url` 是 HTTP URL 且处于 postprocess 阶段 → 直接使用 URL，不下载
2. 若是静态文件（通过 `gr.set_static_paths()` 注册）→ 跳过，不移动
3. 若是本地文件路径：
   - 先调用 `_check_allowed()` 做路径安全校验
   - 再调用 `Block.move_resource_to_block_cache()` 将文件复制/下载到上传目录
4. 生成前端可访问的 URL：
   - 普通文件: `/gradio_api/file=<path>`
   - 流媒体: `/gradio_api/stream/...`

### 3.3 Block 级缓存移动
**定义位置**: [blocks.py#L335-L413](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L335-L413)

`move_resource_to_block_cache(url_or_file_path)`:
- 若参数是 **HTTP URL** → 走 `ssrf_protected_download()`（带 SSRF 防护）下载到 `<upload_dir>/<url_hash>/<filename>`
- 若参数是 **本地文件** 且不在上传目录内 → 走 `save_file_to_cache()` 按内容哈希复制到 `<upload_dir>/<file_hash>/<filename>`
- 已在上传目录内 → 直接返回原路径
- 最终路径加入 `Block.temp_files` 集合，纳入清理追踪

### 3.4 内容哈希与去重
哈希种子：[utils.py#L609-L623](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L609-L623) 中的 `HASH_SEED_PATH`（首次启动生成 UUID 并持久化）

哈希函数（[processing_utils.py#L127-L156](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L127-L156)）：
- `hash_file()`: 对文件内容 + hash_seed 做 SHA-256
- `hash_url()`: 对 URL 字符串 + hash_seed 做 SHA-256
- `hash_bytes()` / `hash_base64()`: 对字节/base64 内容 + hash_seed 做 SHA-256

**效果**: 相同内容的文件（不管上传多少次、从什么路径传入）会落到同一个哈希子目录，实现磁盘去重。

### 3.5 静态文件豁免
通过 `gr.set_static_paths(paths)`（[utils.py#L1298-L1333](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1298-L1333)）注册的文件/目录：
- 判定函数: `is_static_file()`（[utils.py#L1336-L1355](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1336-L1355)）
- 效果：不复制到缓存目录，直接从原始路径提供服务，节省磁盘和启动时间

---

## 四、下载/文件获取与安全校验

### 4.1 文件获取路由
**路由定义**:
- `/gradio_api/file={path_or_url}` 和 `/file={path_or_url}`（[routes.py#L1082-L1087](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1082-L1087)）
- 静态服务器同样实现了这两个路由（[static_server.py#L85-L90](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L85-L90)）

核心处理函数：`file_fetch()`（[route_utils.py#L1190-L1255](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1190-L1255)）

### 4.2 安全校验逻辑（file_fetch 内部）
1. **HTTP URL**: 若 `path_or_url` 是完整 HTTP URL → 302 重定向到该 URL（不代理文件内容）
2. **协议检查**: 若是 SMB/UNC 等特殊协议路径 → 直接 403
3. **路径存在性**: 路径不存在或为目录 → 403
4. **文件访问白名单**（`is_allowed_file()`，[utils.py#L1788-L1806](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1788-L1806)），优先级：
   1. **blocked_paths**: 在黑名单中 → 拒绝
   2. **allowed_paths**: 在开发者显式允许的路径中 → 允许（reason="allowed"）
   3. **created_paths**: 在上传目录或示例缓存目录中（即 Gradio 自己创建的文件）→ 允许（reason="created"）
   4. 其他 → 拒绝（reason="not_created_or_allowed"）
5. **XSS 防护**: 根据 MIME 类型决定 Content-Disposition：
   - 安全 MIME（图片/音视频/纯文本/JSON）→ `inline`，直接在浏览器显示
   - 其他 MIME → `attachment`，强制下载防止脚本执行
6. **Range 请求支持**: 支持 HTTP Range 头，用于音视频拖动播放

### 4.3 路径安全拼接
- `routes_safe_join()`（[route_utils.py#L1123-L1139](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1123-L1139)）：在 `safe_join` 基础上额外做 HTTP 相关检查（空路径、协议路径、目录检查、存在性检查）
- `safe_join()`（[utils.py#L1767-L1785](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1767-L1785)）：防止路径穿越（`../`、绝对路径、备选分隔符）

### 4.4 预处理阶段的文件校验
**函数**: `check_all_files_in_cache()`（[processing_utils.py#L416-L428](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L416-L428)）

在事件数据进入用户函数前调用，确保所有文件路径：
- 要么是 HTTP URL
- 要么在上传目录中（`is_in_or_equal(path, get_upload_folder())`）
- 要么是静态文件

不在上述范围则抛出 `Error`，防止用户传入任意服务器路径读取文件。

### 4.5 _check_allowed 安全检查
**定义位置**: [processing_utils.py#L505-L548](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L505-L548)

在 `move_files_to_cache()` 中对本地路径做更细致的校验：
- `check_in_upload_folder=True`（预处理阶段）：只允许上传目录中的文件，其他一律拒绝（防止用户伪造任意路径）
- `check_in_upload_folder=False`（后处理阶段）：允许当前工作目录、系统临时目录、`allowed_paths`、上传目录；拒绝 dotfiles（`.` 开头的文件）除非显式加入 `allowed_paths`

---

## 五、临时文件清理机制

### 5.1 追踪的数据结构
在 [blocks.py#L148-L162](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L148-L162) 和 [blocks.py#L1148-L1162](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1148-L1162)：
- `Blocks.temp_file_sets`: List[Set[str]]，聚合所有 Block 的 `temp_files` 集合和根级的 `upload_file_set`
- `Block.temp_files`: Set[str]，每个组件自己产生的临时文件路径
- `Block.keep_in_cache`: Set[str]，**豁免清理**的文件路径（组件默认值、Examples 使用的文件）

### 5.2 定时清理任务
**启动入口**: `create_lifespan_handler()`（[route_utils.py#L1041-L1059](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1041-L1059)）

在 App lifespan 中启动 `delete_files_on_schedule()`：
- 每 `frequency` 秒（默认 1 秒）执行一次
- 调用 `delete_files_created_by_app(blocks, age)`
- `age` 默认 1 秒：文件创建时间超过 age 秒才会被删除

### 5.3 清理执行函数
`delete_files_created_by_app()`（[route_utils.py#L981-L1005](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L981-L1005)）逻辑：
1. 收集所有组件的 `keep_in_cache` 到 `dont_delete` 集合
2. 遍历 `blocks.temp_file_sets` 中的每个集合
3. 对每个文件：
   - 在 `dont_delete` 中 → 跳过
   - `age=None` → 无条件删除
   - `age` 有值 → 检查 `ctime`，超过 age 秒才删除
4. 已删除的路径从 temp_file_set 中移除

### 5.4 关闭时清理
在 `_lifespan_handler()` 的 `yield` 之后（App 关闭时）调用 `delete_files_created_by_app(app.get_blocks(), age=None)`，**无条件清理所有追踪到的临时文件**（除了 `keep_in_cache` 的）。

### 5.5 配置入口
通过 `gr.Blocks(delete_cache=(frequency, age))` 或 `demo.launch(delete_cache=(frequency, age))` 传入。
默认值在 `create_lifespan_handler` 中为 `frequency=1, age=1`（秒）。

---

## 六、gr.cache 内存缓存（与文件缓存的区别）

**定义位置**: [caching.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/caching.py)

这是**纯内存**的 LRU 缓存，与磁盘上的上传/缓存目录完全分离：
- `@gr.cache` 装饰器：根据函数参数的内容哈希缓存返回值
- `gr.Cache` 类：作为函数参数注入，提供手动 `get()`/`set()`/`keys()`/`clear()` KV 接口
- 支持 `per_session=True`：每个会话独立命名空间，会话断开时通过 `clear_session_caches()` 清理
- 支持 `max_size`（条目数）和 `max_memory`（内存用量）双维度 LRU 淘汰
- 被缓存装饰的函数在命中时**会跳过 Gradio Queue** 直接返回结果

---

## 七、关键设计关系总览

```
前端拖拽/选择文件
        │
        ▼
  POST /gradio_api/upload  (multipart/form-data)
        │
        ▼
  GradioMultiPartParser ── 写入 NamedTemporaryFile (系统 tmp)
        │                    实时计算 SHA-256
        │                    报告上传进度 (SSE)
        ▼
  upload_fn()
        │ os.rename / shutil.move (跨分区时 BackgroundTasks 异步复制)
        ▼
  上传目录 (GRADIO_TEMP_DIR)  ←──┐
  <upload_dir>/<sha>/<file>       │ 去重：相同内容哈希相同
        │                         │
        │  用户函数执行            │
        │  ↓ preprocess           │
        │  check_all_files_in_cache() ── 确保路径在上传目录内
        │                         │
        │  ↓ 用户函数返回文件路径   │
        │  ↓ postprocess          │
        │  move_files_to_cache() ─┘
        │    ├─ _check_allowed()  路径安全
        │    ├─ move_resource_to_block_cache()  复制到上传目录
        │    └─ 生成 /gradio_api/file=<path> URL
        ▼
  GET /gradio_api/file=<path>
        │
        ▼
  file_fetch()
        ├─ is_allowed_file() ── blocked_paths / allowed_paths / created_paths
        ├─ XSS_SAFE_MIMETYPES ── inline vs attachment
        └─ Range 请求支持 (音视频拖动)

  后台定时清理: delete_files_on_schedule() 每秒扫描 temp_file_sets
  keep_in_cache 的文件被永久豁免
```

## 八、环境变量速查

| 环境变量 | 作用 | 默认值 |
|---|---|---|
| `GRADIO_TEMP_DIR` | 上传目录 + 组件文件缓存目录 | `<系统临时目录>/gradio` |
| `GRADIO_EXAMPLES_CACHE` | Examples 预计算结果缓存目录 | `.gradio/cached_examples` |
| `GRADIO_ALLOWED_PATHS` | 逗号分隔的允许提供服务的路径列表 | 空 |
| `GRADIO_BLOCKED_PATHS` | 逗号分隔的禁止提供服务的路径列表 | 空 |
| `GRADIO_HEARTBEAT_INTERVAL` | 心跳间隔（秒），影响会话断开检测 | 15（测试环境 0.25） |
| `GRADIO_CACHE_EXAMPLES` | 是否默认缓存 Examples | false |
| `GRADIO_CACHE_MODE` | Examples 缓存模式：eager/lazy | eager |
| `GRADIO_RESET_EXAMPLES_CACHE` | 启动时清空 Examples 缓存 | false |
