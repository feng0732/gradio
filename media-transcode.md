# Video & Audio 转码流程梳理

本文档梳理 Gradio 中 Video 和 Audio 组件的媒体转码全流程，明确**上传落盘、格式处理、浏览器播放**三者之间的边界和数据流转。

---

## 1. 整体架构概览

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                               前端 (浏览器)                                    │
│                                                                              │
│  ┌──────────────────── Video ────────────────────┐    ┌────── Audio ──────┐  │
│  │  Upload  ─▶  FFmpeg WASM  ─▶  <video> 播放    │    │  Upload           │  │
│  │                (裁剪用)                        │    │    │              │  │
│  │                            ▲                   │    │    ▼              │  │
│  │                            │ HLS.js            │    │  WaveSurfer.js    │  │
│  └────────────────────────────┼───────────────────┘    │  / <audio> 播放   │  │
│                               │                         │       ▲          │  │
│                               │ HLS                     │       │ HLS.js   │  │
└───────────────────────────────┼─────────────────────────┴───────┼──────────┘
                                │ HTTP (multipart/form-data)      │
                                ▼                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                               后端 (Python)                                   │
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐          │
│  │ upload 路由  │───▶│ preprocess   │───▶│  用户预测函数        │          │
│  └──────────────┘    └──────────────┘    └──────────┬───────────┘          │
│         │            move_files_to_cache            │                       │
│         ▼                    ▲                      ▼                       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐          │
│  │  file 路由   │◀───│ postprocess  │◀───│  move_files_to_cache │          │
│  └──────────────┘    └──────────────┘    └──────────────────────┘          │
│         ▲                                                                   │
│         │ /stream/ (HLS m3u8 + 分片)                                         │
│  ┌──────────────┐                                                           │
│  │ stream 路由  │◀── MediaStream (内存) ◀── stream_output                    │
│  └──────────────┘                                                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

> **重要修正**：FFmpeg WASM 仅用于 Video 前端裁剪，**Audio 前端不使用 FFmpeg**。
> Audio 播放通过 WaveSurfer.js 或原生 `<audio>` 实现，流媒体通过 HLS.js 解码。

---

## 2. 上传与落盘流程

### 2.1 前端上传

**入口组件**：
- Video：[InteractiveVideo.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/InteractiveVideo.svelte)
- Audio：[InteractiveAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/interactive/InteractiveAudio.svelte)
- 通用上传：[Upload.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/upload/src/Upload.svelte)

**上传方式**：
- 拖拽上传 / 点击选择文件
- 摄像头录制（Video）/ 麦克风录制（Audio）
- 录制使用浏览器原生 `MediaRecorder` API

**上传调用链**：
```
Upload.svelte
  → prepare_files()  [@gradio/client]
  → client.upload()
  → POST /gradio_api/upload  (multipart/form-data)
```

### 2.2 后端接收与落盘

