# Video & Audio 转码流程梳理

本文档梳理 Gradio 中 Video 和 Audio 组件的媒体转码全流程，明确**上传落盘、格式处理、浏览器播放**三者之间的边界和数据流转。

---

## 1. 整体架构概览

```
┌──────────────────────────────────────────────────────────────────────┐
│                         前端 (浏览器)                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │ Upload 组件  │───▶│  FFmpeg WASM │───▶│  <video>/<audio> 播放 │  │
│  └──────────────┘    └──────────────┘    └──────────────────────┘  │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ HTTP (multipart/form-data)
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         后端 (Python)                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │  upload 路由  │───▶│ preprocess   │───▶│  用户预测函数        │  │
│  └──────────────┘    └──────────────┘    └──────────┬───────────┘  │
│                                                     │                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────▼───────────┐  │
│  │  file 路由   │◀───│ postprocess  │◀───│  move_files_to_cache │  │
│  └──────────────┘    └──────────────┘    └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

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

**落盘位置**：`app.uploaded_file_dir`（上传文件暂存目录）

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

## 3. 格式处理（转码）

格式处理发生在两个阶段：**preprocess**（输入处理）和 **postprocess**（输出处理）。

### 3.1 Video 格式处理

**核心文件**：[video.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py)

#### 3.1.1 Preprocess（输入 → 用户函数）

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

#### 3.1.2 Postprocess（用户函数 → 输出）

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

### 3.2 Audio 格式处理

**核心文件**：[audio.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py)

#### 3.2.1 Preprocess（输入 → 用户函数）

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

#### 3.2.2 Postprocess（用户函数 → 输出）

[audio.py postprocess](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L269-L319)

**支持的输入格式**：

| 输入类型 | 处理方式 |
|---------|---------|
| `bytes` | 保存到缓存，自动检测格式 |
| `(sample_rate, data)` tuple | 用 `save_audio_to_cache()` 保存为指定格式（默认 wav） |
| `str` / `Path` 文件路径 | 如需格式转换则转换，否则直接使用 |
| `None` | 返回 None |

**格式转换函数**：
- [audio_from_file()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L670-L697)：文件 → numpy
- [audio_to_file()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L700-L717)：numpy → 文件
- [convert_to_16_bit_audio()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L720-L757)：统一转为 16-bit 整型

**返回值**：`FileData` 对象

---

## 4. 文件缓存与 URL 生成

### 4.1 move_files_to_cache

**位置**：[processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py#L431-L502)

**调用时机**：
- preprocess 之前（输入数据）
- postprocess 之后（输出数据）
- 组件初始化时（初始值）

**核心作用**：
1. 将文件从上传目录/用户路径移动到 block cache
2. 生成可访问的 URL（`/file=...` 或 `/stream/...`）
3. 安全检查（SSRF 防护、路径白名单）

**URL 生成规则**：
```python
# 普通文件
url = f"{API_PREFIX}/file={payload.path}"

