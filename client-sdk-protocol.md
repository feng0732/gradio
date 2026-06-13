# 客户端 SDK 与服务端预测协议配合说明

本文档梳理 Gradio 客户端 SDK 与服务端之间的预测协议配合机制，包括调用参数格式、任务提交流程、结果接收方式，以及各版本 SSE 协议的差异。

---

## 1. 核心架构概览

### 1.1 协议版本

Gradio 支持多种通信协议，在 [Config](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L402-L404) 中定义：

```typescript
protocol: "ws" | "sse" | "sse_v1" | "sse_v2" | "sse_v2.1" | "sse_v3"
```

各版本演进历史（从旧到新）：

| 版本 | 核心变化 | 连接模式 |
|------|---------|---------|
| **ws** | WebSocket 协议（已弃用） | 双向长连接 |
| **sse** | 初代 SSE，每次请求独立建连 | 一请求一连接 |
| **sse_v1** | 引入 event_id，会话级连接复用 | 一会话一连接 |
| **sse_v2** | 增加增量 diff 输出，减小传输量 | 一会话一连接 |
| **sse_v2.1** | v2 的小幅优化 | 一会话一连接 |
| **sse_v3** | 服务端控制流关闭时机，更稳定 | 一会话一连接 |

### 1.2 核心模块

| 模块 | 位置 | 职责 |
|------|------|------|
| Client 类 | [client.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/client.ts) | SDK 入口，管理连接、会话、流状态 |
| submit 函数 | [submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts) | 任务提交核心，区分协议版本处理 |
| predict 函数 | [predict.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/predict.ts) | 基于 submit 的 Promise 封装 |
| open_stream | [stream.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts) | 建立会话级 SSE 长连接，消息分发 |
| Queue 类 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py) | 服务端队列管理，消息推送 |
| API 路由 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py) | HTTP 接口定义，SSE 响应 |

### 1.3 核心数据结构

**客户端会话状态**（Client 类成员）：

```typescript
session_hash: string           // 客户端会话标识，随机生成
stream_status: { open: boolean } // SSE 流是否打开
event_callbacks: Record<event_id, callback> // 事件ID -> 回调函数
pending_stream_messages: Record<event_id, msg[]> // 早到消息缓存
unclosed_events: Set<event_id> // 未关闭的事件集合
pending_diff_streams: Record<event_id, data[]> // diff 流的累积状态
```

**服务端会话状态**（Queue 类成员）：

```python
pending_messages_per_session: LRUCache[session_hash, AsyncQueue]  # 每个会话的消息队列
pending_event_ids_session: dict[session_hash, set[event_id]]     # 会话内未完成事件
event_ids_to_events: dict[event_id, Event]                        # event_id -> Event 对象
```

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
3. 连接心跳（如需 state 或 unload 事件）
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

## 3. 任务提交：各协议版本对比

### 3.1 总览：各版本提交与收尾方式

| 协议版本 | 提交方式 | SSE 连接模型 | 流建立时机 | 流关闭时机 |
|---------|---------|-------------|-----------|-----------|
| **非队列** | `POST /run/{api}` | 无 SSE | - | 请求结束即结束 |
| **sse** | `GET /queue/data?fn_index=...&session_hash=...` | 一请求一连接 | 提交时建立 | 数据+complete 都到达后客户端关闭 |
| **sse_v1** | `POST /queue/join` + `GET /queue/data?session_hash=...` | 一会话一连接 | 首次请求时建立 | 会话内所有事件完成后服务端发 close_stream |
| **sse_v2** | 同 sse_v1 | 一会话一连接 | 同 sse_v1 | 同 sse_v1 + 支持 diff |
| **sse_v2.1** | 同 sse_v1 | 一会话一连接 | 同 sse_v1 | 同 sse_v2 |
| **sse_v3** | 同 sse_v1 | 一会话一连接 | 同 sse_v1 | **仅服务端发送 close_stream 才关闭** |

### 3.2 非队列模式（直接调用）

**适用条件**：`config.enable_queue = false` 或 `dependency.queue = false`

**提交** [submit.ts:188-265](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L188-L265)：

```
客户端 POST /run/{endpoint}
   body: { data, session_hash, fn_index, event_data, trigger_id }
   └─> 服务端直接执行函数 call_process_api()
   └─> 同步返回 { data: [...], average_duration, ... }
```

