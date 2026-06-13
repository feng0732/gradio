# Gradio 上传落盘与临时目录清理 — 分支差异深度解析

本文深入对比主服务（FastAPI App）与静态文件服务（StaticWorkerPool）两条路径上，文件上传的落盘、追踪、清理行为差异，并明确说明 `delete_cache` 未配置时是否存在定时回收。

---

## 1. delete_cache 配置解析与默认行为

### 1.1 参数默认值

`Blocks.__init__` 中 `delete_cache` 默认是 `None`：

- 定义位置：[blocks.py#L1067](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1067-L1067)
- 赋值位置：[blocks.py#L1092](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1092-L1092)

```python
delete_cache: tuple[int, int] | None = None
...
self.delete_cache = delete_cache
```

docstring 明确说明：*"If None, no cache deletion will occur."*

### 1.2 解析链

在 `App.create_app` 中（主服务启动入口）：

```python
delete_cache = blocks.delete_cache or (None, None)
app_kwargs["lifespan"] = create_lifespan_handler(
    app_kwargs.get("lifespan", None), *delete_cache
)
```

- 代码位置：[routes.py#L371-L375](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L371-L375)

当用户没有配置 `delete_cache` 时，`blocks.delete_cache` 是 `None`，被 `or` 替换为 `(None, None)`，于是：

```python
create_lifespan_handler(user_lifespan, None, None)
```

### 1.3 生命周期 handler 的条件启动

在 `create_lifespan_handler` 内部：

```python
if frequency and age:
    await stack.enter_async_context(_lifespan_handler(app, frequency, age))
```

- 代码位置：[route_utils.py#L1053-L1054](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1053-L1054)

由于 `None` 在 Python 中是 falsy，`if frequency and age` 条件**不成立**。这意味着：

1. **不会启动 `delete_files_on_schedule` 定时清理任务**（默认 frequency=1s, age=1s 的行为完全不会触发）
2. **不会注册关闭时的 `delete_files_created_by_app(age=None)` 全量清理**
3. 整个应用的生命周期中，**没有任何清理逻辑被挂载**

### 1.4 结论：未配置 delete_cache 时

> **在 `delete_cache=None` 的默认情况下，Gradio 既没有定时回收，也没有关闭时清理。所有文件一旦落盘就会永久保留（直到被操作系统的临时目录清理机制或用户手动删除）。**

只有当用户显式传入 `delete_cache=(frequency, age)` 且两个值都非 0/None 时，清理机制才会启用。

---

## 2. 主服务 vs 静态服务：上传路由实现差异

当启用 Node 前端代理 + `StaticWorkerPool`（`num_workers >= 1`）时，前端请求会被 `js/app/proxy_routes.js` 分类路由。所有上传/download 相关路径都属于 `STATIC_ROUTE_PREFIXES`，优先走静态 worker。

路由决策代码：[proxy_routes.js#L92-L111](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js#L92-L111)

```javascript
if (matchesPrefix(path, STATIC_ROUTE_PREFIXES)) {
    if (!hasWorkers) {
        return { route: "python" };   // 没有 worker，回退到 Python 主服务
    }
    // upload_id 亲和性哈希，保证 /upload 和 /upload_progress 落到同一 worker
    ...
    return { route: "worker" };      // 有 worker，走静态 worker
}
```

即：**有静态 worker 时，上传请求不会到达主服务。**

### 2.1 主服务 `/upload`（Python FastAPI）

代码位置：[routes.py#L1738-L1772](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1738-L1772)

```python
@router.post("/upload", dependencies=[Depends(login_check)])
async def upload_file(
    request: fastapi.Request,
    bg_tasks: BackgroundTasks,
    upload_id: str | None = None,
):
    output_files, files_to_copy, locations = await upload_fn(
        request, app.uploaded_file_dir, ..., force_move=False, ...
    )
    if files_to_copy:
        bg_tasks.add_task(move_uploaded_files_to_cache, files_to_copy, locations)
    blocks.upload_file_set.update(output_files)   # 关键：加入追踪集合
    return output_files
```

主服务做了三件事：
1. 调用 `upload_fn(force_move=False)` 解析 multipart 并尝试重命名
2. 如果跨分区导致 `os.rename` 失败，返回的 `files_to_copy` 被交给 `BackgroundTasks` 异步执行 `shutil.move`
3. **`blocks.upload_file_set.update(output_files)`** — 将最终落盘路径加入 Blocks 级追踪集合

### 2.2 静态服务 `/upload`（StaticWorkerPool 子进程）

代码位置：[static_server.py#L94-L112](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L94-L112)

```python
@app.post("/gradio_api/upload")
@app.post("/upload")
async def upload_file(
    request: fastapi.Request,
    upload_id: str | None = None,
):
    try:
        output_files, _, _ = await upload_fn(   # 注意：_, _ 丢弃了返回值
            request, upload_dir, max_file_size,
            upload_id=upload_id, force_move=False,
            upload_progress=file_upload_statuses if upload_id else None,
        )
    except MultiPartException as exc:
        ...
    return output_files
```

对比主服务，静态服务有 **三个关键缺失**：

| 行为 | 主服务 | 静态服务 |
|------|--------|----------|
| 接收 `files_to_copy` 返回值 | ✅ `output_files, files_to_copy, locations` | ❌ 用 `_, _` 丢弃 |
| `BackgroundTasks` 异步 `shutil.move` | ✅ 有 | ❌ 无（函数签名里甚至没有 `bg_tasks` 参数） |
| 加入 `blocks.upload_file_set` 追踪 | ✅ 有 | ❌ 完全没有，静态 worker 进程内根本没有 `Blocks` 对象 |

### 2.3 缺失一：跨分区落盘失败

`upload_fn` 中的落盘逻辑：

```python
try:
    os.rename(temp_file.file.name, dest)      # 同分区原子重命名
except OSError:
    if force_move:
        shutil.move(temp_file.file.name, dest)
    else:
        files_to_copy.append(temp_file.file.name)  # 跨分区：记录待复制
        locations.append(dest)
```

- 代码位置：[route_utils.py#L1308-L1317](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1308-L1317)

两个服务都传了 `force_move=False`。主服务通过 `BackgroundTasks` 兜底，调用 `move_uploaded_files_to_cache` 执行 `shutil.move`。静态服务直接丢弃 `files_to_copy`，这意味着：

> **当系统临时目录（NamedTemporaryFile 所在位置）与 `GRADIO_TEMP_DIR` 不在同一磁盘分区时，静态 worker 上传的文件会永远留在系统临时目录中，既不会被移动到上传目录，也不会被任何清理机制删除。**

（这些临时文件的文件名是 Python `tempfile` 生成的随机名，通常形如 `/tmp/tmpXXXXXX`，没有后缀。）

### 2.4 缺失二：追踪集合完全不更新

`Blocks.upload_file_set` 是 `Blocks.temp_file_sets` 的初始元素，在 `Blocks.__init__` 中构建：

```python
self.upload_file_set = set()
self.temp_file_sets = [self.upload_file_set]
```

- 代码位置：[blocks.py#L1148-L1149](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L1148-L1149)

此外，每个组件 `Block.render()` 时会把自己的 `temp_files` 集合追加到 `temp_file_sets`：

```python
root_context.root_block.temp_file_sets.append(self.temp_files)
```

- 代码位置：[blocks.py#L221](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L221-L221)

清理函数 `delete_files_created_by_app` 只遍历这些集合：

```python
for temp_set in blocks.temp_file_sets:
    for file in temp_set:
        ...  # 判断 age、豁免、删除
```

- 代码位置：[route_utils.py#L987-L1005](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L987-L1005)

**它不会扫描 `GRADIO_TEMP_DIR` 目录本身。**

因此：
- 主服务上传的文件：加入 `upload_file_set` → 被清理函数感知（如果配置了 `delete_cache`）
- 静态服务上传的文件：**不加入任何集合** → 即使配置了 `delete_cache`，清理函数也看不到它们 → 文件永久存在

---

## 3. StaticWorkerPool 多进程架构与清理调度

### 3.1 静态 worker 的生命周期

静态 worker 在 `Blocks.launch()` 中启动：

```python
self._static_worker_pool = StaticWorkerPool(
    num_workers=resolved_num_workers, config=static_config, ports=worker_ports,
)
self._static_worker_pool.start()
```

- 代码位置：[blocks.py#L3010-L3015](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L3010-L3015)

每个 worker 是一个独立的 `multiprocessing.Process`，入口函数是 `_run_static_worker`：

```python
def _run_static_worker(port: int, config_dict: dict):
    config = StaticServerConfig(**config_dict)
    app = create_static_app(config)
    uvicorn.run(app, host="127.0.0.1", port=port, log_level="warning")
```

- 代码位置：[static_server.py#L170-L174](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L170-L174)

### 3.2 静态 worker 进程内部状态

静态 worker 内的 FastAPI app 由 `create_static_app` 创建，该函数：
- 创建了独立的 `FileUploadProgress` 实例（上传进度追踪只在本进程内有效）
- 注册了 `/upload`、`/upload_progress`、`/file=`、静态资源等路由
- **没有调用 `create_lifespan_handler`**，即没有任何 lifespan 上下文管理器
- **没有 `Blocks` 对象**，没有 `temp_file_sets`、`keep_in_cache`、`delete_cache` 等概念
- **没有任何定时任务或清理逻辑**

静态 worker 只是一个薄的、无状态（除了上传进度）的文件传输进程。

### 3.3 主进程关闭时的静态 worker 处理

`Blocks.close()` 中：

```python
if hasattr(self, "_static_worker_pool") and self._static_worker_pool is not None:
    self._static_worker_pool.shutdown()
    self._static_worker_pool = None
```

- 代码位置：[blocks.py#L3339-L3343](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L3339-L3343)

`StaticWorkerPool.shutdown()` 只是 `process.kill()` + `process.join()`：

```python
def shutdown(self):
    for process in self.workers:
        process.kill()
    for process in self.workers:
        process.join(timeout=0.5)
```

- 代码位置：[static_server.py#L235-L242](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L235-L242)

静态 worker 被 **SIGKILL（kill()）** 直接终止，没有执行任何清理钩子（即使它有，进程内也没有可执行的清理逻辑）。

---

## 4. 后处理/组件级文件追踪的补偿机制

虽然上传路由在静态 worker 上不追踪，但用户上传的文件会经过组件的 `preprocess` → 业务函数 → `postprocess` 管线。在这条链路上有两次机会加入追踪：

### 4.1 move_files_to_cache

在 `processing_utils.move_files_to_cache` 中，后处理阶段会调用：

```python
temp_file_path = block.move_resource_to_block_cache(payload.path)
...
block.temp_files.add(temp_file_path)
```

- 代码位置：[processing_utils.py#L474-L479](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py#L474-L479)

### 4.2 Block.move_resource_to_block_cache

```python
else:
    temp_file_path = url_or_file_path   # 已经在 GRADIO_CACHE 内，不做复制
self.temp_files.add(temp_file_path)
```

- 代码位置：[blocks.py#L369-L371](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L369-L371)

**然而**，这一补偿机制有严格的前提：
1. 文件必须参与了至少一次 Gradio 事件（被组件 `preprocess` 或 `postprocess` 处理）
2. 处理它的组件必须已经 `render()` 过（从而其 `temp_files` 被加入 `Blocks.temp_file_sets`）
3. 用户必须显式配置了 `delete_cache=(frequency, age)`

如果用户只是上传了文件但没有触发事件（比如仅用于前端预览、拖拽后取消等），这些文件永远不会进入任何追踪集合。

---

## 5. 四种场景的实际行为汇总

| 场景 | 是否落盘到上传目录 | 是否被 temp_file_sets 追踪 | 配置 delete_cache 后能否被清理 | 未配置 delete_cache 时能否被清理 |
|------|-------------------|--------------------------|-------------------------------|--------------------------------|
| 主服务上传 + 参与事件 | ✅ | ✅（upload_file_set + temp_files） | ✅ | ❌ |
| 主服务上传 + 未参与事件 | ✅ | ✅（upload_file_set） | ✅ | ❌ |
| 静态服务上传 + 参与事件 | ✅（同分区）<br>⚠️ 跨分区则留在系统 tmp | ✅（通过 move_resource_to_block_cache 补偿） | ✅（仅事件处理过的） | ❌ |
| 静态服务上传 + 未参与事件 | ✅（同分区）<br>⚠️ 跨分区则留在系统 tmp | ❌ | ❌ | ❌ |

**磁盘泄露风险最高的组合**：启用了 `StaticWorkerPool` + 系统临时目录与上传目录不在同一分区 + 用户频繁上传但不触发事件。此时：
- 跨分区的临时文件永远留在 `/tmp`（或 Windows 临时目录）
- 同分区但未参与事件的文件永远留在 `GRADIO_TEMP_DIR`
- 由于静态 worker 多进程并行，泄露速度会随 worker 数量线性增长

---

## 6. 相关代码索引

| 功能模块 | 文件 | 关键行号 |
|---------|------|---------|
| `delete_cache` 参数默认值 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L1067, L1092 |
| `delete_cache` 解析为 (None, None) | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L371-L380 |
| `create_lifespan_handler` 条件启动清理 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py) | L1041-L1059 |
| `delete_files_created_by_app` 遍历集合（非目录扫描） | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py) | L981-L1005 |
| 主服务 `/upload` 路由 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py) | L1738-L1772 |
| 静态服务 `/upload` 路由 | [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py) | L94-L112 |
| `upload_fn` force_move 分支 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py) | L1258-L1320 |
| `Blocks.upload_file_set` + `temp_file_sets` 初始化 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L1148-L1149 |
| `Block.render()` 注册 temp_files 到 temp_file_sets | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L221 |
| `StaticWorkerPool` 启动与 shutdown | [static_server.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py) | L177-L243 |
| `StaticWorkerPool` 在 launch() 中创建 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2982-L3020 |
| 前端路由分类（worker vs python） | [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js) | L83-L116 |
| `move_files_to_cache` 后处理补偿追踪 | [processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/processing_utils.py) | L431-L502 |
| `move_resource_to_block_cache` 加入 temp_files | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L335-L413 |