# 流式文件
url = f"{API_PREFIX}/stream/" + ...
```

### 4.2 缓存目录

| 目录 | 用途 | 生命周期 |
|------|------|---------|
| `uploaded_file_dir` | 上传文件暂存 | 请求期间 |
| `GRADIO_CACHE` (block cache) | 处理后的媒体文件 | 组件生命周期 |
| `tempfile.gettempdir()/gradio` | 临时文件 | 进程退出后清理 |

---

## 5. 浏览器播放

### 5.1 文件服务路由

**路由**：[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/routes.py#L1082-L1086)
```
GET /gradio_api/file={path}
HEAD /gradio_api/file={path}
```

**处理函数**：[file_fetch()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/route_utils.py#L1190-L1249)

**安全机制**：
- 路径白名单检查
- 禁止目录遍历
- SSRF 防护（外部 URL 走代理）
- 范围请求支持（Range 头）

**MIME 类型处理**：
- 安全 MIME 类型（视频/音频/图片等）→ `inline` 内联播放
- 其他类型 → `attachment` 附件下载

### 5.2 Video 播放

**播放组件**：
- 静态模式：[VideoPreview.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/VideoPreview.svelte)
- 交互模式：[Player.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/Player.svelte)

**播放方式**：HTML5 `<video>` 标签

**支持的浏览器可播放格式**（后端保证）：
- MP4 + H.264 / AV1
- WebM + VP9 / VP8 / AV1
- Ogg + Theora

**前端 FFmpeg（WASM）**：
- 位置：[js/video/shared/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/utils.ts)
- 用途：视频裁剪（trimVideo 函数）
- 库：`@ffmpeg/ffmpeg` + `@ffmpeg/util`
- 加载方式：从 `/static/ffmpeg/` 加载 WASM 核心

### 5.3 Audio 播放

**播放组件**：
- 静态模式：[StaticAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/static/StaticAudio.svelte)
- 播放器：[AudioPlayer.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/player/AudioPlayer.svelte)

**播放方式**：HTML5 `<audio>` 标签 + 波形可视化

**支持格式**：浏览器原生支持的音频格式（wav, mp3, ogg, flac 等）

---

## 6. 流媒体（Streaming）

### 6.1 Video 流媒体

**输出流媒体**：
- 格式：`.ts` (MPEG-TS) + H.264 编码
- 传输：HLS (m3u8 playlist)
- 路由：`/stream/{session_hash}/{run}/{component_id}/playlist.m3u8`

**转换函数**：[async_convert_mp4_to_ts()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py#L502-L524)

**流合并**：[combine_stream()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py#L526-L581)
- 将多个 .ts 片段用 ffmpeg concat 合并为一个 mp4

### 6.2 Audio 流媒体

**输出流媒体**：
- 格式：`.aac` (ADTS 容器)
- 转换：[covert_to_adts()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L331-L332)

**流合并**：[combine_stream()](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py#L366-L385)
- 直接拼接字节，保存为 mp3
- 可指定输出格式转换

---

## 7. 边界总结

### 7.1 前端 ↔ 后端边界

| 边界 | 传输格式 | 协议 |
|------|---------|------|
| 上传 | multipart/form-data | HTTP POST |
| 下载播放 | 原始媒体文件 | HTTP GET (Range 支持) |
| 流式输出 | HLS / SSE | HTTP 长连接 |

### 7.2 落盘 ↔ 格式处理边界

| 阶段 | 文件位置 | 格式 | 责任方 |
|------|---------|------|--------|
| 上传后 | uploaded_file_dir | 原始格式 | 上传路由 |
| preprocess 后 | block cache | 用户指定格式 / 原始格式 | 组件 preprocess |
| 用户函数中 | 用户代码决定 | 任意（路径或 numpy） | 用户函数 |
| postprocess 后 | block cache | 浏览器可播放格式 | 组件 postprocess |

### 7.3 格式处理 ↔ 浏览器播放边界

| 介质 | 后端保证的播放格式 | 前端播放方式 |
|------|------------------|------------|
| Video | mp4(h264/av1), webm(vp9/vp8/av1), ogg(theora) | HTML5 `<video>` |
| Audio | wav, mp3 等浏览器支持的格式 | HTML5 `<audio>` |

### 7.4 FFmpeg 使用位置

| 位置 | 用途 | 实现方式 |
|------|------|---------|
| 后端 Video preprocess | 格式转换、翻转、去音轨 | ffmpy 封装系统 ffmpeg |
| 后端 Video postprocess | 浏览器兼容性转换、水印 | ffmpy 封装系统 ffmpeg |
| 后端 Audio 处理 | 格式转换、读写文件 | pydub（底层 ffmpeg） |
| 后端 Video 流式 | mp4 → ts 转换 | ffmpy 封装系统 ffmpeg |
| 前端 Video | 视频裁剪 | @ffmpeg/ffmpeg (WASM) |

---

## 8. 关键文件索引

| 文件 | 作用 |
|------|------|
| [gradio/components/video.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/video.py) | Video 组件后端逻辑 |
| [gradio/components/audio.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/components/audio.py) | Audio 组件后端逻辑 |
| [gradio/processing_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/processing_utils.py) | 媒体处理工具函数 |
| [gradio/_vendor/ffmpy/ffmpy.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/_vendor/ffmpy/ffmpy.py) | FFmpeg Python 封装 |
| [gradio/routes.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/routes.py) | 上传/文件服务路由 |
| [gradio/route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/gradio/route_utils.py) | 文件获取安全检查 |
| [js/video/shared/InteractiveVideo.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/InteractiveVideo.svelte) | Video 前端交互组件 |
| [js/video/shared/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/video/shared/utils.ts) | 前端 FFmpeg WASM 封装 |
| [js/audio/interactive/InteractiveAudio.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/audio/interactive/InteractiveAudio.svelte) | Audio 前端交互组件 |
| [js/upload/src/Upload.svelte](file:///d:/fz/0601/solo-dogfeeding/code/252-gradio/js/upload/src/Upload.svelte) | 通用上传组件 |