**服务端路由** [routes.py:1269-1316](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1269-L1316)：

```python
@router.post("/run/{api_name}")
async def predict(api_name, body, request, username):
    fn = route_utils.get_fn(...)
    output = await route_utils.call_process_api(...)
    return ORJSONResponse(output)
```

**收尾**：
- 一次性请求，响应返回即结束
- 无状态维护，无 SSE 连接
- 不支持生成器函数的中间输出

### 3.3 旧版 SSE（sse）

**提交方式** [submit.ts:266-394](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L266-L394)：

直接以 SSE 方式连接 `/queue/data`，同时携带 `fn_index` 和 `session_hash` 参数：

```typescript
// sse 版本：一次请求对应一条 SSE 连接
let url = new URL(
    `${config.root}/${SSE_URL}?fn_index=${fn_index}&session_hash=${session_hash}`
);
stream = this.stream(url);  // 为每个请求单独建立 EventSource
```

**服务端处理**：
- SSE 连接建立后，服务端自动将任务入队
- 相当于把 "提交" 和 "接收" 合在一个 SSE 连接里完成

**消息交互**：
1. 连接建立后，服务端先推送 `send_hash` / `estimation` 等状态消息
2. 当收到 `send_data` 消息时，客户端需要额外 `POST /queue/data` 提交实际数据
3. 然后继续在同一条 SSE 流上接收结果

```
客户端                                服务端
   |                                     |
   |--- GET /queue/data?fn_index=X&session_hash=Y -->|
   |                                     |
   |<--- msg: "send_hash" ---------------|   (让客户端发送 session_hash)
   |                                     |
   |<--- msg: "estimation" --------------|   (队列位置估计)
   |                                     |
   |<--- msg: "send_data" ---------------|   (要求客户端发送数据)
   |                                     |
   |--- POST /queue/data { data, event_id } -->|  (提交实际输入数据)
   |                                     |
   |<--- msg: "process_starts" ----------|
   |<--- msg: "process_generating" ------|  (可选，生成器)
   |<--- msg: "process_completed" -------|
   |                                     |
   | 客户端检测到 complete，关闭流       |
   |--- EventSource.close() ------------|
```

**收尾机制**：
- 客户端收到 `process_completed` 且 data 也到达后，主动调用 `stream.close()`
- 每条请求独立关闭，不影响其他请求
- **缺点**：并发多个请求需要建立多条 SSE 连接，资源开销大

### 3.4 SSE v1：会话级连接复用

**核心变化** [submit.ts:395-631](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L395-L631)：

1. 提交和接收分离：先 POST 提交拿 event_id，再通过共享 SSE 流接收
2. 同一个 session_hash 共用一条 SSE 连接
3. 每条消息带 event_id，客户端按 event_id 分发到对应回调

**提交流程**：

```
客户端                                服务端
   |                                     |
   |--- POST /queue/join ---------------->|
   |    { data, session_hash, fn_index }  |
   |                                     |
   |<--- { event_id: "abc123" } ---------|   返回事件ID
   |                                     |
   |  [如果 SSE 流未打开]                 |
   |--- GET /queue/data?session_hash=Y ->|   建立会话级长连接
   |                                     |
   |<--- msg: "estimation"  (event_id: "abc123")
   |<--- msg: "process_starts" (event_id: "abc123")
   |<--- msg: "process_completed" (event_id: "abc123")
```

**提交代码** [submit.ts:427-436](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L427-L436)：

```typescript
// sse_v1+：先 POST 提交数据，拿到 event_id
post_data(`${config.root}/${SSE_DATA_URL}?${url_params}`, {
    ...payload,       // { data, fn_index, event_data, ... }
    session_hash
})
// 返回 { event_id: "xxx" }
```

> 注意：常量名容易混淆。`SSE_DATA_URL = "queue/join"` 是**提交数据**的端点；`SSE_URL = "queue/data"` 是**SSE 接收**的端点。

**服务端提交路由** [routes.py:1357-1399](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1357-L1399)：

```python
@router.post("/queue/join")
async def queue_join(body, request, username):
    body = PredictBodyInternal(**body.model_dump(), request=request)
    success, event_id, state = await blocks._queue.push(body, request, username)
    return {"event_id": event_id}
```

**入队处理** [queueing.py:279-468](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L279-L468)：

