# 客户端 SDK 与服务端预测协议配合说明

本文档梳理 Gradio 客户端 SDK 与服务端之间的预测协议配合机制，包括调用参数格式、任务提交流程和结果接收方式。

## 1. 核心架构概览

### 1.1 协议版本

Gradio 支持多种通信协议，在 [Config](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L402-L404) 中定义：

```typescript
protocol: "ws" | "sse" | "sse_v1" | "sse_v2" | "sse_v2.1" | "sse_v3"
```

- **ws**: WebSocket（已弃用）
- **sse**: 基础 Server-Sent Events
- **sse_v1**: SSE 第一版
- **sse_v2/v2.1**: 支持增量输出（diff stream），减少传输数据量
- **sse_v3**: 仅在后端发送关闭消息时才关闭流，更稳定

### 1.2 核心模块

| 模块 | 位置 | 职责 |
|------|------|------|
| Client 类 | [client.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/client.ts) | SDK 入口，管理连接、会话 |
| submit 函数 | [submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts) | 任务提交核心逻辑 |
| predict 函数 | [predict.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/predict.ts) | 基于 submit 的 Promise 封装 |
| Queue 类 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py) | 服务端队列管理 |
| API 路由 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py) | HTTP 接口定义 |

---

## 2. 调用参数详解

### 2.1 客户端入口

**Client.connect()** - 建立连接：

```typescript
static async connect(
    app_reference: string,           // URL 或 HF Space 名称
    options: ClientOptions = {
        events: ["data"],             // 订阅的事件类型
        token?: `hf_${string}`,       // HF token
        auth?: [string, string],      // 用户名密码
        session_hash?: string         // 自定义会话哈希
    }
): Promise<Client>
```

