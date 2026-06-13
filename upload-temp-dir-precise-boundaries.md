# Gradio 静态 Worker 精确边界分析：鉴权暴露条件、路径限制与跨分区组件后果

本文精确回答两个边界问题：
1. 鉴权漏洞在什么条件下实际暴露，文件访问具体受哪些路径限制
2. 跨分区上传后，不同文件组件遇到"空路径"时的具体后果分支

---

## 1. 鉴权漏洞的精确暴露条件

### 1.1 StaticWorkerPool 启用的三重前置条件

不是所有 Gradio 应用都会启动静态 Worker。代码中 `resolved_num_workers >= 1` 需要满足以下条件：

**条件 A：必须在生产 SSR 模式下（有 Node 前端代理）**

```python
if self.ssr_mode:
    ...
    elif self.node_path:          # 有可用的 Node 可执行文件
        ...
        if resolved_num_workers is not None and resolved_num_workers >= 1:
            # 启动 StaticWorkerPool
```

- 代码位置：[blocks.py#L2847-L2889](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L2847-L2889)

非 SSR 模式（`ssr_mode=False`，默认）或 SSR 开发模式（`is_dev_mode`，使用 vite）都不会走到这里。

**条件 B：`num_workers` 显式 >= 1 或 `GRADIO_NUM_WORKERS` 环境变量设置为 >= 1**

```python
resolved_num_workers = num_workers
if resolved_num_workers is None:
    env_val = os.environ.get("GRADIO_NUM_WORKERS")
    if env_val is not None:
        resolved_num_workers = int(env_val)
```

- 代码位置：[blocks.py#L2832-L2836](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L2832-L2836)

两个值都没有设置（默认情况）时，`resolved_num_workers` 保持 `None`，不启动静态 Worker。

**条件 C：不是 SSR 开发模式（vite dev server）**

SSR 开发模式走 Python 前端代理 vite 的架构，没有 Node 做 front proxy，静态 Worker 不启动。

### 1.2 实际暴露条件汇总

鉴权漏洞（上传/下载绕过 login_check）**当且仅当**以下所有条件同时成立：

| # | 条件 | 来源 |
|---|------|------|
| 1 | `ssr_mode=True`（或 `GRADIO_SSR_MODE=True`） | [blocks.py#L2828](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L2828-L2828) |
| 2 | 系统上有 `node` 可执行文件 | [blocks.py#L2848](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L2848-L2848) |
| 3 | `num_workers >= 1`（launch 参数或 `GRADIO_NUM_WORKERS`） | [blocks.py#L2832-L2836](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L2832-L2836) |
| 4 | 应用配置了 `auth=` 或 `auth_dependency=`（需要认证才有意义） | [routes.py#L408-L419](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L408-L419) |

**不暴露的常见场景**：
- 纯 Python 直接跑（没有 SSR、没有 Node） → 条件 1/2 不成立
- `gr.load()` 加载远程 Space → 完全不同的架构，不涉及本地静态 Worker
- SSR 模式但没设置 `num_workers` / 环境变量 → 条件 3 不成立
- SSR + num_workers >= 1 但没有 `auth` → 条件 4 不成立，谈不上"绕过"

### 1.3 无静态 Worker 时的回退行为

在 Node 代理 + SSR 模式下，如果 `num_workers=0` 或未设置，`classifyRoute` 的处理：

```javascript
if (matchesPrefix(path, STATIC_ROUTE_PREFIXES)) {
    if (!hasWorkers) {
        return { route: "python" };   // ← 回退到 Python 主服务
    }
    ...
    return { route: "worker" };
}
```

- 代码位置：[proxy_routes.js#L92-L111](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js#L92-L111)

此时上传和文件访问**全部路由到 Python 主服务**，`login_check` 依赖正常工作，无鉴权缺口。

---

## 2. 文件访问的四层路径安全判定

无论走主服务还是静态 Worker，`file_fetch` 在通过登录层（如果有的话）之后，内部还有四层路径安全判定。

### 2.1 is_allowed_file 四层判定

`is_allowed_file` 定义于 [utils.py#L1788-L1806](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py#L1788-L1806)：

```python
def is_allowed_file(path, blocked_paths, allowed_paths, created_paths):
    in_blocklist = any(is_in_or_equal(path, bp) for bp in blocked_paths)
    if in_blocklist:
        return False, "in_blocklist"                    # 第 1 层：黑名单最高优先级
    if any(is_in_or_equal(path, ap) for ap in allowed_paths):
        return True, "allowed"                          # 第 2 层：用户显式白名单
    if any(is_in_or_equal(path, cp) for cp in created_paths):
        return True, "created"                          # 第 3 层：应用自身创建的路径
    return False, "not_created_or_allowed"              # 第 4 层：其余全部拒绝
```

判定是严格的短路顺序，一旦命中就立即返回。

### 2.2 file_fetch 中的参数取值

**主服务**调用方式 [routes.py#L1085-L1086](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/routes.py#L1085-L1086)：

```python
return file_fetch(path_or_url, request, blocks, app.uploaded_file_dir)
```

传入的 `blocks` 对象含 `allowed_paths` 和 `blocked_paths`。`file_fetch` 内部构造：

```python
allowed, reason = utils.is_allowed_file(
    abs_path,
    blocked_paths=blocks_or_config.blocked_paths,
    allowed_paths=blocks_or_config.allowed_paths + _StaticFiles.all_paths,
    created_paths=[upload_dir, str(utils.get_cache_folder())],
)
```

- 代码位置：[route_utils.py#L1213-L1218](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1213-L1218)

**静态 Worker** 调用方式 [static_server.py#L89-L90](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L89-L90)：

```python
return file_fetch(path_or_url, request, config, upload_dir)
```

这里传入的是 `StaticServerConfig` 对象（不是 `blocks`），其字段：

```python
@dataclass
class StaticServerConfig:
    allowed_paths: list[str] = field(default_factory=list)
    blocked_paths: list[str] = field(default_factory=list)
    ...
```

- 代码位置：[static_server.py#L42-L49](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/static_server.py#L42-L49)

这些值在 launch 时从 Blocks 复制注入 [blocks.py#L2990-L2998](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py#L2990-L2998)。

### 2.3 四层白名单的精确含义

| 层 | 含义 | 填充来源 | 是否可通过 created_paths 访问任意系统文件？ |
|----|------|---------|------------------------------------------|
| 1 blocked_paths | 用户显式拒绝的路径 | `launch(blocked_paths=...)` + `GRADIO_BLOCKED_PATHS` 环境变量 | — |
| 2 allowed_paths | 用户显式允许的路径 | `launch(allowed_paths=...)` + `GRADIO_ALLOWED_PATHS` 环境变量 + `gr.set_static_paths()` | 此层命中的文件走 inline（非强制 attachment 下载） |
| 2a _StaticFiles.all_paths | 静态文件注册 | `gr.set_static_paths(["dir/"])` 或 SVG 图标自动注册 | 同 allowed_paths，inline |
| 3 created_paths | 应用自身创建的目录 | 固定两个：`upload_dir`（= GRADIO_TEMP_DIR）+ `get_cache_folder()`（=.gradio/cached_examples） | ❌ 不能访问 cwd、/etc 等其他目录 |
| 4 其他全部 | 拒绝 | — | 403 Forbidden |

### 2.4 路径安全的旁路检查

`file_fetch` 在调用 `is_allowed_file` **之前**还有三道前置检查：

```python
if client_utils.is_http_url_like(path_or_url):      # 1. 是 HTTP URL → 302 重定向
    return RedirectResponse(...)
if starts_with_protocol(path_or_url):               # 2. 含协议头（smb://、file://）→ 403
    raise HTTPException(403, "File not allowed")
abs_path = utils.abspath(path_or_url)
if abs_path.is_dir() or not abs_path.exists():      # 3. 是目录或不存在 → 403
    raise HTTPException(403, "File not allowed")
```

- 代码位置：[route_utils.py#L1196-L1209](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py#L1196-L1209)

关键：第 3 步的 `not abs_path.exists()` 返回 403，和第 4 层拒绝的状态码、错误信息完全一致，外部无法区分是"文件不存在"还是"路径不被允许"。

---

## 3. 跨分区上传后的精确状态

### 3.1 时间点状态表

以下表格描述静态 Worker 在跨分区上传完成（HTTP 响应 200 OK）时，相关路径的状态：

| 时间点 | `GRADIO_TEMP_DIR/<hash>/filename`（返回给前端的 dest） | 系统临时目录 `/tmp/tmpXXXXXX`（NamedTemporaryFile） |
|--------|------------------------------------------------------|---------------------------------------------------|
| multipart 解析中 | ❌ 不存在 | ✅ 正在写入，文件未关闭 |
| `upload_fn` 进入 rename 分支前 | ❌ 不存在（但 `directory.mkdir()` 已创建空的 hash 目录） | ✅ 文件已关闭 |
| `os.rename` 抛出 OSError（跨分区） | ❌ 不存在 | ✅ 完整文件在 |
| `output_files.append(dest)` 执行后 | ❌ 不存在（目录存在） | ✅ 完整文件在 |
| 响应 200 OK 返回给前端 | ❌ 不存在（目录存在） | ✅ 完整文件在（永久残留） |
| 后续任意时间 | ❌ 不存在（目录永久为空壳） | ✅ 文件永久残留，文件名随机 `tmpXXXXXX` 无后缀 |

### 3.2 前端视角

`@gradio/client` 的 upload 函数在收到响应后：

```javascript
return response.files.map((f, i) => {
    const file = new FileData({
        ...file_data[i],
        path: f,                                         // 空壳路径
        url: `${root_url}${this.api_prefix}/file=${f}`  // 构造的下载 URL
    });
    return file;
});
```

- 代码位置：[client/js/src/upload.ts#L31-L39](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/client/js/src/upload.ts#L31-L39)

前端不会做存在性校验，立即进入以下两种消费路径之一。

---

## 4. 各文件组件对"空路径"的精确后果

以下分析均假设：**文件路径为 `GRADIO_TEMP_DIR/<hash>/filename`，但该文件实际不存在**（跨分区上传未被移动的情况）。

每个组件分两个场景分析：
- **场景 A**：前端直接预览（请求 `/file=` URL）
- **场景 B**：触发事件，在 preprocess 阶段处理

---

### 4.1 gr.File（文件组件）

**组件代码**：[components/file.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/file.py)

**场景 A：前端预览**

组件前端通常只显示文件名、大小、图标等元数据，不主动请求 `/file=` URL，除非用户点击"下载"。点击下载时请求走静态 Worker 的 `file_fetch`，因 `not abs_path.exists()` 返回 **403 Forbidden**。

**场景 B：preprocess** → `_process_single_file`（[file.py#L143-L157](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/file.py#L143-L157)）

| type 分支 | 代码路径 | 结果 |
|-----------|---------|------|
| `type="filepath"`（默认） | 第 152 行：创建一个新的空 `NamedTemporaryFile(delete=False, dir=self.GRADIO_CACHE)`，然后 `file.name = file_name`，再用 `file_name` 构造 `NamedString` 返回 | ✅ **不崩溃**！组件完全不读取原文件。用户函数拿到的是正确的路径字符串（但指向不存在的文件）；如果用户函数内部 `open()` 才会自行报错 |
| `type="binary"` | 第 156 行：`with open(file_name, "rb") as file_data: return file_data.read()` | ❌ **FileNotFoundError**：`[Errno 2] No such file or directory: '.../GRADIO_TEMP_DIR/<hash>/filename'` |

关键：`type="filepath"` 的默认行为实际上创建了一个**无关的空临时文件**，但返回给用户函数的仍然是 `file_name`（原不存在路径），不是新文件的路径。用户函数稍后自行访问时才会崩溃。

---

### 4.2 gr.Image（图片组件）

**组件代码**：[components/image.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/image.py) → 实际调用 `image_utils.preprocess_image`

**场景 A：前端预览**

`<img src="/gradio_api/file=...">` → 静态 Worker `file_fetch` → `not abs_path.exists()` → **403 Forbidden**，浏览器显示"图片破裂"图标。

**场景 B：preprocess_image**（[image_utils.py#L265-L325](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/image_utils.py#L265-L325)）

前几关：
1. SVG 分支跳过（通常是 jpg/png）
2. `payload.path` 不为空

到达 **第 302 行**：

```python
im = PIL.Image.open(file_path)   # ← 崩溃点
```

无论后续 `type` 是 `filepath`、`pil` 还是 `numpy`，都要先经过这一步（注意 `type="filepath"` 的短路判断在第 303 行，在 `Image.open` 之后）：

```python
im = PIL.Image.open(file_path)
if type == "filepath" and (image_mode in [None, im.mode]):
    return str(file_path)
```

**因此对 gr.Image 而言，所有 type 分支都崩溃**，异常类型：
- `FileNotFoundError: [Errno 2] No such file or directory: '...'`（被 `blocks.py` 的 `ComponentProcessingError` 包装后返回前端）

---

### 4.3 gr.Audio（音频组件）

**组件代码**：[components/audio.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/audio.py)

**场景 A：前端预览**

`<audio src="/gradio_api/file=...">` → 403 Forbidden → 音频控件灰色不可用。

**场景 B：preprocess**（[audio.py#L228-L267](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/audio.py#L228-L267)）

| type 分支 | 代码路径 | 结果 |
|-----------|---------|------|
| `type="numpy"`（默认） | 第 251 行：`return processing_utils.audio_from_file(payload.path)` | ❌ `audio_from_file` 内部使用 scipy/librosa 读取 → `FileNotFoundError` |
| `type="filepath"` 且 `format is None` 或格式匹配 | 第 253-254 行：`if not needs_conversion: return payload.path` | ✅ **不崩溃**！直接返回路径字符串。用户函数内部访问文件时才会报错 |
| `type="filepath"` 且格式不匹配 | 第 255-260 行：`audio_from_file(path)` → 转换格式 → 写新文件 | ❌ 第 255 行 `audio_from_file` 立即 `FileNotFoundError` |

---

### 4.4 gr.Video（视频组件）

**组件代码**：[components/video.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/video.py)

**场景 A：前端预览**

`<video src="/gradio_api/file=...">` → 403 Forbidden → 黑屏。

**场景 B：preprocess**（[video.py#L192-L251](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/video.py#L192-L251)）

前置检查通过（`payload.path` 非空）。后续分三条路：

| 条件分支 | 代码路径 | 结果 |
|---------|---------|------|
| `needs_formatting=True`（需要转码）或 `flip=True` | 第 221-238 行：构造 FFmpeg，`inputs={str(file_name): None}` → `ff.run()` | ❌ FFmpeg 输入文件不存在 → ffmpeg-python 抛 `Error`（通常是 `ffmpeg 返回非零退出码 + stderr`） |
| `not include_audio`（去除音频） | 第 239-249 行：同上，`ff.run()` 构造 `-an` 转码 | ❌ 同上，FFmpeg 找不到输入文件 |
| 默认分支（不转码、不去音频） | 第 250-251 行：`return str(file_name)` | ✅ **不崩溃**！直接返回路径字符串。用户函数内部访问才报错 |

---

### 4.5 gr.Model3D（3D 模型组件）

**组件代码**：[components/model3d.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/model3d.py)

**场景 B：preprocess**（[model3d.py#L120-L129](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/model3d.py#L120-L129)）只有三行：

```python
def preprocess(self, payload: FileData | None) -> str | None:
    if payload is None:
        return payload
    return payload.path    # ← 原样返回，完全不读文件
```

✅ **始终不崩溃**。用户函数拿到路径后如果用 trimesh/open3d 等库读取才会自行报错。

---

### 4.6 gr.Gallery（图库组件）

**组件代码**：[components/gallery.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/gallery.py)

**场景 B：preprocess**（[gallery.py#L187-L229](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/gallery.py#L187-L229)）对每个 Gallery 元素：

```python
for gallery_element in payload.root:
    ...
    media = (
        gallery_element.video.path                    # GalleryVideo：直接返回路径
        if (type(gallery_element) is GalleryVideo)
        else self.convert_to_type(gallery_element.image.path, self.type)  # GalleryImage
    )
```

`convert_to_type` 静态方法（[gallery.py#L321-L328](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/gallery.py#L321-L328)）：

| type 分支 | 代码 | 结果 |
|-----------|------|------|
| `type="filepath"`（默认） | `return img` | ✅ 不崩溃 |
| `type="pil"` 或 `type="numpy"` | `PIL.Image.open(img)` | ❌ `FileNotFoundError` |
| GalleryVideo 任何 type | 始终 `return gallery_element.video.path` | ✅ 不崩溃 |

---

### 4.7 组件后果总览表

| 组件 | 预览（/file= 请求） | preprocess type=filepath（默认） | preprocess 其他 type |
|------|---------------------|-------------------------------|---------------------|
| **gr.File** | 403（仅点击下载时） | ✅ 返回路径字符串（但创建了一个无关空临时文件） | ❌ FileNotFoundError（type=binary） |
| **gr.Image** | 403 破裂图标 | ❌ FileNotFoundError（PIL.Image.open 在所有分支之前） | ❌ FileNotFoundError |
| **gr.Audio** | 403 灰色控件 | ✅ 返回路径（格式匹配时）；❌ 格式转换需读文件 | ❌ FileNotFoundError（type=numpy） |
| **gr.Video** | 403 黑屏 | ✅ 返回路径（默认分支不转码） | ❌ ffmpeg 错误（转码/去音频时） |
| **gr.Model3D** | 403 无法加载 | ✅ 直接 return payload.path | —（无其他 type） |
| **gr.Gallery（图片）** | 403 每张图破裂 | ✅ 返回路径（默认 type=filepath） | ❌ FileNotFoundError（pil/numpy） |
| **gr.Gallery（视频）** | 403 黑屏 | ✅ 始终返回路径 | — |

**规律**：组件的 `preprocess` 越"薄"（只做路径透传，不做解码），越容易在用户函数内部才暴露问题；越是需要提前解码的组件（Image、Audio 的 numpy 模式），越早在管线内崩溃并包装成用户可见的错误。

---

## 5. 相关代码索引

| 模块 | 文件 | 关键行号 |
|------|------|---------|
| StaticWorkerPool 启用三条件 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2828, L2832-L2836, L2847-L2889 |
| `num_workers` docstring（仅 SSR 生效） | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2696 |
| classifyRoute 无 worker 时回退到 python | [proxy_routes.js](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/js/app/proxy_routes.js) | L92-L95 |
| `is_allowed_file` 四层判定 | [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/utils.py) | L1788-L1806 |
| `file_fetch` 前置三道检查 + is_allowed_file 调用 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/route_utils.py) | L1196-L1220 |
| `StaticServerConfig` 注入 allowed/blocked_paths | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/blocks.py) | L2990-L2998 |
| `_StaticFiles.all_paths` 静态文件注册 | [data_classes.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/data_classes.py) | L348-L361 |
| gr.File._process_single_file | [file.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/file.py) | L143-L157 |
| `preprocess_image`（PIL.Image.open 在所有 type 之前） | [image_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/image_utils.py) | L265-L325 |
| gr.Audio.preprocess 三条分支 | [audio.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/audio.py) | L228-L267 |
| gr.Video.preprocess FFmpeg 分支 | [video.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/video.py) | L192-L251 |
| gr.Model3D.preprocess 纯透传 | [model3d.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/model3d.py) | L120-L129 |
| gr.Gallery.convert_to_type | [gallery.py](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/gradio/components/gallery.py) | L321-L328 |
| Client upload 响应处理（不校验存在性） | [upload.ts](file:///d:/fz/0601/solo-dogfeeding/code/243-gradio/client/js/src/upload.ts) | L31-L39 |