1. 验证 fn_index
2. 检查队列是否已满
3. 执行验证器（如有）
4. 检查缓存命中（如有 cache 装饰器）
5. 创建 Event 对象，生成 event_id（uuid4）
6. 初始化会话消息队列 `pending_messages_per_session[session_hash]`
7. 事件加入 `event_queue_per_concurrency_id` 等待调度
8. 广播队列位置估计（`EstimationMessage`）

### 3.5 SSE v2/v2.1：增量 Diff 输出

**核心变化**：生成器函数的中间结果只发送增量 diff，减少数据传输量。

**Diff 格式** [stream.ts:121-178](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L121-L178)：

```typescript
// diff 是一组编辑操作
[action, path, value][]

// action: "replace" | "append" | "add" | "delete"
// path: [key1, index1, ...] 嵌套定位路径
// value: 新值

// 示例：给 output[0].text 追加内容
[["append", [0, "text"], " more content"]]

// 示例：替换数组第3项
[["replace", [2], {"name": "new", "value": 42}]]
```

**客户端累积** [stream.ts:101-119](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L101-L119)：

```typescript
// apply_diff_stream 为每个 event_id 维护完整状态
function apply_diff_stream(pending_diff_streams, event_id, data) {
    if (!pending_diff_streams[event_id]) {
        // 首次：保存完整数据
        pending_diff_streams[event_id] = data.data;
    } else {
        // 后续：应用 diff 到已有数据
        data.data.forEach((value, i) => {
            let new_data = apply_diff(pending_diff_streams[event_id][i], value);
            pending_diff_streams[event_id][i] = new_data;
            data.data[i] = new_data;  // 替换为完整数据供上层使用
        });
    }
}
```

**适用条件** [submit.ts:551-557](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L551-L557)：

```typescript
if (
    data &&
    dependency.connection !== "stream" &&  // 非 stream 连接
    ["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)
) {
    apply_diff_stream(pending_diff_streams, event_id!, data);
}
```

### 3.6 SSE v3：服务端控制流关闭

**核心变化**：只有当服务端发送 `close_stream` 消息时，客户端才关闭 SSE 连接。

**之前版本的问题**：
- sse/sse_v1/sse_v2 中，单个请求完成后客户端可能考虑关闭流
- 多请求并发时，流的关闭时机复杂，容易导致连接异常断开

**sse_v3 的改进**：
- 服务端维护会话内所有未完成事件
- 所有事件都完成后，服务端主动发送 `close_stream`
- 客户端收到后关闭连接

**服务端逻辑** [routes.py:1526-1557](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1526-L1557)：

```python
if isinstance(message, ProcessCompletedMessage) and message.event_id:
    # 从 pending 中移除该 event_id
    blocks._queue.pending_event_ids_session[session_hash].remove(message.event_id)
    
    # 如果会话内没有未完成事件了，或服务端停止了
    if message.msg == ServerMessage.server_stopped or (
        message.msg == ServerMessage.process_completed
        and len(blocks._queue.pending_event_ids_session[session_hash]) == 0
    ):
        # 发送 close_stream 消息
        message = CloseStreamMessage()
        yield process_msg(message)
        return  # 关闭 SSE 连接
```

**客户端处理** [stream.ts:41-46](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L41-L46)：

```typescript
stream.onmessage = async function (event) {
    let _data = JSON.parse(event.data);
    if (_data.msg === "close_stream") {
        close_stream(stream_status, that.abort_controller);
        return;  // 直接关闭，不交给回调
    }
    // ... 其他消息分发
};
```

---

## 4. 结果接收：会话级长连接与消息分发

### 4.1 会话级 SSE 连接的建立

**open_stream()** [stream.ts:5-89](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L5-L89) 是 sse_v1+ 共用的会话级长连接入口：

```typescript
export async function open_stream(this: Client): Promise<void> {
    stream_status.open = true;  // 标记流已打开
    
    // 只带 session_hash，不带 fn_index
    let url = new URL(`${config.root}/${SSE_URL}?session_hash=${this.session_hash}`);
    stream = this.stream(url);
    
    // 统一的 onmessage 处理器
    stream.onmessage = function (event) {
        let _data = JSON.parse(event.data);
        // 分发逻辑...
    };
}
```

**连接时机** [submit.ts:626-628](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L626-L628)：

```typescript
// 注册回调时检查，如果流未打开则建立
if (!stream_status.open) {
    await this.open_stream();
}
```

**连接复用的关键数据**（都挂载在 Client 实例上）：