初始化流程 [client.ts:223-236](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/client.ts#L223-L236)：
1. 解析 endpoint，获取 host 和 protocol
2. 获取 config（包含组件、依赖、协议版本信息）
3. 连接心跳（如需）
4. 获取 API info（端点参数定义）
5. 建立 api_name -> fn_index 映射

### 2.2 预测调用参数

**predict() 方法** [predict.ts:4-51](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/predict.ts#L4-L51)：

```typescript
predict<T = unknown>(
    endpoint: string | number,    // API 名称或 fn_index
    data: unknown[] | Record<string, unknown> = {}  // 参数
): Promise<PredictReturn<T>>
```

**submit() 方法** [submit.ts:32-716](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L32-L716)：

```typescript
submit(
    endpoint: string | number,
    data: unknown[] | Record<string, unknown> = {},
    event_data?: unknown,
    trigger_id?: number | null,
    all_events?: boolean
): SubmitIterable<GradioEvent>
```

### 2.3 参数映射机制

**参数解析** [api_info.ts:429-480](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/helpers/api_info.ts#L429-L480)：

- **位置参数**：`data: [value1, value2]` 按顺序映射
- **命名参数**：`data: { param1: value1, param2: value2 }` 按键名映射
- 默认值填充：未提供的参数使用 `parameter_default`
- 验证：必填参数缺失时抛出错误

### 2.4 请求数据结构

**PredictBody** [data_classes.py:90-119](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L90-L119)：

```python
class PredictBody(BaseModel):
    session_hash: str | None = None      # 会话标识
    event_id: str | None = None          # 事件ID（由服务端生成）
    data: list[Any]                      # 输入数据数组
    event_data: Any | None = None        # 事件特定数据
    fn_index: int | None = None          # 函数索引
    trigger_id: int | None = None        # 触发器ID
    simple_format: bool = False
    batched: bool | None = False         # 是否批量请求
```

### 2.5 文件处理机制

**文件上传流程** [handle_blob.ts:16-62](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/handle_blob.ts#L16-L62)：

1. `walk_and_store_blobs()` - 递归遍历数据，收集 Blob/Buffer/File 对象
2. `upload_files()` - 上传文件到 `/upload` 端点，获取服务器文件路径
3. 替换原始 Blob 为 FileData 对象，包含 `path` 和 `url`

**FileData 结构** [data_classes.py:230-251](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L230-L251)：

```python
class FileData(GradioModel):
    path: str                    # 服务器文件路径
    url: str | None = None       # 访问URL
    size: int | None = None
    orig_name: str | None = None # 原始文件名
    mime_type: str | None = None
    is_stream: bool = False
    meta: FileDataMeta = {"_type": "gradio.FileData"}
```

---

## 3. 任务提交流程

### 3.1 两种提交模式

根据 `config.enable_queue` 和 `dependency.queue` 决定是否使用队列：

**模式1：跳过队列（直接调用）** [submit.ts:188-265](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L188-L265)：

```
客户端 POST /run/{endpoint}
    └─> 服务端直接执行函数
    └─> 同步返回结果
```

**模式2：队列模式（异步处理）** [submit.ts:266-631](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L266-L631)：

```
客户端 POST /queue/join
    └─> 服务端返回 event_id
    └─> 客户端建立 SSE 连接 GET /queue/data
    └─> 服务端通过 SSE 推送状态和结果
```

### 3.2 队列模式完整流程

#### 步骤1：提交任务

**客户端** [submit.ts:427-437](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L427-L437)：
```typescript
post_data(`${config.root}/queue/join`, {
    ...payload,      // PredictBody
    session_hash     // 客户端会话标识
})
```

**服务端路由** [routes.py:1357-1399](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1357-L1399)：
```python
@router.post("/queue/join")
async def queue_join(body: PredictBody, request, username):
    return await queue_join_helper(body, request, username)

async def queue_join_helper(body, request, username):
    body = PredictBodyInternal(**body.model_dump(), request=request)
    success, event_id, state = await blocks._queue.push(
        body=body, request=request, username=username
    )
    return {"event_id": event_id}
```

#### 步骤2：入队处理

**Queue.push()** [queueing.py:279-468](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L279-L468)：

1. 验证 fn_index 是否存在
2. 检查队列是否已满（max_size）
3. 执行验证器（如有）
4. 检查缓存命中（如有 cache 装饰器）
5. 创建 Event 对象，生成 event_id（uuid4）
6. 初始化会话消息队列 `pending_messages_per_session`
7. 事件加入 `event_queue_per_concurrency_id`
8. 广播队列位置估计

#### 步骤3：工作线程处理

**Queue.start_processing()** [queueing.py:521-563](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L521-L563)：

- 循环检查队列，取出待处理事件
- 支持批量处理（batch=True）
- 按 concurrency_id 管理并发限制
- 调用 `process_events()` 执行实际任务

**Queue.process_events()** [queueing.py:787-1079](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L787-L1079)：

1. 发送 `ProcessStartsMessage` 通知开始处理
2. 调用 `route_utils.call_process_api()` 执行函数
3. 对于生成器函数（is_generating=True）：
   - 循环调用获取中间结果
   - 发送 `ProcessGeneratingMessage` 推送中间结果
4. 发送 `ProcessCompletedMessage` 通知完成

---

## 4. 结果接收机制

### 4.1 SSE 连接建立

**客户端 open_stream()** [stream.ts:5-89](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L5-L89)：

```typescript
GET /queue/data?session_hash=xxx
Accept: text/event-stream
```

**服务端 queue_data_helper()** [routes.py:1473-1577](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1473-L1577)：

- 建立长连接，持续推送消息
- 内置心跳机制（默认 15 秒）
- 监听客户端断开，清理资源

### 4.2 消息类型与处理

**服务端消息** [server_messages.py:7-90](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/server_messages.py#L7-L90)：

| 消息类型 | 触发时机 | 包含字段 |
|---------|---------|---------|
| `estimation` | 入队后 | rank, queue_size, rank_eta |
| `process_starts` | 开始执行 | eta |
| `process_generating` | 生成器中间结果 | output, success, time_limit |
| `process_streaming` | 流式输出 | output, time_limit |
| `process_completed` | 执行完成 | output, success, used_cache |
| `progress` | 进度更新 | progress_data |
| `log` | 日志消息 | log, level, title |
| `heartbeat` | 保活 | - |
| `close_stream` | 关闭连接 | - |
| `unexpected_error` | 异常 | message |

**客户端消息处理** [api_info.ts:234-404](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/helpers/api_info.ts#L234-L404)：

`handle_message()` 函数将服务端消息转换为客户端事件：

```typescript
{
    type: "update" | "generating" | "streaming" | "complete" | "log" | "heartbeat" | ...,
    status?: Status,       // 状态信息
    data?: any             // 结果数据
}
```

### 4.3 客户端事件类型

**GradioEvent** 类型 [types.ts:358-435](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/types.ts#L358-L435)：

| 事件类型 | 说明 | 字段 |
|---------|------|------|
| `data` | 输出数据 | type: "data", data, fn_index, endpoint |
| `status` | 状态变化 | type: "status", stage, queue, eta, position |
| `log` | 日志消息 | type: "log", log, level, title |
| `render` | 动态渲染 | type: "render", data |

**Status.stage** 枚举：
- `pending`: 排队中
- `generating`: 生成中（生成器）
- `streaming`: 流式输出中
- `complete`: 完成
- `error`: 错误

### 4.4 增量更新（Diff Stream）

**sse_v2+ 协议优化** [stream.ts:101-119](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L101-L119)：

对于生成器函数的中间结果，服务端只发送增量 diff：

```python
# diff 格式
[action, path, value]
# action: "replace" | "append" | "add" | "delete"
# path: [key1, index1, ...] 定位路径
```

**apply_diff_stream()** 维护状态，累积增量更新：
- 首次生成：保存完整数据
- 后续生成：应用 diff 到已有数据

### 4.5 predict() 的 Promise 封装

[predict.ts:26-50](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/predict.ts#L26-L50)：

```typescript
return new Promise(async (resolve, reject) => {
    const app = this.submit(endpoint, data, null, null, true);
    let result: unknown;
    let data_returned = false;
    let status_complete = false;

    for await (const message of app) {
        if (message.type === "data") {
            data_returned = true;
            result = message;
            if (status_complete) resolve(result);
        }
        if (message.type === "status") {
            if (message.stage === "error") reject(message);
            if (message.stage === "complete") {
                status_complete = true;
                if (data_returned) resolve(result);
            }
        }
    }
});
```

---

## 5. 完整时序图

### 5.1 队列模式预测流程

```
客户端 (Client)                          服务端 (Server)
     |                                         |
     | 1. POST /queue/join                    |
     |    { session_hash, fn_index, data }    |
     |---------------------------------------->|
     |                                         |
     | 2. 返回 event_id                       |
     |    { event_id: "abc123" }              |
     |<----------------------------------------|
     |                                         |
     | 3. GET /queue/data?session_hash=xxx    |
     |    (SSE 长连接)                        |
     |---------------------------------------->|
     |                                         |
     | 4. 推送 estimation                     |
     |    msg: "estimation"                   |
     |<----------------------------------------|
     |                                         |
     | 5. 推送 process_starts                 |
     |    msg: "process_starts"               |
     |<----------------------------------------|
     |                                         |
     | 6. 推送 process_generating (可选)      |
     |    msg: "process_generating"           |
     |<----------------------------------------|
     |                                         |
     | 7. 推送 process_completed              |
     |    msg: "process_completed"            |
     |    output: { data: [...] }             |
     |<----------------------------------------|
     |                                         |
     | 8. 推送 close_stream                   |
     |    msg: "close_stream"                 |
     |<----------------------------------------|
```

### 5.2 取消任务流程

```
客户端                                     服务端
     |                                         |
     | 1. POST /cancel                         |
     |    { event_id, session_hash, fn_index } |
     |---------------------------------------->|
     |                                         |
     | 2. 从队列移除或取消执行                 |
     | 3. 发送 ProcessCompletedMessage         |
     |    (success=True, output={})            |
     |<----------------------------------------|
     | 4. POST /reset                          |
     |    { event_id }                         |
     |---------------------------------------->|
```

---

## 6. 关键端点汇总

| 端点 | 方法 | 用途 |
|------|------|------|
| `/config` | GET | 获取应用配置 |
| `/info` | GET | 获取 API 信息（端点参数） |
| `/upload` | POST | 上传文件 |
| `/run/{endpoint}` | POST | 非队列模式直接执行 |
| `/queue/join` | POST | 队列模式提交任务 |
| `/queue/data` | GET | SSE 接收结果 |
| `/cancel` | POST | 取消任务 |
| `/reset` | POST | 重置迭代器状态 |
| `/heartbeat/{session_hash}` | GET | 心跳保活 |
| `/component_server` | POST | 组件服务端方法调用 |

---

## 7. 错误处理

### 7.1 客户端错误

| 场景 | 处理方式 |
|------|---------|
| 队列已满（503） | 发送 status stage="error"，message="Queue is full" |
| 验证失败（422） | 发送 status stage="error"，code="validation_error" |
| 连接断开 | 发送 status stage="error"，broken=true |
| 会话未找到 | 发送 status stage="error"，session_not_found=true |

### 7.2 服务端错误处理

- 业务异常（`gr.Error`）：包装为 `ProcessCompletedMessage(success=False)`
- 未预期异常：发送 `UnexpectedErrorMessage`
- 队列停止：返回 503 `Queue is stopped`

---

## 8. 核心代码参考

### 8.1 客户端提交逻辑入口

[submit.ts:67-74](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L67-L74)
```typescript
let { fn_index, endpoint_info, dependency } = get_endpoint_info(
    api_info, endpoint, api_map, config
);
let resolved_data = map_data_to_params(data, endpoint_info);
```

### 8.2 服务端入队处理

[queueing.py:373-388](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L373-L388)
```python
event = Event(body.session_hash, fn, request, username)
event.data = body
self.pending_event_ids_session[body.session_hash].add(event._id)
self.event_ids_to_events[event._id] = event
event_queue.queue.append(event)
```

### 8.3 消息分发回调

[submit.ts:619-628](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L619-L628)
```typescript
if (event_id in pending_stream_messages) {
    pending_stream_messages[event_id].forEach((msg) => callback(msg));
    delete pending_stream_messages[event_id];
}
event_callbacks[event_id] = callback;
if (!stream_status.open) {
    await this.open_stream();
}
```