**路由**：[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/routes.py#L1738-L1772) 中的 `upload_file`

**落盘函数**：[upload_fn()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/route_utils.py#L1258-L1320)

**落盘位置**：`app.uploaded_file_dir`

落盘目录结构：
```
{uploaded_file_dir}/{sha256_hash}/{filename}
```

其中 `sha256_hash` 是文件内容的哈希，用于去重。

**核心逻辑**：
```python
# routes.py L1748-L1757
output_files, files_to_copy, locations = await upload_fn(
    request,
    app.uploaded_file_dir,  # 落盘目录
    blocks.max_file_size,
    upload_id,
    force_move=False,
    upload_progress=file_upload_statuses if upload_id else None,
)

# 后台异步将文件从临时位置移动到最终位置
if files_to_copy:
    bg_tasks.add_task(
        move_uploaded_files_to_cache, files_to_copy, locations
    )
```

**返回格式**：`FileData` 对象列表
```python
FileData(
    path="/path/to/uploaded/file.mp4",
    orig_name="original_filename.mp4",
    size=123456,
    meta={"_type": "gradio.FileData"}
)
```

---

## 3. 文件缓存与 URL 生成机制

### 3.1 缓存目录层级

Gradio 有三层文件存储，各司其职：

| 层级 | 目录变量 | 位置来源 | 用途 | 生命周期 |
|------|---------|---------|------|---------|
| 第一层 | `uploaded_file_dir` | `get_upload_folder()` | 上传文件暂存 | 请求期间 |
| 第二层 | `GRADIO_CACHE` (block cache) | `get_upload_folder()` | 处理后的媒体文件缓存 | 组件生命周期 |
| 第三层 | 系统临时目录 | `tempfile.gettempdir()` | 转码过程中的临时文件 | 函数调用结束 |

> **注意**：`uploaded_file_dir` 和 `GRADIO_CACHE` 实际上是同一个根目录（都是 `get_upload_folder()`），它们的区别在于：
> - `uploaded_file_dir` 是上传路由使用的目录变量名
> - `GRADIO_CACHE` 是每个 Block 实例持有的缓存目录变量名
> - 两者都指向 `GRADIO_TEMP_DIR` 环境变量或 `{tempdir}/gradio`

### 3.2 move_files_to_cache 详解

**位置**：[processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L431-L502)

**核心作用**：
1. 将文件从上传目录/用户路径移动/复制到 block cache
2. 生成可访问的 URL（`/file=...` 或 `/stream/...`）
3. 安全检查（SSRF 防护、路径白名单）

**调用时机（共 4 个场景）**：

| 调用场景 | 代码位置 | 方向 | 参数 |
|---------|---------|------|------|
| 组件初始化 | [base.py L214](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/base.py#L214-L219) | 初始值 → 前端 | `postprocess=True, keep_in_cache=True` |
| 输入 preprocess 前 | [blocks.py L1849](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/blocks.py#L1849-L1853) | 前端上传 → 后端 | `check_in_upload_folder=True` |
| 输出 postprocess 后 | [blocks.py L2062](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/blocks.py#L2062-L2066) | 后端输出 → 前端 | `postprocess=True` |
| 流式输出 | [blocks.py L2127](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/blocks.py#L2127-L2131) | 流式输出 → 前端 | `postprocess=True` |

**URL 生成规则**：
```python
# 普通文件
url = f"{API_PREFIX}/file={payload.path}"

# 流式文件（is_stream=True）
url = f"{API_PREFIX}/stream/" + ...
```

### 3.3 move_resource_to_block_cache

Block 级别的文件缓存方法，是 move_files_to_cache 的底层实现之一。

**同步版本**：[blocks.py L375](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/blocks.py#L375-L410)
**异步版本**：[blocks.py L335](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/blocks.py#L335-L373)

**处理逻辑**：
- 如果是 HTTP URL：下载到 cache dir（SSRF 防护）
- 如果是本地路径且不在 cache dir 内：复制到 cache dir
- 如果已经在 cache dir 内：直接返回路径
- 所有文件路径记入 `self.temp_files` 集合，用于生命周期管理

---

## 4. 两种文件 URL：/file= 与 /stream/

### 4.1 /file= URL（普通文件服务）

**路由**：[routes.py L1082-L1086](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/routes.py#L1082-L1086)
```
GET  /gradio_api/file={path}
HEAD /gradio_api/file={path}
```

**处理函数**：[file_fetch()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/route_utils.py#L1190-L1256)

**特点**：
- 直接从磁盘读取文件
- 支持 HTTP Range 请求（断点续传/拖动播放）
- 有完整的安全检查（路径白名单、目录遍历防护）
- MIME 类型判断：安全类型 inline 播放，其他 attachment 下载
- 用于所有非流式的媒体文件播放

**安全机制**：
```python
# 路径白名单检查
allowed, reason = is_allowed_file(
    abs_path,
    blocked_paths=blocks_or_config.blocked_paths,
    allowed_paths=blocks_or_config.allowed_paths + _StaticFiles.all_paths,
    created_paths=[upload_dir, str(get_cache_folder())],
)
```

### 4.2 /stream/ URL（HLS 流媒体服务）

**路由**：
- Playlist：`/gradio_api/stream/{session_hash}/{run}/{component_id}/playlist.m3u8`
- 分片：`/gradio_api/stream/{session_hash}/{run}/{component_id}/{segment_id}.{ext}`
- 合并文件：`/gradio_api/stream/{session_hash}/{run}/{component_id}/playlist-file`

**实现位置**：[routes.py L1104-L1180](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/routes.py#L1104-L1180)

**存储结构**：[MediaStream 类](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/route_utils.py#L1062-L1082)
```python
class MediaStream:
    segments: list[MediaStreamChunk]   # 内存中的分片列表
    combined_file: str | None           # 合并后的文件路径
    ended: bool                         # 流是否结束
    max_duration: int                   # 最大分片时长
```

**特点**：
- 分片数据存储在**内存**中（MediaStream 对象）
- 使用 HLS 协议（m3u8 播放列表 + ts/aac 分片）
- 支持渐进式播放（边生成边播放）
- 流结束后可合并为完整文件

**两种分片格式**：
| 类型 | 扩展名 | MIME 类型 | 适用组件 |
|------|--------|----------|---------|
| 视频 | `.ts` | `video/MP2T` | Video |
| 音频 | `.aac` | `audio/aac` | Audio |

### 4.3 对比总结

| 维度 | /file= | /stream/ |
|------|--------|----------|
| 数据来源 | 磁盘文件 | 内存中的 MediaStream 对象 |
| 协议 | HTTP 静态文件 | HLS (m3u8 + 分片) |
| 适用场景 | 普通媒体文件播放 | 流式输出（边生成边播放） |
| 支持 Range | 是 | 否（由 HLS 分片机制处理） |
| 生命周期 | 取决于 cache 清理策略 | session/run 生命周期 |
| 组件范围 | 所有文件类组件 | StreamingOutput 组件（开启 streaming） |

---

## 5. 格式处理（转码）

格式处理发生在两个阶段：**preprocess**（输入处理）和 **postprocess**（输出处理）。

### 5.1 Video 格式处理

**核心文件**：[video.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py)

#### 5.1.1 Preprocess（输入 → 用户函数）

[video.py preprocess](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py#L191-L251)

**触发条件**：
- `self.format` 不为 None 且上传格式 != 指定格式
- 来源为 webcam 且开启镜像翻转
- `include_audio=False` 需要去除音频

**转码工具**：FFmpeg（通过 [ffmpy](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/_vendor/ffmpy/ffmpy.py) 封装）

**处理逻辑**：
```python
if needs_formatting or flip:
    # 使用 FFmpeg 转换格式/翻转/去音频
    ff = FFmpeg(
        inputs={str(file_name): None},
        outputs={output_file_name: output_options},
    )
    ff.run()
elif not self.include_audio:
    # 仅去除音频轨
    ff = FFmpeg(
        inputs={str(file_name): None},
        outputs={output_file_name: ["-an"]},
    )
    ff.run()
```

**返回值**：文件路径字符串（传给用户函数）

#### 5.1.2 Postprocess（用户函数 → 输出）

[video.py postprocess](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py#L253-L353) → `_format_video()`

**处理步骤**：

1. **URL 直返**：如果是 HTTP URL 且不需要转换和水印，直接返回 URL
   
2. **下载 URL 文件**：如果是 URL 且需要处理，先下载到缓存
   ```python
   video = processing_utils.save_url_to_cache(video, cache_dir=self.GRADIO_CACHE)
   ```

3. **浏览器兼容性检查**：检查视频是否可在浏览器播放
   - 检查函数：[video_is_playable()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L1071-L1100)
   - 使用 `ffprobe` 检测容器和编码
   - 可播放组合：
     - `.mp4` + `h264` / `av1`
     - `.webm` + `vp9` / `vp8` / `av1`
     - `.ogg` + `theora`

4. **自动转换为可播放 mp4**：如果不可播放，自动转换
   - 转换函数：[convert_video_to_playable_mp4()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L1103-L1124)
   - 使用 FFmpeg 转换为 mp4（默认 h264 编码）

5. **用户指定格式转换**：如果 `self.format` 不为 None 且格式不匹配
6. **水印处理**：如果配置了水印，叠加水印图片

**返回值**：`FileData` 对象

### 5.2 Audio 格式处理

**核心文件**：[audio.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py)

#### 5.2.1 Preprocess（输入 → 用户函数）

[audio.py preprocess](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L228-L267)

**两种输出类型**（由 `type` 参数控制）：

| type 值 | 输出格式 | 工具 |
|---------|---------|------|
| `"numpy"` | `(sample_rate: int, data: np.ndarray)` 元组 | pydub + numpy |
| `"filepath"` | 文件路径字符串 | pydub（如需格式转换） |

**转码工具**：pydub（底层也是 FFmpeg）

**numpy 模式处理逻辑**：
```python
# processing_utils.py L670-L697
audio = AudioSegment.from_file(filename)  # pydub 读取
data = np.array(audio.get_array_of_samples())
if audio.channels > 1:
    data = data.reshape(-1, audio.channels)
return audio.frame_rate, data
```

**filepath 模式处理逻辑**：
- 如果不需要转换：直接返回原路径
- 如果需要转换：先读成 numpy，再用 `audio_to_file()` 写出指定格式

#### 5.2.2 Postprocess（用户函数 → 输出）

[audio.py postprocess](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L269-L319)

**支持的输入格式**：

| 输入类型 | 处理方式 |
|---------|---------|
| `bytes` | 保存到缓存，自动检测格式（wav/mp3） |
| `(sample_rate, data)` tuple | 用 `save_audio_to_cache()` 保存为指定格式（默认 wav） |
| `str` / `Path` 文件路径 | 如需格式转换则转换，否则直接使用 |
| `None` | 返回 None |

**格式转换函数**：
- [audio_from_file()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L670-L697)：文件 → numpy
- [audio_to_file()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L700-L717)：numpy → 文件
- [convert_to_16_bit_audio()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L720-L757)：统一转为 16-bit 整型

**返回值**：`FileData` 对象

---

## 6. Video 与 Audio 可播放性保障对比

这是 Video 和 Audio 组件在设计上最重要的差异之一。

### 6.1 Video：主动保障浏览器可播放

Video 组件有**完整的浏览器兼容性检查和自动转码机制**，确保输出视频一定能在浏览器中播放。

**保障机制**：
1. **格式检测**：用 `ffprobe` 检测容器格式和视频编码
2. **可播放判定**：对照白名单（mp4+h264, webm+vp9 等）
3. **自动转码**：不可播放则自动转成 mp4(h264)
4. **失败回退**：转码失败则返回原文件（尽力而为）

**相关代码**：
```python
# processing_utils.py L1071-L1124
def video_is_playable(video_filepath: str) -> bool:
    # 用 ffprobe 检测容器和编码
    probe = FFprobe(...)
    video_codec = output["streams"][0]["codec_name"]
    return (container, video_codec) in [
        (".mp4", "h264"), (".mp4", "av1"),
        (".webm", "vp9"), (".webm", "vp8"), (".webm", "av1"),
        (".ogg", "theora"),
    ]

def convert_video_to_playable_mp4(video_path: str) -> str:
    # 自动转成 mp4（默认 h264 编码）
    ff = FFmpeg(...)
    ff.run()
```

**设计理念**：
- 视频编码格式繁多（h264, h265, vp9, av1, theora 等），容器格式也多（mp4, mkv, avi, mov 等）
- 浏览器支持的组合非常有限
- 用户可能上传任意格式的视频，必须主动保障可播放性
- 因此 Video 组件承担了"浏览器兼容性守门员"的角色

### 6.2 Audio：按格式透传或转写

Audio 组件**没有浏览器可播放性检查**，采用"按 format 参数透传/转码"的策略。

**核心差异**：
- 没有类似 `audio_is_playable()` 的检测函数
- 没有自动转码为浏览器兼容格式的逻辑
- 完全依赖 `format` 参数控制输出格式
- `format=None` 时原样透传，能否播放依赖浏览器

**format 参数行为**：

| format 值 | 输入为文件路径 | 输入为 numpy tuple |
|-----------|--------------|------------------|
| `None` | 原样透传，不转换 | 默认保存为 wav |
| `"wav"` | 转换为 wav | 保存为 wav |
| `"mp3"` | 转换为 mp3 | 保存为 mp3 |

**相关代码**：
```python
# audio.py postprocess L304-L316
if self.format is not None and original_suffix != f".{self.format}":
    # 只有 format 明确指定时才转换
    sample_rate, data = processing_utils.audio_from_file(str(value))
    file_path = processing_utils.save_audio_to_cache(
        data, sample_rate, format=self.format, ...
    )
else:
    # 否则直接使用原文件
    file_path = str(value)
```

**设计理念**：
- 音频格式相对简单，主流浏览器普遍支持 wav, mp3, ogg, flac 等
- `format` 参数让用户明确控制输出格式
- numpy 输入默认用 wav（无损、通用）
- 整体更"轻量"，依赖浏览器原生能力

### 6.3 对比总结表

| 维度 | Video | Audio |
|------|-------|-------|
| 可播放性检测 | 有（`video_is_playable()` + ffprobe） | 无 |
| 自动转码保障 | 有（不可播放则转 mp4/h264） | 无（依赖 format 参数） |
| format 参数作用 | 输入输出格式转换 + 浏览器兼容 | 输入输出格式转换 |
| format=None 行为 | 仍会做浏览器兼容性检查 | 原样透传（文件）/ 默认 wav（numpy） |
| 转码工具 | ffmpy（直接调用 ffmpeg） | pydub（封装 ffmpeg） |
| 失败处理 | 转码失败返回原文件（警告） | 依赖 pydub/ffmpeg 抛出异常 |
| 编码考虑 | 必须考虑视频编码（h264, vp9 等） | 不单独考虑编码，由格式决定 |

---

## 7. 浏览器播放

前端播放是整个链路的最终环节。Video 和 Audio 各自有多种播放路径，根据文件类型（普通文件/流式）和配置参数选择不同的播放器实现。

### 7.1 Video 播放

**核心播放组件**：[Video.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Video.svelte)

**上层组件**：
- 静态模式：[VideoPreview.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/VideoPreview.svelte)
- 交互模式：[Player.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Player.svelte)

#### 7.1.1 两种播放路径

Video 有**两条播放路径**，由 `is_stream` 属性决定：

| 路径 | 触发条件 | 实现方式 | 代码位置 |
|------|---------|---------|---------|
| 原生 `<video>` | `is_stream=false`（普通文件） | 直接设置 `video.src` | [Video.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Video.svelte) |
| HLS.js | `is_stream=true`（流式输出） | HLS.js 加载 m3u8 → 绑定到 `<video>` | [Video.svelte L70-L110](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Video.svelte#L70-L110) |

**播放选择逻辑**：
```javascript
// Video.svelte
if (is_stream && Hls.isSupported()) {
    // 路径1：HLS.js 流式播放
    const hls = new Hls({ lowLatencyMode: true });
    hls.loadSource(src);       // 加载 m3u8 播放列表
    hls.attachMedia(node);      // 绑定到 <video> 元素
} else {
    // 路径2：原生 <video> 播放普通文件
    // src 直接设置到 <video src=...>
}
```

**支持的浏览器可播放格式**（后端保证）：
- MP4 + H.264 / AV1
- WebM + VP9 / VP8 / AV1
- Ogg + Theora

#### 7.1.2 前端 FFmpeg（WASM）

**仅 Video 组件使用**，Audio 无此功能。

- 位置：[js/video/shared/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/utils.ts)
- 用途：视频裁剪（`trimVideo` 函数）
- 库：`@ffmpeg/ffmpeg` + `@ffmpeg/util`
- 加载方式：从 `/static/ffmpeg/` 加载 WASM 核心
- 触发时机：用户在前端对视频进行裁剪编辑时

### 7.2 Audio 播放

**核心播放组件**：[AudioPlayer.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/player/AudioPlayer.svelte)

**上层组件**：
- 静态模式：[StaticAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/static/StaticAudio.svelte)
- 交互模式：[InteractiveAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/interactive/InteractiveAudio.svelte)

#### 7.2.1 三条播放路径

Audio 有**三条播放路径**，由两个维度决定：
1. 是否为流式（`is_stream`）
2. 是否显示波形（`show_recording_waveform`）

| 路径 | 播放引擎 | 触发条件 | 适用场景 |
|------|---------|---------|---------|
| 路径 A | WaveSurfer.js | `show_recording_waveform=true` 且 非流式 | 默认模式，显示波形 |
| 路径 B | 原生 `<audio>` | `show_recording_waveform=false` 且 非流式 | 简化模式，原生控件 |
| 路径 C | HLS.js + `<audio>` | `is_stream=true` | 流式输出 |

#### 7.2.2 路径 A：WaveSurfer.js 波形播放

**触发条件**：`waveform_options.show_recording_waveform = true`（默认）且非流式

**实现方式**：
- WaveSurfer.js 负责解码音频、绘制波形、控制播放
- 内部使用 Web Audio API
- 不依赖 `<audio>` 标签播放（但组件中仍有隐藏的 `<audio>` 用于流式场景）

**相关代码**：
```javascript
// AudioPlayer.svelte L199-L207
function load_audio(data: string): void {
    stream_active = false;
    if (waveform_options.show_recording_waveform) {
        waveform?.load(data);   // WaveSurfer.js 加载并解码音频
    } else if (audio_player) {
        audio_player.src = data;  // 原生 <audio>
    }
}
```

**特点**：
- 支持精细的波形显示和交互
- 支持裁剪、缩放等编辑操作
- 需要完整下载后才能绘制波形
- 依赖浏览器 Web Audio API

#### 7.2.3 路径 B：原生 `<audio>` 播放

**触发条件**：`show_recording_waveform = false` 且非流式

**实现方式**：直接使用浏览器原生 `<audio controls>` 控件

**相关代码**：
```html
<!-- AudioPlayer.svelte L390-L400 -->
<audio
    class="standard-player"
    class:hidden={use_waveform}
    controls
    bind:this={audio_player}
    preload="metadata"
></audio>
```

**特点**：
- 浏览器原生 UI，风格随浏览器而异
- 支持 HTTP Range 请求，可边下边播
- 内存占用小
- 无波形显示

#### 7.2.4 路径 C：HLS.js 流式播放

**触发条件**：`value.is_stream = true`（流式输出模式）

**实现方式**：
- HLS.js 加载 m3u8 播放列表
- 将解码后的音视频流绑定到 `<audio>` 元素
- 不使用 WaveSurfer.js（流式时禁用波形）

**相关代码**：
```javascript
// AudioPlayer.svelte L219-L261
function load_stream(value: FileData | null): void {
    if (Hls.isSupported() && !stream_active) {
        const hls = new Hls({ lowLatencyMode: true });
        hls.loadSource(value.url);     // 加载 m3u8
        hls.attachMedia(audio_player);  // 绑定到 <audio>
        hls.on(Hls.Events.MANIFEST_PARSED, function () {
            if (waveform_settings.autoplay) audio_player.play();
        });
    } else if (!stream_active) {
        // 浏览器原生支持 HLS 时（如 Safari）
        audio_player.src = value.url;
    }
}
```

**特点**：
- 边生成边播放，低延迟
- 使用 HLS 协议（m3u8 + aac 分片）
- 波形显示不可用（`use_waveform = false` 当 is_stream 时）
- 有错误自动恢复机制（网络错误、媒体错误）

#### 7.2.5 播放路径选择流程图

```
                    接收到 Audio FileData
                            │
                            ▼
                     is_stream == true ?
                       /            \
                     是              否
                     │                │
                     ▼                ▼
               HLS.js 播放     show_recording_waveform ?
               (路径 C)             /          \
                                   是            否
                                   │              │
                                   ▼              ▼
                           WaveSurfer.js    原生 <audio>
                           (路径 A)         (路径 B)
```

**支持格式**：依赖浏览器原生支持（wav, mp3, ogg, flac, aac 等）

> **重要**：Audio 前端**不使用 FFmpeg WASM**。所有音频解码和播放都依赖浏览器原生能力（Web Audio API 或 `<audio>` 标签）。

---

## 8. 流媒体（Streaming）

### 8.1 Video 流媒体

**输出流媒体**：
- 格式：`.ts` (MPEG-TS) + H.264 编码
- 传输：HLS (m3u8 playlist)
- 路由：`/stream/{session_hash}/{run}/{component_id}/playlist.m3u8`

**转换函数**：[async_convert_mp4_to_ts()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py#L502-L524)

**流合并**：[combine_stream()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py#L526-L581)
- 将多个 .ts 片段用 ffmpeg concat 合并为一个 mp4

### 8.2 Audio 流媒体

**输出流媒体**：
- 格式：`.aac` (ADTS 容器)
- 转换：[covert_to_adts()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L331-L332)

**流合并**：[combine_stream()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L366-L385)
- 直接拼接字节，保存为 mp3
- 可指定输出格式转换

---

## 9. 全流程时序：以文件上传到播放为例

### 9.1 Video 输入输出完整链路

```
前端上传
  │
  ▼
POST /gradio_api/upload
  │  (multipart/form-data)
  ▼
upload_fn()  →  落盘到 uploaded_file_dir/{hash}/{filename}
  │
  ▼
返回 FileData(path=..., orig_name=...)
  │
  ▼
前端发起 predict 请求（携带 FileData）
  │
  ▼
async_move_files_to_cache()  ←─── preprocess 前的安全检查
  │  check_in_upload_folder=True
  ▼
Video.preprocess()
  │  ├─ 格式转换（如 format 指定）
  │  ├─ 翻转（webcam + mirror）
  │  └─ 去音频（include_audio=False）
  │
  ▼
用户函数（接收文件路径字符串）
  │
  ▼
Video.postprocess() → _format_video()
  │  ├─ URL 直返？
  │  ├─ 下载 URL 到缓存
  │  ├─ 浏览器可播放性检查（ffprobe）
  │  ├─ 不可播放则转 mp4(h264)
  │  ├─ 按 format 转换（如有）
  │  └─ 水印（如有）
  │
  ▼
async_move_files_to_cache() ←─── postprocess 后生成 URL
  │  postprocess=True
  │  生成 /file=... URL
  ▼
前端接收 FileData(url=...)
  │
  ▼
<video src="/file=..."> 播放
```

### 9.2 Audio 输入输出完整链路

```
前端上传
  │
  ▼
POST /gradio_api/upload
  │  (multipart/form-data)
  ▼
upload_fn()  →  落盘到 uploaded_file_dir/{hash}/{filename}
  │
  ▼
返回 FileData(path=..., orig_name=...)
  │
  ▼
前端发起 predict 请求（携带 FileData）
  │
  ▼
async_move_files_to_cache()  ←─── preprocess 前的安全检查
  │  check_in_upload_folder=True
  ▼
Audio.preprocess()
  │  ├─ type=numpy: pydub 读取 → (sample_rate, np.array)
  │  └─ type=filepath: 按需转换格式 → 文件路径
  │
  ▼
用户函数（接收 tuple 或文件路径）
  │
  ▼
Audio.postprocess()
  │  ├─ bytes → save_bytes_to_cache（自动检测格式）
  │  ├─ tuple → save_audio_to_cache（format 指定或默认 wav）
  │  ├─ 文件路径 → 按需转换格式（format 指定时）
  │  └─ 无浏览器可播放性检查
  │
  ▼
async_move_files_to_cache() ←─── postprocess 后生成 URL
  │  postprocess=True
  │  生成 /file=... URL
  ▼
前端接收 FileData(url=...)
  │
  ▼
WaveSurfer.js / <audio src="/file=..."> 播放
```

---

## 10. 边界总结

### 10.1 前端 ↔ 后端边界

| 边界 | 传输格式 | 协议 |
|------|---------|------|
| 上传 | multipart/form-data | HTTP POST |
| 下载播放 | 原始媒体文件 | HTTP GET (Range 支持) |
| 流式输出 | HLS (m3u8 + ts/aac) | HTTP 长连接 |

### 10.2 落盘 ↔ 格式处理边界

| 阶段 | 文件位置 | 格式 | 责任方 |
|------|---------|------|--------|
| 上传后 | uploaded_file_dir/{hash}/ | 原始格式 | 上传路由 |
| move_files_to_cache 后 | block cache（同目录不同 hash 子目录） | 原始格式 | move_files_to_cache |
| preprocess 后 | block cache | 用户指定格式 / 原始格式 | 组件 preprocess |
| 用户函数中 | 用户代码决定 | 任意（路径或 numpy） | 用户函数 |
| postprocess 后 | block cache | 浏览器可播放格式（Video）/ 指定格式（Audio） | 组件 postprocess |

### 10.3 格式处理 ↔ 浏览器播放边界

| 介质 | 后端保证 | 前端播放方式 |
|------|---------|------------|
| Video | 一定是浏览器可播放格式（mp4+h264 等组合） | 两条路径：<br>• 普通文件：原生 `<video>`<br>• 流式：HLS.js + `<video>` |
| Audio | 按 format 参数透传或转换，不保证浏览器兼容 | 三条路径：<br>• WaveSurfer.js（波形模式，默认）<br>• 原生 `<audio>`（无波形）<br>• 流式：HLS.js + `<audio>` |

> **注意**：前端 FFmpeg WASM **仅用于 Video 裁剪**，Audio 前端不使用 FFmpeg，完全依赖浏览器原生解码能力。

### 10.4 FFmpeg 使用位置

| 位置 | 用途 | 实现方式 |
|------|------|---------|
| 后端 Video preprocess | 格式转换、翻转、去音轨 | ffmpy 封装系统 ffmpeg |
| 后端 Video postprocess | 浏览器兼容性转换、水印 | ffmpy 封装系统 ffmpeg |
| 后端 Video 可播放性检测 | ffprobe 检测容器和编码 | ffmpy 封装系统 ffprobe |
| 后端 Audio 处理 | 格式转换、读写文件 | pydub（底层 ffmpeg） |
| 后端 Video 流式 | mp4 → ts 转换 | ffmpy 封装系统 ffmpeg |
| 后端 Audio 流式 | 转 ADTS/AAC | pydub（底层 ffmpeg） |
| 前端 Video | 视频裁剪 | @ffmpeg/ffmpeg (WASM) |

---

## 11. 关键文件索引

| 文件 | 作用 |
|------|------|
| [gradio/components/video.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py) | Video 组件后端逻辑 |
| [gradio/components/audio.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py) | Audio 组件后端逻辑 |
| [gradio/processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py) | 媒体处理工具函数（转码、缓存等） |
| [gradio/route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/route_utils.py) | 上传函数、文件获取、MediaStream 类 |
| [gradio/routes.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/routes.py) | 上传/文件服务/流媒体路由 |
| [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/blocks.py) | move_resource_to_block_cache、流处理调度 |
| [gradio/components/base.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/base.py) | 组件基类，初始化时 move_files_to_cache |
| [gradio/_vendor/ffmpy/ffmpy.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/_vendor/ffmpy/ffmpy.py) | FFmpeg Python 封装 |
| [js/video/shared/Video.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Video.svelte) | Video 核心播放组件（HLS 支持） |
| [js/video/shared/Player.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Player.svelte) | Video 播放器交互组件 |
| [js/video/shared/VideoPreview.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/VideoPreview.svelte) | Video 静态预览组件 |
| [js/video/shared/InteractiveVideo.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/InteractiveVideo.svelte) | Video 前端交互组件 |
| [js/video/shared/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/utils.ts) | 前端 FFmpeg WASM 封装（裁剪用） |
| [js/audio/interactive/InteractiveAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/interactive/InteractiveAudio.svelte) | Audio 前端交互组件 |
| [js/audio/player/AudioPlayer.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/player/AudioPlayer.svelte) | Audio 播放器（WaveSurfer.js + HLS.js） |
| [js/audio/static/StaticAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/static/StaticAudio.svelte) | Audio 静态展示组件 |
| [js/upload/src/Upload.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/upload/src/Upload.svelte) | 通用上传组件 |