| 成员变量 | 类型 | 作用 |
|---------|------|------|
| `stream_status` | `{ open: boolean }` | 标记 SSE 流是否已打开 |
| `event_callbacks` | `Record<event_id, callback>` | 事件ID 到回调函数的映射 |
| `pending_stream_messages` | `Record<event_id, msg[]>` | 早到消息的缓存 |
| `unclosed_events` | `Set<event_id>` | 未完成的事件集合 |
| `abort_controller` | `AbortController` | 用于中止 fetch 请求 |

### 4.2 消息分发机制

**核心问题**：同一条 SSE 流上会传来多个事件的消息，怎么知道哪条消息属于哪个请求？

**答案**：每条消息都带 `event_id` 字段，客户端用 `event_callbacks` 表分发。

**分发逻辑** [stream.ts:41-77](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L41-L77)：

```typescript
stream.onmessage = async function (event: MessageEvent) {
    let _data = JSON.parse(event.data);
    
    // 1. 特殊消息：close_stream 直接处理
    if (_data.msg === "close_stream") {
        close_stream(stream_status, that.abort_controller);
        return;
    }
    
    const event_id = _data.event_id;
    
    if (!event_id) {
        // 2. 无 event_id 的消息：广播给所有回调
        // 例如某些全局通知（注意：实际中 heartbeat 是怎么发的？）
        await Promise.all(
            Object.keys(event_callbacks).map(eid => event_callbacks[eid](_data))
        );
    } else if (event_callbacks[event_id]) {
        // 3. 有回调：直接调用
        let fn = event_callbacks[event_id];
        // 浏览器环境下用 setTimeout 避免阻塞 UI
        setTimeout(fn, 0, _data);
    } else {
        // 4. 无回调：缓存起来（竞态处理）
        if (!pending_stream_messages[event_id]) {
            pending_stream_messages[event_id] = [];
        }
        pending_stream_messages[event_id].push(_data);
    }
};
```

### 4.3 竞态处理：消息早于回调

**为什么会有竞态？**

提交任务是 POST 请求，建立 SSE 连接是 GET 请求。
如果 POST 很快返回，但 SSE 连接还没建好（或者回调还没注册），
服务端的消息就可能先到了。

**怎么解决？** 用 `pending_stream_messages` 做缓存。

**回调注册时补消费** [submit.ts:619-625](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L619-L625)：

```typescript
// 注册回调前，先检查有没有早到的消息
if (event_id in pending_stream_messages) {
    // 有缓存：逐条喂给回调
    pending_stream_messages[event_id].forEach((msg) => callback(msg));
    // 清掉缓存
    delete pending_stream_messages[event_id];
}
// 注册回调
event_callbacks[event_id] = callback;
unclosed_events.add(event_id);
```

**时序图解**：

```
客户端                                服务端
   |                                     |
   |-- POST /queue/join -->|             |
   |                      |              |
   |                [服务端处理入队]     |
   |                      |              |
   |<-- {event_id: "a"} --|              |  (POST 响应返回)
   |                                     |
   |  [注册 callback["a"] = fn]          |
   |  [发现流未开，开始建连接]            |
   |                                     |
   |-- GET /queue/data?session_hash=Y ->|
   |                                     |
   |   [此时服务端可能已经发了几条消息]    |
   |                                     |
   |<-- msg1 (event_id: "a") -----------|
   |<-- msg2 (event_id: "a") -----------|
   |     这两条消息进 pending_stream_messages["a"]
   |                                     |
   |  [SSE 连接建好，onmessage 开始工作]   |
   |  [同时检查 pending，发现有缓存]       |
   |  [立即用 msg1、msg2 调用 callback]   |
   |                                     |
   |<-- msg3 (event_id: "a") -----------|  后续消息直接走 callback
```

### 4.4 单请求回调的内部处理

每个 submit() 调用都会创建一个专属的 callback 函数 [submit.ts:489-617](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L489-L617)：

