# Gradio Image 组件：输入输出全链路分析

## 概览

`Image` 组件的完整处理链路分为输入侧和输出侧：

**输入侧**：
```
前端上传 → ImageData payload → Image.preprocess() → image_utils.preprocess_image() → 用户函数
```

**输出侧**：
```
用户函数返回值 → Image.postprocess() → image_utils.postprocess_image() → ImageData/Base64ImageData → 前端显示
```

[Image.preprocess()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/components/image.py#L194-L209) 将输入逻辑委托给 [image_utils.preprocess_image()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L264-L325)。

[Image.postprocess()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/components/image.py#L211-L225) 将输出逻辑委托给 [image_utils.postprocess_image()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L328-L359)。

`ImageData` 数据结构定义在 [data_classes.py#L429-L447](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/data_classes.py#L429-L447)：

```python
class ImageData(GradioModel):
    path: str | None       # 服务端本地文件路径
    url: str | None        # 公开 URL 或 base64 data URL
    size: int | None       # 文件大小（字节）
    orig_name: str | None  # 原始文件名
    mime_type: str | None  # MIME 类型
    is_stream: bool = False
    meta: dict = {"_type": "gradio.FileData"}

class Base64ImageData(GradioModel):
    url: str               # base64 编码的图片 data URL
```

- `ImageData`：完整的图像数据表示，既可以指向本地文件（`path`），也可以包含 base64 数据或远程 URL（`url`）
- `Base64ImageData`：简化的 base64 表示，仅用于 streaming 输出场景

---

## 一、preprocess_image 完整执行顺序

函数签名：

```python
def preprocess_image(
    payload: ImageData | None,
    cache_dir: str,
    format: str,
    image_mode: "1"|"L"|"P"|"RGB"|"RGBA"|... | None,
    type: "numpy"|"pil"|"filepath",
) -> np.ndarray | PIL.Image.Image | str | None
```

以下是 **逐行** 的执行分支：

### 1. payload 为 None（[L275-L276](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L275-L276)）

```python
if payload is None:
    return payload
```

- 用户没有上传图片时直接返回 None

### 2. base64 data URL 分支（[L277-L283](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L277-L283)）

```python
if payload.url and payload.url.startswith("data:"):
    if type == "pil":
        return decode_base64_to_image(payload.url)
    elif type == "numpy":
        return decode_base64_to_image_array(payload.url)
    elif type == "filepath":
        return decode_base64_to_file(payload.url, cache_dir, format)
```

**触发场景**：前端 canvas 裁剪、webcam 快照、剪贴板粘贴

**子分支处理**：

| type | 处理流程 | EXIF 旋转 | image_mode 转换 | 重编码 |
|------|---------|-----------|----------------|--------|
| `pil` | `base64 → bytes → PIL.Image.open()` | ✅ 有（在 `decode_base64_to_image` 内） | ❌ 无 | ❌ 无 |
| `numpy` | 先解码为 PIL，再 `np.asarray()` | ✅ 有 | ❌ 无 | ❌ 无 |
| `filepath` | 解码为 PIL → `save_image()` 存缓存 → 返回路径 | ✅ 有 | ❌ 无 | ✅ 有（按 `format` 参数） |

**EXIF 旋转实现**（[L191-L203](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L191-L203)）：
```python
def decode_base64_to_image(encoding):
    img = PIL.Image.open(BytesIO(base64.b64decode(...)))
    if hasattr(ImageOps, "exif_transpose"):
        img = ImageOps.exif_transpose(img)
    return img
```

⚠️ **注意**：base64 分支 **不做** `image_mode` 转换。如果 `image_mode="RGB"` 而 base64 图片是 RGBA，返回的仍然是 RGBA。

### 3. 路径缺失检查（[L284-L285](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L284-L285)）

```python
if payload.path is None:
    raise ValueError("Image path is None.")
```

- 如果 payload 既不是 base64 data URL，也没有 `path`，抛出 **`ValueError`**
- 这是一个内部异常，不会显示给最终用户

### 4. 解析文件名与格式后缀（[L286-L295](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L286-L295)）

```python
file_path = Path(payload.path)
if payload.orig_name:
    p = Path(payload.orig_name)
    name = p.stem
    suffix = p.suffix.replace(".", "")
    if suffix in ["jpg", "jpeg"]:
        suffix = "jpeg"
else:
    name = "image"
    suffix = "webp"
```

- 从 `orig_name` 提取文件名主干和格式后缀
- `jpg` 统一归一化为 `jpeg`
- 无 `orig_name` 时默认后缀为 `webp`

### 5. SVG 特殊处理（[L297-L300](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L297-L300)）

```python
if suffix.lower() == "svg":
    if type == "filepath":
        return str(file_path)
    raise Error("SVG files are not supported as input images for this app.")
```

| 条件 | 行为 |
|------|------|
| `suffix == "svg"` 且 `type == "filepath"` | 直接返回原文件路径 |
| `suffix == "svg"` 且 `type != "filepath"` | 抛出 **`gr.Error`** —— 这是用户可见的模态框错误 |

⚠️ **注意**：这里抛出的是 `gradio.exceptions.Error`（[exceptions.py#L69](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/exceptions.py#L69)），不是普通的 `ValueError`。`gr.Error` 会在前端显示为红色模态框。

### 6. 用 PIL 打开文件（[L302](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L302)）

```python
im = PIL.Image.open(file_path)
```

### 7. 快速路径 —— 直接返回原路径（[L303-L304](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L303-L304)）

```python
if type == "filepath" and (image_mode in [None, im.mode]):
    return str(file_path)
```

**触发条件**：
- `type == "filepath"`
- `image_mode` 为 `None`，或 `image_mode` 与图片实际 mode 相同

**⚠️ 关键行为**：
- ✅ **直接返回原文件路径**，不做任何处理
- ❌ **跳过 EXIF 旋转**（如果图片有 EXIF 旋转信息，用户拿到的是方向不正确的原图）
- ❌ **跳过 image_mode 转换**
- ❌ **跳过重编码**

**设计权衡**：这是为了避免不必要的文件 I/O 和重编码，但代价是可能返回方向不正确的图片。例如用户上传一张手机拍摄的竖版照片（EXIF 标记为旋转 90°），当 `type="filepath"` 且 `image_mode="RGB"`（默认值，而 JPEG 本身就是 RGB）时，会触发快速路径，用户函数拿到的文件方向是错误的。

### 8. EXIF 旋转（[L306-L312](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L306-L312)）

```python
exif = im.getexif()
if exif.get(274, 1) != 1 and hasattr(ImageOps, "exif_transpose"):
    try:
        im = ImageOps.exif_transpose(im)
    except Exception:
        warnings.warn(f"Failed to transpose image {file_path} based on EXIF data.")
```

- EXIF tag 274 = Orientation，值为 1 表示方向正确
- 方向不正确时用 `ImageOps.exif_transpose` 自动旋转
- 旋转失败仅发出 `warnings.warn`，不中断流程

### 9. image_mode 转换（[L313-L317](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L313-L317)）

```python
if suffix.lower() != "gif" and im is not None:
    with warnings.catch_warnings():
        warnings.simplefilter("ignore")
        if image_mode is not None:
            im = im.convert(image_mode)
```

| 条件 | 行为 |
|------|------|
| `suffix == "gif"` | **跳过**转换，保留原始帧结构和模式 |
| `image_mode is None` | **跳过**转换，保留原始模式（如 PNG 的 RGBA） |
| `image_mode` 非 None 且非 GIF | 转换为目标模式（默认 `"RGB"`，会将 RGBA/L/P 等转为 RGB） |

### 10. format_image 统一输出（[L319-L325](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L319-L325)）

```python
return format_image(im, type=type, cache_dir=cache_dir, name=name, format=suffix)
```

[format_image()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L48-L81) 根据 `type` 返回三种格式：

| type | 返回 | 说明 |
|------|------|------|
| `"pil"` | `PIL.Image.Image` | 直接返回 |
| `"numpy"` | `np.ndarray` | `np.array(im)`，shape=(H, W, C)，dtype=uint8 |
| `"filepath"` | `str` | 保存为缓存文件，优先用 `suffix` 格式，失败回退 `png` |

---

## 二、异常处理汇总

| 异常场景 | 异常类型 | 抛出位置 | 用户可见 |
|---------|---------|---------|---------|
| `payload.path is None`（非 base64 分支） | `ValueError("Image path is None.")` | L285 | ❌ 内部错误 |
| SVG 输入且 `type != "filepath"` | `gr.Error("SVG files are not supported...")` | L300 | ✅ 模态框 |
| base64 EXIF 旋转失败 | `print()`（不是 warn） | L198-201 | ❌ 仅日志 |
| 文件路径 EXIF 旋转失败 | `warnings.warn()` | L312 | ❌ 仅日志 |
| `format_image` 未知 type | `ValueError` | L77 | ❌ 内部错误 |
| `open_image` 未知类型 | `ValueError` | L41 | ❌ 内部错误 |

---

## 三、文件保存逻辑

### 3.1 save_image（[L84-L110](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L84-L110)）

```python
def save_image(y, cache_dir, format="webp"):
    if isinstance(y, np.ndarray):
        path = processing_utils.save_img_array_to_cache(y, cache_dir, format)
    elif isinstance(y, PIL.Image.Image):
        path = processing_utils.save_pil_to_cache(y, cache_dir, format)
    elif isinstance(y, Path):
        path = str(y)
    elif isinstance(y, str):
        path = y
```

- **np.ndarray** → 先转 PIL 再保存
- **PIL.Image** → 直接保存
- **Path/str** → 原样返回（假设已是有效文件路径）

### 3.2 save_pil_to_cache（[processing_utils.py#L160-L171](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/processing_utils.py#L160-L171)）

```python
def save_pil_to_cache(img, cache_dir, name="image", format="webp"):
    bytes_data = encode_pil_to_bytes(img, format)
    temp_dir = Path(cache_dir) / hash_bytes(bytes_data)
    temp_dir.mkdir(exist_ok=True, parents=True)
    filename = str((temp_dir / f"{name}.{format}").resolve())
    (temp_dir / f"{name}.{format}").resolve().write_bytes(bytes_data)
    return filename
```

关键特性：
- **内容寻址**：用 `SHA-256(bytes + hash_seed)` 生成目录名，相同内容 → 相同路径 → 天然去重
- **格式处理**：GIF 保留所有帧（`save_all=True`），PNG 保留元数据，其他格式保留 EXIF

### 3.3 保存回退机制（format_image 中）

```python
elif type == "filepath":
    try:
        path = processing_utils.save_pil_to_cache(im, cache_dir, name=name, format=format)
    except (KeyError, ValueError):
        path = processing_utils.save_pil_to_cache(im, cache_dir, name=name, format="png")
```

- 优先使用原始后缀格式保存（如 `jpeg`、`webp`）
- 若 PIL 不支持该格式，回退为 `png`

---

## 四、完整流程图

```
preprocess_image(payload, ...)
    │
    ├─ payload is None → return None
    │
    ├─ payload.url starts with "data:" (base64 分支)
    │    ├─ type=pil   → decode_base64_to_image()          → PIL.Image
    │    │                  └─ 有 EXIF 旋转，无 mode 转换
    │    ├─ type=numpy → decode_base64_to_image_array()    → np.ndarray
    │    │                  └─ 有 EXIF 旋转，无 mode 转换
    │    └─ type=filepath → decode_base64_to_file()         → 保存为缓存文件 → str
    │                       └─ 有 EXIF 旋转，无 mode 转换，有重编码
    │
    ├─ payload.path is None → raise ValueError("Image path is None.")
    │
    ├─ 解析 orig_name → name, suffix
    │
    ├─ suffix == "svg"
    │    ├─ type == "filepath" → return str(file_path)
    │    └─ 否则 → raise gr.Error("SVG files are not supported...")
    │
    ├─ im = PIL.Image.open(file_path)
    │
    ├─ type == "filepath" AND (image_mode is None OR image_mode == im.mode)
    │    └─ 【快速路径】return str(file_path)
    │       └─ ⚠️ 跳过 EXIF 旋转，跳过 mode 转换，跳过重编码
    │
    ├─ EXIF 旋转 (exif tag 274 != 1)
    │
    ├─ suffix != "gif" AND image_mode is not None
    │    └─ im = im.convert(image_mode)
    │
    └─ format_image(im, type, ...)
         ├─ type=pil   → PIL.Image
         ├─ type=numpy → np.array(im) → np.ndarray
         └─ type=filepath → save_pil_to_cache() → str（优先 suffix，失败回退 png）
```

---

## 五、关键设计问题与权衡

### 问题 1：快速路径跳过 EXIF 旋转

当 `type="filepath"` 且 `image_mode` 与图片 mode 匹配时，**快速路径直接返回原文件，不做 EXIF 旋转**。这意味着：
- 用户上传的手机照片（通常带 EXIF 旋转标记）方向可能不正确
- 这是性能（避免重编码）与正确性的权衡

### 问题 2：base64 分支与文件路径分支的归一化不一致

| 处理 | base64 分支 | 文件路径分支（非快速路径） |
|------|------------|------------------------|
| EXIF 旋转 | ✅ 有 | ✅ 有 |
| image_mode 转换 | ❌ 无 | ✅ 有（非 GIF 且 mode 非 None） |
| GIF 特殊处理 | ❌ 无 | ✅ 跳过 mode 转换 |

### 问题 3：SVG 仅支持 filepath 模式

SVG 是矢量图，无法转为 numpy 数组或 PIL Image，因此仅在 `type="filepath"` 时允许输入。

---

## 六、输出侧：postprocess_image 完整分析

### 6.1 函数签名

```python
def postprocess_image(
    value: np.ndarray | PIL.Image.Image | str | Path | None,
    cache_dir: str,
    format: str,
    watermark: WatermarkOptions | None = None,
) -> ImageData | None
```

用户函数可以返回以下类型的值：
- `np.ndarray`：numpy 数组形式的图像
- `PIL.Image.Image`：PIL 图像对象
- `str` / `Path`：本地文件路径或 URL
- `None`：无图像

### 6.2 逐分支执行顺序

#### 1. 值为 None（[L343-L344](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L343-L344)）

```python
if value is None:
    return None
```

直接返回 `None`，前端显示为空。

#### 2. SVG 文件特殊处理（[L345-L354](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L345-L354)）

```python
if isinstance(value, str) and value.lower().endswith(".svg"):
    svg_content = extract_svg_content(value)
    if watermark is not None:
        Warning("Watermarking for SVG images is currently not supported...")
    return ImageData(
        orig_name=Path(value).name,
        url=f"data:image/svg+xml,{quote(svg_content)}",
    )
```

**处理逻辑**：
- 判断 `value` 是字符串且以 `.svg` 结尾
- 调用 `extract_svg_content()` 读取 SVG 内容（支持本地文件和 HTTP URL）
- 如果设置了水印，发出警告（SVG 不支持水印）
- 返回 `ImageData`，`url` 为 `data:image/svg+xml,...` 形式的 data URL
- **不设置 `path`**，前端直接通过 `url` 渲染 SVG

**安全性说明**：SVG 内容通过 `quote()` 进行 URL 编码后内联到 data URL 中，避免 XSS 风险。

#### 3. 水印叠加（[L355-L356](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L355-L356)）

```python
if watermark and watermark.watermark is not None:
    value = add_watermark(value, watermark)
```

- 仅当配置了水印且水印图片不为 None 时生效
- `add_watermark()` 内部先调用 `open_image()` 将水印转为 PIL Image
- 在 `RGBA` 模式下进行 alpha 合成，然后转回原图模式
- 水印位置支持预设（top-left/top-right/bottom-left/bottom-right）或自定义坐标
- 水印越界时自动调整到右下角（10px 边距）

#### 4. 保存到缓存（[L357](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L357)）

```python
saved = save_image(value, cache_dir=cache_dir, format=format)
```

调用 `save_image()` 将图像保存到缓存目录，返回文件绝对路径。

`save_image` 的分支逻辑（[L84-L110](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L84-L110)）：

| value 类型 | 处理 |
|-----------|------|
| `np.ndarray` | `save_img_array_to_cache()` → 转 PIL → 保存 |
| `PIL.Image.Image` | `save_pil_to_cache()` → 直接保存 |
| `Path` / `str` | 原样返回路径（假设已是有效文件） |

⚠️ **注意**：如果 `value` 是字符串类型的**远程 URL**（如 `https://...`），`save_image` 会直接返回该 URL 字符串，不会下载到本地。此时 `Path(saved).exists()` 为 `False`。

#### 5. 构造 ImageData 返回（[L358-L359](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L358-L359)）

```python
orig_name = Path(saved).name if Path(saved).exists() else None
return ImageData(path=saved, orig_name=orig_name)
```

- `path`：设置为保存后的文件路径（或原始字符串/URL）
- `orig_name`：仅当文件存在时设置为文件名，URL 场景下为 `None`
- `url`：**不设置**，由前端或路由层根据 `path` 生成可访问的 URL

### 6.3 postprocess 返回值的使用

`Image.postprocess()` 的返回类型声明为 `ImageData | Base64ImageData | None`（[image.py#L213](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/components/image.py#L213)），但实际 `postprocess_image()` 只返回 `ImageData | None`。

返回的 `ImageData` 会通过以下路径到达前端：
1. 在 Blocks 事件处理中，`postprocess` 的结果会被 `model_dump()` 序列化为字典
2. 路由层将 `path` 转换为可通过 `/file=...` 端点访问的 URL
3. 前端 Image 组件根据 `url` 或 `path` 渲染图像

> **关于 `Base64ImageData`**：目前 `postprocess_image()` 函数本身并不直接返回 `Base64ImageData`。该类型主要用于 API 文档声明（`api_info_as_output` 中 `streaming == "base64"` 时）以及 MCP 等协议层。实际的 base64 转换由 `encode_image_to_base64()` / `encode_image_file_to_base64()` 等工具函数在其他调用点完成。

---

## 七、Streaming 场景

`Image` 组件的 `streaming` 参数在输入侧和输出侧有不同的含义。

### 7.1 输入侧 streaming：Webcam 流

当 `streaming=True` 且 `sources=["webcam"]` 时，组件支持 webcam 实时视频流输入。

- 继承自 `StreamingInput` 接口（[base.py#L404-L411](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/components/base.py#L404-L411)）
- `check_streamable()` 验证 streaming 配置（仅允许 webcam 单源）
- 前端以固定间隔（`stream_every`，默认 0.5 秒）将 webcam 帧作为图片发送给后端
- 每帧图像通过正常的 `preprocess_image()` 流程处理

### 7.2 输出侧 streaming：Base64 模式

`Image` 组件的 `streaming` 参数文档说明：*"If the component is an output component, will automatically convert images to base64."*

从代码层面可以看到以下设计：

**1. API 文档层面**（[image.py#L227-L232](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/components/image.py#L227-L232)）：

```python
def api_info_as_output(self) -> dict[str, Any]:
    if self.streaming == "base64":
        schema = Base64ImageData.model_json_schema()
        schema.pop("description", None)
        return schema
    return self.api_info()
```

- 当 `self.streaming == "base64"` 时，API 输出 schema 为 `Base64ImageData`
- `Base64ImageData` 只有 `url` 字段，值为 base64 data URL

**2. 数据模型层面**（[data_classes.py#L445-L447](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/data_classes.py#L445-L447)）：

```python
class Base64ImageData(GradioModel):
    url: str = Field(description="base64 encoded image")
```

**3. 工具函数层面**：`image_utils.py` 中提供了 base64 编码函数：

- [encode_image_to_base64()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L227-L232)：PIL Image → JPEG base64
- [encode_image_file_to_base64()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L235-L240)：图像文件 → base64（保留原始格式）
- [encode_image_array_to_base64()](file:///d:/fz/0601/solo-dogfeeding/code/251-gradio/gradio/image_utils.py#L216-L224)：numpy 数组 → JPEG base64

### 7.3 streaming 的完整图景

| 场景 | streaming 值 | 行为 |
|------|-------------|------|
| 输入侧 webcam 流 | `True`（bool） | 启用 webcam 实时流输入，前端定时发送帧 |
| 输出侧 base64 | `"base64"`（str） | API 输出为 Base64ImageData，前端直接渲染 base64 |
| 非 streaming | `False`（默认） | 正常文件路径传输，通过 `/file=` 端点访问 |

**注意**：
- `Image` 组件**不继承** `StreamingOutput`（与 Video/Audio 不同）
- 输出侧的 base64 streaming 主要用于 API 层面的简化，以及需要减少 HTTP 请求的场景
- 与 Video/Audio 的 chunk streaming 不同，Image 的 streaming 是单帧 base64 传输

---

## 八、输出侧流程图

```
用户函数返回 value (np.ndarray|PIL|str|Path|None)
    │
    ├─ value is None → return None
    │
    ├─ value 是 str 且以 .svg 结尾
    │    ├─ extract_svg_content() 读取内容
    │    ├─ 有水印 → Warning（不支持）
    │    └─ return ImageData(url="data:image/svg+xml,...", orig_name=...)
    │       (path 为 None)
    │
    ├─ 有水印配置 → add_watermark(value, watermark)
    │    └─ 转为 RGBA 模式叠加后转回
    │
    ├─ save_image(value, cache_dir, format)
    │    ├─ np.ndarray → 转 PIL → save_pil_to_cache
    │    ├─ PIL.Image → save_pil_to_cache
    │    └─ str/Path → 原样返回
    │
    └─ return ImageData(path=saved, orig_name=...)
       (url 为 None，由路由层/前端生成可访问 URL)
```

---

## 九、完整数据流转图

### 输入侧（preprocess）

```
前端 ImageData
    ├─ url 为 data: base64
    │    ├─ type=pil → PIL.Image
    │    ├─ type=numpy → np.ndarray
    │    └─ type=filepath → 解码保存 → 缓存文件路径
    │
    └─ path 为本地文件
         ├─ SVG + type=filepath → 原路径
         ├─ SVG + 其他 type → gr.Error
         ├─ filepath + mode匹配 → 【快速路径】原路径
         └─ 通用路径 → EXIF旋转 → mode转换 → format_image
```

### 输出侧（postprocess）

```
用户返回值
    ├─ None → None
    ├─ SVG 路径 → ImageData(url=data:image/svg+xml,...)
    ├─ numpy/PIL/路径 → 水印 → save_image → ImageData(path=...)
    └─ streaming="base64" → Base64ImageData(url=data:image/...;base64,...)
```

---

## 十、设计问题与权衡（续）

### 问题 4：输出侧字符串路径不区分本地文件和 URL

`save_image()` 对 `str` 类型直接返回，不检查是本地路径还是远程 URL。这意味着：
- 如果用户函数返回 URL，`postprocess_image` 会将其作为 `path` 返回
- `orig_name` 会因 `Path(saved).exists() == False` 而为 `None`
- 前端可能无法正确显示远程 URL 的图片

### 问题 5：输入输出的对称性

| 能力 | 输入侧 preprocess | 输出侧 postprocess |
|------|-----------------|------------------|
| base64 处理 | ✅ 完整支持 | ⚠️ 仅 SVG 用 data URL，普通图不直接返回 base64 |
| EXIF 旋转 | ✅ 有（非快速路径） | ❌ 无（输出时保留原始方向） |
| image_mode 转换 | ✅ 有（非快速路径、非 GIF） | ❌ 无（按原始格式保存） |
| SVG 支持 | ⚠️ 仅 filepath | ✅ 内联为 data URL |
| 水印 | — | ✅ 有 |

### 问题 6：快速路径的两面性

输入侧的快速路径（`type=filepath` + mode 匹配）：
- ✅ 优点：零拷贝、零重编码，性能最优
- ❌ 缺点：跳过 EXIF 旋转，可能导致方向错误
- ❌ 缺点：与 base64 分支、非快速路径的行为不一致