```typescript
let callback = async function (_data: object): Promise<void> {
    const { type, status, data, original_msg } = handle_message(_data, last_status[fn_index]);
    
    if (type == "heartbeat") return;  // 心跳直接忽略
    
    if (type === "update" && status && !complete) {
        fire_event({ type: "status", ...status });  // 状态更新
    } else if (type === "complete") {
        complete = status;  // 标记完成
    } else if (type === "generating" || type === "streaming") {
        fire_event({ type: "status", stage: status.stage, ... });
        if (sse_v2+) {
            apply_diff_stream(pending_diff_streams, event_id, data);  // 应用 diff
        }
    }
    
    if (data) {
        fire_event({ type: "data", data: handle_payload(...) });  // 数据事件
        if (complete) {
            fire_event({ type: "status", stage: "complete", ... });
            close();  // 本请求的迭代器结束
        }
    }
    
    if (status?.stage === "complete" || status?.stage === "error") {
        delete event_callbacks[event_id];  // 清理回调
        delete pending_diff_streams[event_id];  // 清理 diff 状态
        close();
    }
};
```

### 4.5 消息类型一览

**服务端消息类型** [server_messages.py:7-90](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/server_messages.py#L7-L90)：

| 消息 msg 字段 | 触发时机 | 对应客户端 type |
|-------------|---------|----------------|
| `estimation` | 入队后，定期更新队列位置 | `update` (stage: pending) |
| `process_starts` | 任务开始执行 | `update` (stage: pending) |
| `process_generating` | 生成器中间结果 | `generating` + 可选 data |
| `process_streaming` | 流式输入输出 | `streaming` + 可选 data |
| `process_completed` | 执行完成 | `complete` + data |
| `progress` | 进度更新 | `update` |
| `log` | 日志消息 | `log` |
| `heartbeat` | 定期保活 | `heartbeat` (忽略) |
| `close_stream` | 服务端要求关闭流 | 直接关闭，不进回调 |
| `unexpected_error` | 未预期异常 | `unexpected_error` |
| `broken_connection` | 连接断开 | `broken_connection` |

**客户端事件类型** [types.ts:358-435](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/types.ts#L358-L435)：

| 事件 type | 说明 | 触发条件 |
|----------|------|---------|
| `data` | 输出数据 | process_generating / process_completed 带 output 时 |
| `status` | 状态变化 | estimation / process_starts / complete / error 等 |
| `log` | 日志消息 | log 消息 |
| `render` | 动态渲染 | 服务端返回 render_config 时 |

**Status.stage 枚举**：
- `pending`: 排队中 / 刚开始处理
- `generating`: 生成器输出中
- `streaming`: 流式输出中
- `complete`: 完成
- `error`: 错误

---

## 5. 服务端消息推送机制

### 5.1 消息如何从服务端发到 SSE

**服务端数据流**：

```
工作线程 process_events()
   └─> Queue.send_message(event, message)
          └─> message.event_id = event._id
          └─> pending_messages_per_session[session_hash].put_nowait(message)
                      │
                      ▼
          queue_data_helper() SSE 循环
              └─> 从 AsyncQueue 取消息
              └─> 格式化为 SSE data 帧
              └─> yield 给客户端
```

**send_message()** [queueing.py:240-249](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L240-L249)：

```python
def send_message(self, event, event_message):
    if not event.alive:
        return
    event_message.event_id = event._id  # 打上 event_id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)  # 放入会话消息队列
```

**SSE 推送循环** [routes.py:1490-1572](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1490-L1572)：

```python
async def sse_stream(request):
    heartbeat_task = asyncio.create_task(heartbeat())
    try:
        while True:
            if await request.is_disconnected():
                await blocks._queue.clean_events(session_hash=session_hash)
                return
            
            # 从队列取消息，超时 10 秒
            message = await asyncio.wait_for(
                pending_messages_per_session[session_hash].get(),
                timeout=10
            )
            
            if blocks._queue.stopped:
                message = UnexpectedErrorMessage(message="Server stopped unexpectedly.")
            
            if message:
                response = process_msg(message)
                if response is not None:
                    yield response
                
                # 如果是 process_completed，检查是否所有事件都完成了
                if isinstance(message, ProcessCompletedMessage) and message.event_id:
                    pending_event_ids_session[session_hash].remove(message.event_id)
                    
                    # 所有事件完成 → 发 close_stream
                    if len(pending_event_ids_session[session_hash]) == 0:
                        yield process_msg(CloseStreamMessage())
                        return
    except BaseException as e:
        # 异常处理
        ...
```

### 5.2 心跳机制

**服务端心跳** [routes.py:1481-1488](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1481-L1488)：

```python
async def heartbeat():
    while blocks.is_running:
        await asyncio.sleep(heartbeat_rate)  # 默认 15 秒
        queue = blocks._queue.pending_messages_per_session.get(session_hash)
        if queue:
            await queue.put(HeartbeatMessage())
```

**客户端处理** [submit.ts:496-498](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L496-L498)：

```typescript
if (type == "heartbeat") {
    return;  // 直接忽略，不向上层传递
}
```

> 注意：心跳消息没有 `event_id`，所以会广播给所有回调，但每个回调都直接 return 忽略。

### 5.3 广播 vs 单播

| 消息类型 | 是否带 event_id | 分发方式 |
|---------|----------------|---------|
| estimation | ✅ 是 | 按 event_id 单播 |
| process_starts | ✅ 是 | 按 event_id 单播 |
| process_generating | ✅ 是 | 按 event_id 单播 |
| process_completed | ✅ 是 | 按 event_id 单播 |
| progress | ✅ 是 | 按 event_id 单播 |
| log | ✅ 是 | 按 event_id 单播 |
| heartbeat | ❌ 否 | 广播给所有回调（被忽略） |
| close_stream | ❌ 否 | 全局处理，关闭连接 |
| unexpected_error | 不一定 | 看情况 |
| broken_connection | 不一定 | 看情况 |

---

## 6. 完整时序图

### 6.1 sse_v1+ 完整流程（会话级连接）

```
客户端 (Client)                          服务端 (Server)
     |                                         |
     |  第1个请求 submit()                     |
     |  POST /queue/join { session_hash, fn_index, data }
     |---------------------------------------->|
     |                                         |
     |  { event_id: "evt_a" }                  |
     |<----------------------------------------|
     |                                         |
     |  注册 event_callbacks["evt_a"] = cb_a   |
     |  发现 stream_status.open = false        |
     |  GET /queue/data?session_hash=Y         |
     |---------------------------------------->|  (建立 SSE 长连接)
     |                                         |
     |  msg: estimation (event_id: "evt_a")   |
     |<----------------------------------------|
     |    → cb_a 处理，fire status event       |
     |                                         |
     |  第2个请求 submit() 并发发起             |
     |  POST /queue/join { session_hash, fn_index, data }
     |---------------------------------------->|
     |                                         |
     |  { event_id: "evt_b" }                  |
     |<----------------------------------------|
     |                                         |
     |  注册 event_callbacks["evt_b"] = cb_b   |
     |  流已打开，不用新建                      |
     |                                         |
     |  msg: process_starts (event_id: "evt_a")|
     |<----------------------------------------|
     |    → cb_a 处理                           |
     |                                         |
     |  msg: estimation (event_id: "evt_b")   |
     |<----------------------------------------|
     |    → cb_b 处理                           |
     |                                         |
     |  msg: process_completed (event_id: "evt_a")
     |<----------------------------------------|
     |    → cb_a 处理，delete cb_a             |
     |    → evt_a 的 iterator 结束              |
     |                                         |
     |  msg: process_completed (event_id: "evt_b")
     |<----------------------------------------|
     |    → cb_b 处理，delete cb_b             |
     |    → evt_b 的 iterator 结束              |
     |                                         |
     |  检测：所有事件都完成了                   |
     |  msg: close_stream                      |
     |<----------------------------------------|
     |    → 关闭 SSE 流                         |
     |    → stream_status.open = false          |
```

### 6.2 旧版 sse 流程（每请求一连接）

```
客户端                                服务端
   |                                     |
   |  第1个请求 submit()                 |
   |  GET /queue/data?fn_index=X&session_hash=Y
   |------------------------------------>|  (建立 SSE 连接1)
   |                                     |
   |  msg: send_hash                     |
   |<------------------------------------|
   |  msg: estimation                    |
   |<------------------------------------|
   |  msg: send_data                     |
   |<------------------------------------|
   |                                     |
   |  POST /queue/data { data, event_id }|
   |------------------------------------>|  (提交数据)
   |                                     |
   |  msg: process_starts                |
   |<------------------------------------|
   |  msg: process_completed             |
   |<------------------------------------|
   |  客户端主动 close()                  |
   |  EventSource.close()                |
   |                                     |
   |  第2个请求 submit()                 |
   |  GET /queue/data?fn_index=X&session_hash=Y
   |------------------------------------>|  (建立 SSE 连接2)
   |  ... 重复上述流程 ...                |
```

### 6.3 取消任务流程

```
客户端                                服务端
   |                                     |
   |  1. POST /cancel                    |
   |     { event_id, session_hash, fn_index }
   |------------------------------------>|
   |                                     |
   |  2. 服务端处理：                     |
   |     - 从队列移除（如果还在排队）     |
   |     - 或取消正在运行的任务           |
   |                                     |
   |  3. 发送 ProcessCompletedMessage    |
   |     success=True, output={}         |
   |<------------------------------------|
   |     (走正常 SSE 分发路径)            |
   |                                     |
   |  4. POST /reset                     |
   |     { event_id }                    |
   |------------------------------------>|
   |     重置迭代器状态                   |
```

---

## 7. 关键端点汇总

| 端点 | 方法 | 用途 | 适用协议 |
|------|------|------|---------|
| `/config` | GET | 获取应用配置 | 所有 |
| `/info` | GET | 获取 API 信息（端点参数） | 所有 |
| `/upload` | POST | 上传文件 | 所有 |
| `/run/{endpoint}` | POST | 非队列模式直接执行 | 非队列 |
| `/queue/join` | POST | 队列模式提交任务，返回 event_id | sse_v1+ |
| `/queue/data` | GET | SSE 长连接接收结果 | sse / sse_v1+ |
| `/queue/data` | POST | sse 模式下提交数据 | sse (旧版) |
| `/cancel` | POST | 取消任务 | 所有队列模式 |
| `/reset` | POST | 重置迭代器状态 | 所有队列模式 |
| `/heartbeat/{session_hash}` | GET | 心跳保活（state 相关） | 所有 |
| `/component_server` | POST | 组件服务端方法调用 | 所有 |

---

## 8. 错误处理

### 8.1 客户端错误场景

| 场景 | 状态码 | 处理方式 |
|------|-------|---------|
| 队列已满 | 503 | 发送 status stage="error"，message="Queue is full" |
| 验证失败 | 422 | 发送 status stage="error"，code="validation_error" |
| 连接断开 | - | 发送 status stage="error"，broken=true |
| 会话未找到 | 404 | 发送 status stage="error"，session_not_found=true |
| 服务端异常 | 500 | 发送 status stage="error" |

### 8.2 服务端错误处理

- **业务异常**（`gr.Error`）：包装为 `ProcessCompletedMessage(success=False)`，在 SSE 流中正常返回
- **未预期异常**：发送 `UnexpectedErrorMessage`
- **队列停止**：POST /queue/join 返回 503 `Queue is stopped`
- **客户端断开**：SSE 循环检测到断开，清理会话事件

---

## 9. 核心代码参考

### 9.1 客户端提交逻辑入口

[submit.ts:67-74](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L67-L74)
```typescript
let { fn_index, endpoint_info, dependency } = get_endpoint_info(
    api_info, endpoint, api_map, config
);
let resolved_data = map_data_to_params(data, endpoint_info);
```

### 9.2 会话级 SSE 消息分发

[stream.ts:41-77](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L41-L77)
```typescript
stream.onmessage = async function (event) {
    let _data = JSON.parse(event.data);
    if (_data.msg === "close_stream") {
        close_stream(stream_status, that.abort_controller);
        return;
    }
    const event_id = _data.event_id;
    if (!event_id) {
        // 广播给所有回调
    } else if (event_callbacks[event_id]) {
        // 分发给对应回调
        event_callbacks[event_id](_data);
    } else {
        // 缓存早到的消息
        pending_stream_messages[event_id].push(_data);
    }
};
```

### 9.3 回调注册与竞态处理

[submit.ts:619-628](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L619-L628)
```typescript
if (event_id in pending_stream_messages) {
    pending_stream_messages[event_id].forEach((msg) => callback(msg));
    delete pending_stream_messages[event_id];
}
event_callbacks[event_id] = callback;
unclosed_events.add(event_id);
if (!stream_status.open) {
    await this.open_stream();
}
```

### 9.4 服务端入队处理

[queueing.py:373-388](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L373-L388)
```python
event = Event(body.session_hash, fn, request, username)
event.data = body
self.pending_event_ids_session[body.session_hash].add(event._id)
self.event_ids_to_events[event._id] = event
event_queue.queue.append(event)
```

### 9.5 服务端消息推送

[queueing.py:240-249](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L240-L249)
```python
def send_message(self, event, event_message):
    if not event.alive:
        return
    event_message.event_id = event._id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```
