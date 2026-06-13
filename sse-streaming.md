# Gradio SSE 流式通信代码走向分析（修正版）

本文档严格按照代码逻辑，梳理 Gradio 中基于 Server-Sent Events (SSE) 的流式通信完整链路，**明确划分每个模块的职责边界**，重点说明：
- 生成器推进发生的准确层级
- 后端流式 diff 的产出位置
- 前端消息解析与提交回调的职责划分

---

## 一、整体架构与职责划分总览

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              前端 (JavaScript)                               │
│                                                                              │
│  [submit.ts] submit()                                                         │
│     │ 1. POST /queue/join 入队，拿 event_id                                   │
│     │ 2. 注册 callback[event_id] = 业务处理函数                                │
│     │ 3. 若无全局流则调 open_stream()                                          │
│     ▼                                                                        │
│  [stream.ts] open_stream()                                                    │
│     │ 建立 EventSource 连 /queue/data?session_hash=xxx                        │
│     │ onmessage: 纯分发 —— JSON.parse → 按 event_id 分发给 callback            │
│     ▼                                                                        │
│  [submit.ts] callback (_data)                                                 │
│     │ 1. handle_message() → 转内部事件类型 (update/generating/complete/...)    │
│     │ 2. apply_diff_stream() → 后端 diff 合并为全量数据                        │
│     │ 3. fire_event() → 触发组件更新 (status / data / log 等)                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼ HTTP
┌──────────────────────────────────────────────────────────────────────────────┐
│                              后端 (Python)                                   │
│                                                                              │
│  [routes.py] /queue/join (POST)                                              │
│     │ 调用 Queue.push() → 创建 Event，登记 session_hash                       │
│     ▼                                                                        │
│  [queueing.py] Queue 核心                                                    │
│     │                                                                        │
│     ├─ push(): 创建 Event → 入 event_queue_per_concurrency_id                │
│     │       → 为 session 建 AsyncQueue: pending_messages_per_session[sh]     │
│     │                                                                        │
│     ├─ start_processing(): 后台协程调度，从队列取 Event 交给 process_events    │
│     │                                                                        │
│     └─ process_events(): 流式循环核心                                        │
│          │                                                                   │
│          ├─ while response.is_generating:                                    │
│          │    ├─ send_message(ProcessGeneratingMessage) → 写入 AsyncQueue    │
│          │    └─ call_process_api() → 调用 blocks.process_api                │
│          │                                                                   │
│          └─ 循环结束后 send_message(ProcessCompletedMessage)                 │
│                                                                              │
│  [route_utils.py] call_process_api()                                         │
│     │ 桥接层：恢复 session_state 和 iterator → 调 blocks.process_api          │
│     │ 保存 iterator 到 App.iterators[event_id]                               │
│     ▼                                                                        │
│  [blocks.py] process_api()                                                   │
│     │ 1. preprocess_data()                                                  │
│     │ 2. call_function() → ★ 生成器推进就在这一层 ★                           │
│     │ 3. postprocess_data()                                                 │
│     │ 4. handle_streaming_outputs() → 音频/视频流处理                        │
│     │ 5. handle_streaming_diffs() → ★ 后端 diff 就在这产出 ★                 │
│     ▼                                                                        │
│  [blocks.py] call_function()                                                 │
│     │ 首次调用: fn(*inputs) → 创建 generator                                 │
│     │ 后续调用: await async_iteration(iterator) → anext(iterator)            │
│     ▼                                                                        │
│  [utils.py] diff(prev, new)                                                  │
│     │ 字符串: if new.startswith(old) → ["append", [], new[len(old):]]        │
│     │ 其他: 递归比较 → [replace/append/add/delete, path, value]              │
│                                                                              │
│  [routes.py] /queue/data (GET)                                               │
│     │ queue_data_helper() 循环                                               │
│     │   ├─ await messages.get() → 从 AsyncQueue 取消息                       │
│     │   ├─ process_msg() → 序列化为 "data: <JSON>\n\n"                       │
│     │   └─ yield → StreamingResponse 流式输出                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、阶段 1：消息生成（后端 Python 层）—— 职责与层级精准划分

### 2.1 生成器推进的准确位置：`blocks.call_function`

**之前误解**：以为生成器 `__anext__` 在 `process_api` 或 `call_process_api`。  
**实际代码**：`__anext__` 发生在 **`blocks.call_function`** 内部。

位置：[blocks.py#L1660-L1676](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L1660-L1676)

```python
async def call_function(self, block_fn, processed_input, iterator=None, ...):
    # ... 省略非 generator 分支 ...

    if inspect.isgeneratorfunction(fn) or inspect.isasyncgenfunction(fn):
        try:
            if iterator is None:
                # 首次调用：prediction 是用户函数返回的 generator 对象
                iterator = cast(AsyncIterator[Any], prediction)
            if inspect.isgenerator(iterator):
                iterator = utils.SyncToAsyncIterator(iterator, self.limiter)

            # ★★★ 生成器推进的真正位置 ★★★
            prediction = await utils.async_iteration(iterator)   # L1666
            is_generating = True
        except StopAsyncIteration:
            # 生成结束
            prediction = components._Keywords.FINISHED_ITERATING
            iterator = None

    return {
        "prediction": prediction,     # 当前 chunk 的原始结果
        "duration": duration,
        "is_generating": is_generating,
        "iterator": iterator,         # 传递回外层保存
    }
```

`utils.async_iteration` 的极简实现（[utils.py#L885-L886](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/utils.py#L885-L886)）：
```python
async def async_iteration(iterator):
    return await anext(iterator)   # 就是标准的取下一个值
```

> **关键结论**：生成器的每一次 `yield` 对应一次 `call_function` 调用，由 queueing.py 的 while 循环驱动。

### 2.2 调用链层级（从外到内）

| 层级 | 函数 | 职责 |
|------|------|------|
| 1 | `queueing.process_events` **while 循环** | 控制是否继续迭代（检查 `is_generating`），负责消息封装和发送 |
| 2 | `route_utils.call_process_api` | 桥接：恢复/保存 `iterator` 到 `App.iterators[event_id]` |
| 3 | `blocks.process_api` | 流程编排：预处理 → 调用函数 → 后处理 → 流处理 → **diff 计算** |
| 4 | `blocks.call_function` | **实际执行 `anext(iterator)` 推进生成器** |

---

## 三、阶段 2：后端流式 diff 的产出位置

### 3.1 核心方法：`blocks.handle_streaming_diffs`

**之前误解**：以为 diff 在前端或 queueing 层做。  
**实际代码**：diff 在 **`blocks.handle_streaming_diffs`** 中产出，且后端维护自己的 diff 状态。

位置：[blocks.py#L2140-L2172](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L2140-L2172)

```python
def handle_streaming_diffs(self, block_fn, data, session_hash, run, final, simple_format=False):
    if session_hash is None or run is None:
        return data

    # 后端也有 pending_diff_streams! (blocks.py L1088 初始化)
    # self.pending_diff_streams = defaultdict(dict)
    first_run = run not in self.pending_diff_streams[session_hash]
    if first_run:
        self.pending_diff_streams[session_hash][run] = [None] * len(data)
    last_diffs = self.pending_diff_streams[session_hash][run]

    for i in range(len(block_fn.outputs)):
        if final:
            # ★ 最后一次：直接返回全量数据（不再是 diff）
            data[i] = last_diffs[i]
            continue

        if first_run:
            # 第一次：保存全量，不做 diff
            last_diffs[i] = data[i]
        else:
            # ★★★ 后端计算 diff 的核心 ★★★
            prev_chunk = last_diffs[i]
            last_diffs[i] = data[i]
            if not simple_format:
                data[i] = utils.diff(prev_chunk, data[i])   # 替换成 diff!

    if final:
        del self.pending_diff_streams[session_hash][run]

    return data
```

### 3.2 何时被调用

位置：[blocks.py#L2305-L2323](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L2305-L2323)

```python
# blocks.process_api 内部
if is_generating or was_generating:
    run = id(old_iterator) if was_generating else id(iterator)
    async with trace_phase("streaming_diff"):
        data = await self.handle_streaming_outputs(block_fn, data, ...)
        data = self.handle_streaming_diffs(block_fn, data, session_hash=session_hash,
                                          run=run, final=not is_generating,
                                          simple_format=simple_format)
```

> **关键结论**：
> - 第 1 次生成：后端发出 **全量数据**，同时存入 `pending_diff_streams`
> - 中间 N 次生成：后端发出 **diff 数据**（`utils.diff()` 计算）
> - 最后 1 次（`final=True`，即 `is_generating=False`）：后端再次发出 **全量数据**

### 3.3 `utils.diff` 算法

位置：[utils.py#L1438-L1490](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/utils.py#L1438-L1490)

```python
def diff(old, new):
    def compare_objects(obj1, obj2, path=None):
        if obj1 == obj2:
            return []
        if type(obj1) is not type(obj2):
            return [["replace", path, obj2]]
        # ★ 字符串优化：如果新串以旧串开头，只发增量部分
        if isinstance(obj1, str) and obj2.startswith(obj1):
            return [["append", path, obj2[len(obj1):]]]
        if isinstance(obj1, list):
            # ... 递归比较列表 ...
        if isinstance(obj1, dict):
            # ... 递归比较字典 ...
    return compare_objects(old, new)
```

diff 输出格式：`[action, path, value][]`，action 包括 `replace` / `append` / `add` / `delete`。

---

## 四、阶段 3：队列转发（Queue 层）

### 4.1 核心数据结构

位置：[queueing.py#L126-L132](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L126-L132)

```python
class Queue:
    def __init__(self, ...):
        # ★ 每个 session 一个 AsyncQueue，所有消息按序入队
        self.pending_messages_per_session: LRUCache[str, AsyncQueue[EventMessage]] = LRUCache(2000)
        # 跟踪 session 下还有哪些未完成的 event_id
        self.pending_event_ids_session: dict[str, set[str]] = {}
        # event_id → Event 对象全局索引
        self.event_ids_to_events: dict[str, Event] = {}
```

### 4.2 流式循环：`process_events`

位置：[queueing.py#L898-L1006](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L898-L1006)

```python
# 第一次调用 call_process_api
response = await route_utils.call_process_api(app=app, body=body, ...)

if response and response.get("is_generating", False):
    while response and response.get("is_generating", False):
        # 1. 把上一次的 response 封装成 ProcessGeneratingMessage 发送
        for event in awake_events:
            self.send_message(event, ProcessGeneratingMessage(
                msg=ServerMessage.process_generating if not event.streaming
                    else ServerMessage.process_streaming,
                output=old_response,    # 这里已经是 handle_streaming_diffs 处理过的 diff
                success=True,
            ))
        # 2. 若是 streaming 连接，等前端 POST 新数据
        if awake_events[0].streaming:
            awake_events, closed_events = await Queue.wait_for_batch(awake_events, timeouts)
        # 3. 再次调用，推进生成器
        response = await route_utils.call_process_api(app=app, body=body, ...)
    # 循环结束，发送最终结果
    for event in awake_events:
        self.send_message(event, ProcessCompletedMessage(output=output, success=True))
```

### 4.3 `send_message` —— 唯一写入点

位置：[queueing.py#L240-L249](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L240-L249)

```python
def send_message(self, event: Event, event_message: EventMessage):
    if not event.alive:
        return
    event_message.event_id = event._id
    # ★ 唯一出口：写入 session 的 AsyncQueue，等待 SSE endpoint 取出
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```

---

## 五、阶段 4：SSE Endpoint（HTTP 层）

### 5.1 核心实现：`queue_data_helper`

位置：[routes.py#L1473-L1577](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1473-L1577)

**职责**：
1. 建立长连接，循环从 `pending_messages_per_session[session_hash]` 取消息
2. 将消息对象序列化为 SSE 文本格式
3. 通过 `yield` 流式输出
4. 所有事件完成后发 `close_stream` 并关闭连接

```python
async def queue_data_helper(request, session_hash, process_msg):
    async def heartbeat():
        while blocks.is_running:
            await asyncio.sleep(heartbeat_rate)
            queue = blocks._queue.pending_messages_per_session.get(session_hash)
            if queue:
                await queue.put(HeartbeatMessage())

    async def sse_stream(request):
        heartbeat_task = asyncio.create_task(heartbeat())
        try:
            while True:
                if await request.is_disconnected():
                    await blocks._queue.clean_events(session_hash=session_hash)
                    return

                # ★ 从 AsyncQueue 取一条消息（10s 超时防挂死）
                message = None
                try:
                    messages = blocks._queue.pending_messages_per_session[session_hash]
                    message = await asyncio.wait_for(messages.get(), timeout=10)
                except TimeoutError:
                    pass

                if message:
                    # ★ 序列化为 SSE 格式："data: <JSON>\n\n"
                    response = process_msg(message)
                    if response is not None:
                        yield response

                    # 如果是完成消息，检查该 session 是否还有未完成事件
                    if isinstance(message, ProcessCompletedMessage) and message.event_id:
                        blocks._queue.pending_event_ids_session[session_hash].remove(message.event_id)
                        if len(blocks._queue.pending_event_ids_session[session_hash]) == 0:
                            # 全部完成，发关闭消息
                            message = CloseStreamMessage()
                            yield process_msg(message)
                            return
        finally:
            heartbeat_task.cancel()

    return StreamingResponse(sse_stream(request), media_type="text/event-stream")
```

### 5.2 两种序列化格式

| 路由 | `process_msg` 实现 | 输出格式 | 使用方 |
|------|-------------------|----------|--------|
| `/queue/data` | [routes.py#L1468-L1469](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1468-L1469) | `data: <完整JSON>\n\n` | 内部前端 |
| `/call/{api}/{event_id}` | [routes.py#L1439-L1459](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1439-L1459) | `event: generating\ndata: <data部分JSON>\n\n` | 外部 API / curl |

---

## 六、阶段 5：前端接收 —— 两层职责严格分离

前端有**两层**处理 SSE 消息，职责完全不重叠：

| 层级 | 所在文件 | 函数 | 核心职责 | 不做什么 |
|------|----------|------|----------|----------|
| **第一层：全局分发层** | [stream.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts) | `open_stream.onmessage` | 1. `JSON.parse(event.data)`<br>2. 按 `event_id` 路由到对应 callback<br>3. 处理 `close_stream` 全局关闭<br>4. 处理回调未就绪时的暂存 (`pending_stream_messages`) | ❌ 不调用 handle_message<br>❌ 不做 diff 合并<br>❌ 不触发组件更新 |
| **第二层：业务处理层** | [submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts) | `callback (_data)` | 1. `handle_message()` → 转内部事件类型<br>2. `apply_diff_stream()` → 合并后端 diff<br>3. `fire_event()` → 触发组件渲染 | ❌ 不做 JSON 解析<br>❌ 不关心 event_id 路由 |

---

### 6.1 第一层：`open_stream.onmessage` —— 纯分发

位置：[stream.ts#L41-L77](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts#L41-L77)

```typescript
stream.onmessage = async function (event: MessageEvent) {
    // 1. 只做 JSON 解析
    let _data = JSON.parse(event.data);

    // 2. 全局关闭消息
    if (_data.msg === "close_stream") {
        close_stream(stream_status, that.abort_controller);
        return;
    }

    const event_id = _data.event_id;

    if (!event_id) {
        // 无 event_id → 广播给所有回调
        await Promise.all(Object.keys(event_callbacks).map(id => event_callbacks[id](_data)));
    } else if (event_callbacks[event_id]) {
        // ★ 按 event_id 分发给 submit.ts 注册的 callback
        if (_data.msg === "process_completed") unclosed_events.delete(event_id);
        // setTimeout 让浏览器有机会刷新 UI
        setTimeout(event_callbacks[event_id], 0, _data);
    } else {
        // 回调还没注册 → 暂存，等 callback 注册后重放
        if (!pending_stream_messages[event_id]) pending_stream_messages[event_id] = [];
        pending_stream_messages[event_id].push(_data);
    }
};
```

### 6.2 第二层：`callback` —— 业务处理

位置：[submit.ts#L489-L617](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L489-L617)

```typescript
let callback = async function (_data: object): Promise<void> {
    try {
        // ★ 1. 后端原始消息 → 前端内部事件
        const { type, status, data, original_msg } = handle_message(_data, last_status[fn_index]);

        if (type == "heartbeat") return;  // 心跳直接丢

        if (type === "update" && status) {
            fire_event({ type: "status", ...status });   // 排队/进度更新
        } else if (type === "complete") {
            complete = status;                            // 标记完成
        } else if (type === "log") {
            fire_event({ type: "log", ...data });          // 日志提示
        } else if (type === "generating" || type === "streaming") {
            fire_event({ type: "status", ...status });     // 生成中状态

            // ★ 2. 后端 diff → 前端合并成完整数据
            if (data && dependency.connection !== "stream" && ["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)) {
                apply_diff_stream(pending_diff_streams, event_id!, data);
            }
        }

        // ★ 3. 触发组件数据更新
        if (data) {
            fire_event({
                type: "data",
                data: handle_payload(data.data, dependency, config.components, "output", ...)
            });
        }

        if (status?.stage === "complete" || status?.stage === "error") {
            // 清理资源
            delete event_callbacks[event_id!];
            delete pending_diff_streams[event_id!];
            close();
        }
    } catch (e) {
        console.error("Unexpected client exception", e);
    }
};

// callback 注册到 event_callbacks
event_callbacks[event_id] = callback;

// 如果之前有暂存的消息，现在重放
if (event_id in pending_stream_messages) {
    pending_stream_messages[event_id].forEach(msg => callback(msg));
    delete pending_stream_messages[event_id];
}
```

### 6.3 `handle_message` —— 消息类型转换器

位置：[api_info.ts#L234-L355](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/helpers/api_info.ts#L234-L355)

后端 `msg` → 前端 `type` 映射表：

| 后端 msg (EventMessage.msg) | 前端 type | 说明 |
|----------------------------|-----------|------|
| `"estimation"` | `"update"` | 排队位置/ETA |
| `"process_starts"` | `"update"` | 开始处理 |
| `"progress"` | `"update"` | 进度条更新 |
| `"process_generating"` | `"generating"` | 生成中（带 diff 数据） |
| `"process_streaming"` | `"streaming"` | 双向流数据 |
| `"process_completed"` | `"complete"` + `data` | 最终结果 |
| `"log"` | `"log"` | 日志消息 |
| `"heartbeat"` | `"heartbeat"` | 直接忽略 |
| `"unexpected_error"` | `"unexpected_error"` | 异常 |
| `"broken_connection"` | `"broken_connection"` | 连接断开 |

### 6.4 `apply_diff_stream` —— 前端 diff 合并

位置：[stream.ts#L101-L178](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts#L101-L178)

**注意**：前端也有 `pending_diff_streams`，但和后端的不是一回事：

| 位置 | 数据结构 | 用途 |
|------|----------|------|
| 后端 | `blocks.pending_diff_streams[session_hash][run]` | 存**上一份全量**，用于计算 diff |
| 前端 | `client.pending_diff_streams[event_id]` | 存**当前累积全量**，用于合并 diff |

```typescript
export function apply_diff_stream(pending_diff_streams, event_id, data): void {
    let is_first_generation = !pending_diff_streams[event_id];
    if (is_first_generation) {
        // 第一次是全量数据 → 直接保存
        pending_diff_streams[event_id] = [];
        data.data.forEach((value, i) => {
            pending_diff_streams[event_id][i] = value;
        });
    } else {
        // 后续是 diff → 合并到累积全量上
        data.data.forEach((value, i) => {
            let new_data = apply_diff(pending_diff_streams[event_id][i], value);
            pending_diff_streams[event_id][i] = new_data;
            data.data[i] = new_data;  // ★ 原地替换 data.data 为全量
        });
    }
}
```

> **关键结论**：`apply_diff_stream` 之后，`data.data` 已经是完整数据，后续 `fire_event("data")` 直接用全量更新组件。

---

## 七、时序图（generator + sse_v3，含 diff 流程）

```
前端 (JS)                                            后端 (Python)
   │                                                    │
   │ 1. submit()
   │    POST /queue/join {fn_index, data, session_hash} │
   │───────────────────────────────────────────────────►│
   │                                                    │  Queue.push(event)
   │                                                    │   → 创建 AsyncQueue[session_hash]
   │                                                    │   → event 入 concurrency 队列
   │◄───────────────────────────────────────────────────│
   │       {event_id: "abc123"}                         │
   │                                                    │
   │ 2. 注册 callback["abc123"] = fn                    │
   │ 3. if (!stream_status.open) → open_stream()        │
   │                                                    │
   │ 4. GET /queue/data?session_hash=xxx                │
   │◄───────────────────────────────────────────────────│
   │    (SSE 连接建立)                                  │
   │                                                    │
   │                                                    │  Queue.start_processing 调度
   │                                                    │  → process_events(event)
   │                                                    │
   │                                                    │  call_process_api()
   │                                                    │    → blocks.process_api()
   │                                                    │      → call_function()
   │                                                    │        → generator = fn()  ★首次创建
   │                                                    │        → anext(generator) → "Hel"
   │                                                    │      → handle_streaming_diffs():
   │                                                    │        first_run → 存全量，不做 diff
   │◄── data: {msg:"process_generating",               │
   │           event_id:"abc123",                       │
   │           output:{data:["Hel"], is_generating:true}} │
   │                                                    │
   │  stream.onmessage 分发                             │
   │  → callback(_data)                                 │
   │    → handle_message → type:"generating"            │
   │    → apply_diff_stream: 首次 → 存全量              │
   │    → fire_event("data") → Textbox "Hel"            │
   │                                                    │
   │                                                    │  下一轮循环
   │                                                    │  call_process_api()
   │                                                    │    → blocks.process_api()
   │                                                    │      → call_function()
   │                                                    │        → anext(generator) → "lo"
   │                                                    │      → handle_streaming_diffs():
   │                                                    │        非首次 → utils.diff("Hel","Hello")
   │                                                    │        → [["append", [0], "lo"]]
   │◄── data: {msg:"process_generating",               │
   │           event_id:"abc123",                       │
   │           output:{data:[["append",[0],"lo"]], ...}}│
   │                                                    │
   │  callback(_data)                                   │
   │    → handle_message → type:"generating"            │
   │    → apply_diff_stream: 非首次                     │
   │        apply_diff("Hel", [["append",[0],"lo"]])    │
   │        → "Hello" (data.data 原地替换)              │
   │    → fire_event("data") → Textbox "Hello"          │
   │                                                    │
   │                  ... 更多 chunk ...                │
   │                                                    │
   │                                                    │  StopAsyncIteration
   │                                                    │  call_process_api()
   │                                                    │    → handle_streaming_diffs():
   │                                                    │      final=True → data[i] = last_diffs[i]
   │                                                    │      （全量数据："Hello World!"）
   │◄── data: {msg:"process_completed",                │
   │           event_id:"abc123",                       │
   │           output:{data:["Hello World!"], ...}}     │
   │                                                    │
   │  callback(_data)                                   │
   │    → handle_message → type:"complete" + data       │
   │    → fire_event("data") → 最终显示                 │
   │    → fire_event("status", stage:"complete")        │
   │                                                    │
   │                                                    │  pending_event_ids_session 清空
   │◄── data: {msg:"close_stream"}                      │
   │  → close_stream() → abort_controller               │
```

---

## 八、关键数据结构速查（区分前后端）

### 后端（Python）

| 结构 | 位置 | 作用 |
|------|------|------|
| `Queue.pending_messages_per_session` | [queueing.py#L126-L128](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L126-L128) | `{session_hash: AsyncQueue[EventMessage]}` 每个会话的 SSE 输出队列 |
| `Queue.pending_event_ids_session` | [queueing.py#L129](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L129) | `{session_hash: {event_id,...}}` 跟踪未完成事件，判断是否关流 |
| `Queue.event_ids_to_events` | [queueing.py#L130](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L130) | `{event_id: Event}` 事件全局索引 |
| `App.iterators` | [routes.py#L238](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L238) | `{event_id: AsyncIterator}` 生成器对象持久化 |
| `Blocks.pending_diff_streams` | [blocks.py#L1088](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L1088) | `{session_hash: {run_id: [last_chunk,...]}}` 后端存上一份全量，用于算 diff |

### 前端（TypeScript）

| 结构 | 位置 | 作用 |
|------|------|------|
| `Client.event_callbacks` | [client.ts#L65](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/client.ts#L65) | `{event_id: callback}` 按 event_id 分发消息 |
| `Client.pending_stream_messages` | [client.ts#L63](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/client.ts#L63) | `{event_id: [msg,...]}` 回调未就绪时暂存消息 |
| `Client.pending_diff_streams` | [client.ts#L64](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/client.ts#L64) | `{event_id: [accumulated_data,...]}` 前端存累积全量，用于合 diff |
| `Client.stream_status` | [client.ts#L61](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/client.ts#L61) | `{open: boolean}` 全局 SSE 连接状态 |

---

## 九、核心修正点总结（累计）

针对多轮理解的偏差，这里全部明确纠正：

| 问题 | 之前误解 | 实际代码 |
|------|----------|----------|
| **生成器推进层级** | `blocks.process_api` 内部直接调 `__anext__` | `blocks.call_function` 内部通过 `utils.async_iteration(iterator)` → `await anext(iterator)` |
| **后端 diff 位置** | 在前端或 queueing 层 | 在 `blocks.handle_streaming_diffs()`，通过 `utils.diff(prev, new)` 计算 |
| **后端 diff 状态** | 只有前端存 diff 累积 | 后端也有 `pending_diff_streams[session_hash][run]` 存上一份全量 |
| **前端两层职责** | `onmessage` 里做业务处理 | `open_stream.onmessage` 只做**分发**，**业务处理全部在 submit 的 callback 里** |
| **最后一次数据** | 最后一次也是 diff | `final=True` 时后端返回**全量数据**（`data[i] = last_diffs[i]`） |
| **旧版 SSE 入口** | 先 POST `/queue/join` 再连 SSE | `protocol === "sse"` 时**直接连 `/queue/data?fn_index=X&session_hash=Y`**，跳过 `/queue/join` |
| **`SSE_URL` 常量含义** | 名字即含义 | **命名反直觉**：`SSE_URL = "queue/data"`（流），`SSE_DATA_URL = "queue/join"`（入队） |
| **`/call/v2` vs `/call`** | 同一入口 | `/call/v2` 接收 `dict[str, Any]`（命名参数），`/call` 接收 `list[Any]`（位置参数），都注入 `simple_format=True` |
| **`GET /call/{api}/{event_id}` key** | 用 session_hash 查队列 | 外部 API 不传 session_hash → `event.session_hash = event._id`，所以用 event_id 作为队列 key |
| **`POST /queue/data` 后端路由** | 存在对应路由 | 前端有调用但后端 routes.py 中**无此 POST 路由**，为历史遗留代码 |
| **iterator 取消防护** | 仅靠删除字典 | `App.iterators_to_reset` 黑名单 + `restore_session_state` 入口检查，双重防护防竞态 |
| **`/reset` 路由功能** | 实际重置 iterator | **空操作**，所有清理逻辑已移到 `/cancel`，保留仅为兼容 |

---

## 十、边界点 1：所有 API 入口与 `simple_format` / diff 处理对应关系

### 10.1 后端路由完整清单

后端 routes.py 中共有 **6 个** 与 SSE / 事件触发相关的入口，按职责分为三类：

#### 类别 A：入队请求（POST）—— 创建 Event 对象并入队

| 路由 | 所在函数 | 调用方 | `simple_format` | 说明 |
|------|----------|--------|----------------|------|
| `POST /queue/join` | [routes.py#L1357-L1368](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1357-L1368) `queue_join` | Gradio 前端 UI（内部） | **False**（默认） | 前端 Web 界面触发事件，body 是完整 `PredictBody`，必须带 `session_hash` |
| `POST /call/{api_name}` | [routes.py#L1342-L1355](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1342-L1355) `simple_predict_post` | 外部 API 客户端（Gradio Client SDK / curl） | **True**（显式注入） | body 是 `SimplePredictBody(data: list[Any], session_hash: str \| None)`，按**位置参数**传参 |
| `POST /call/v2/{api_name}` | [routes.py#L1318-L1340](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1318-L1340) 匿名函数 | 同上（v2 协议） | **True**（显式注入） | body 是 `dict[str, Any]`，按**命名参数**传参，内部通过 `client_utils.construct_args()` 转成位置参数 |

**所有三条入队路由最终都会进入同一个 `queue_join_helper`**，返回 `{"event_id": "..."}`。

---

#### 类别 B：SSE 接收端点（GET）—— 流式接收事件消息

| 路由 | 所在函数 | 调用方 | 说明 |
|------|----------|--------|------|
| `GET /queue/data` | [routes.py#L1463-L1471](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1463-L1471) `queue_data` | Gradio 前端 UI（内部） | query 参数 `session_hash`，从 `pending_messages_per_session[session_hash]` 取消息，**输出完整 JSON** |
| `GET /call/{api_name}/{event_id}` | [routes.py#L1434-L1461](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1434-L1461) `simple_predict_get` | 外部 API 客户端 | path 参数 `event_id`，**把 event_id 当作 session_hash 用**，从 `pending_messages_per_session[event_id]` 取消息，输出简化的 `event: ...\ndata: ...\n\n` 格式 |
| `GET /call/v2/{api_name}/{event_id}` | 同上（共享同个装饰器） | 同上 | 与上面完全相同，只是多了 v2 路径别名 |

**关键：`GET /call/{api}/{event_id}` 的 `event_id` 重用为 `session_hash`**

[queueing.py#L63](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L63) 中 Event 的 `session_hash` 有一个 fallback 逻辑：
```python
class Event:
    def __init__(self, session_hash: str | None, ...):
        self._id = uuid.uuid4().hex
        self.session_hash: str = session_hash or self._id   # ★ 关键！
```

这意味着：
- 前端 UI 调用 `/queue/join`：传了 `session_hash` → `event.session_hash = 传入值`
- 外部 API 调用 `/call/{api_name}`：**没传 `session_hash`** → `event.session_hash = event._id`（即 event_id）
- 所以外部 API 后续调用 `GET /call/{api}/{event_id}` 时，把 `event_id` 传给 `queue_data_helper` 的 `session_hash` 参数，就能精确匹配到 `pending_messages_per_session[event_id]` 这个独立队列

---

#### 类别 C：取消端点

| 路由 | 说明 |
|------|------|
| `POST /cancel` | 取消指定 event_id 的执行，清理 iterator（见第十一章） |
| `POST /reset` | 空操作（兼容遗留代码，所有逻辑已移到 `/cancel`） |

---

### 10.2 `simple_format` 的源头注入与传递链路

#### 源头注入（routes.py）

```python
# /call/{api_name} 和 /call/v2/{api_name} —— 注入 simple_format=True
full_body = PredictBody(**body.model_dump(), simple_format=True)   # routes.py L1335 / L1350

# /queue/join —— PredictBody 本身 simple_format: bool = False 默认值
# 前端不传，所以 simple_format=False                                 (data_classes.py L97)
```

#### 传递链路

```
入队路由 (routes.py)
    ↓ 构造 PredictBody(simple_format=X)
queue_join_helper → Queue.push(event)
    ↓ event 入队等待
Queue.process_events → call_process_api(body, ...)
    ↓ body.simple_format 一路透传
route_utils.call_process_api
    ↓ [route_utils.py L390]
blocks.process_api(simple_format=body.simple_format, ...)
    ↓ [blocks.py L2321]
blocks.handle_streaming_diffs(..., simple_format=simple_format)
    ↓ [blocks.py L2165]
    if not simple_format:
        data[i] = utils.diff(prev_chunk, data[i])   # 只有 False 才做 diff
```

### 10.3 入口与 `simple_format`、diff、接收端的完整对应矩阵

| 入队入口 | `simple_format` | 中间 N 次生成 | SSE 接收端 | 前端协议 | 是否 `apply_diff_stream` |
|----------|----------------|-------------|------------|----------|-------------------------|
| `POST /queue/join`（前端内部） | **False** | **diff 数据** | `GET /queue/data`（完整 JSON） | sse_v2/v3 | ✅ 需要（sse_v2+ 才启用） |
| `POST /call/{api_name}`（外部 API） | **True** | **全量数据**（不做 diff） | `GET /call/{api}/{event_id}`（简化格式） | — | ❌ 不需要 |
| `POST /call/v2/{api_name}`（外部 API v2） | **True** | **全量数据**（不做 diff） | `GET /call/v2/{api}/{event_id}`（同上） | — | ❌ 不需要 |

> **设计意图**：外部 API 的消费方（curl、第三方 SDK）通常不具备 diff 合并能力，所以直接发全量，牺牲带宽换兼容性；前端内部 UI 对延迟敏感，走 diff 压缩。

---

## 十一、边界点 2：iterator 在取消、reset、会话恢复时的状态隔离

### 11.1 核心安全机制：`App.iterators_to_reset` 集合

这是防止旧 iterator 状态泄漏的**关键屏障**。

位置：[routes.py#L238-L240](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L238-L240)（App 类字段）

```python
class App:
    def __init__(self, **kwargs):
        self.iterators: dict[str, AsyncIterator] = {}         # 运行中的生成器
        self.iterators_to_reset: set[str] = set()            # ★ 已取消的 event_id
```

### 11.2 取消流程（用户点击停止按钮 / 前端调用 `cancel()`）

#### 前端触发

[submit.ts#L110-L139](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L110-L139)

```typescript
async function cancel(): Promise<void> {
    cancel_request = { event_id, session_hash, fn_index };
    await fetch(`${config.root}${api_prefix}/${CANCEL_URL}`, {  // CANCEL_URL = "cancel"
        method: "POST", body: JSON.stringify(cancel_request)
    });
    await fetch(`${config.root}${api_prefix}/${RESET_URL}`, {   // RESET_URL = "reset"
        method: "POST", body: JSON.stringify({ event_id })
    });
}
```

#### 后端 `/cancel` 处理

[routes.py#L1401-L1429](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1401-L1429)

```python
@router.post("/cancel")
async def cancel_event(body: CancelBody):
    await cancel_tasks({f"{body.session_hash}_{body.fn_index}"})
    blocks = app.get_blocks()
    # 1. 从队列移除
    await blocks._queue.remove_from_queue(body.event_id)
    # 2. 若会话还开着，塞一条 ProcessCompletedMessage 让前端优雅退出
    session_open = body.session_hash in blocks._queue.pending_messages_per_session
    event_running = body.event_id in blocks._queue.pending_event_ids_session.get(body.session_hash, {})
    if session_open and event_running:
        blocks._queue.pending_messages_per_session[body.session_hash].put_nowait(
            ProcessCompletedMessage(output={}, success=True, event_id=body.event_id)
        )
    # 3. ★★★ 核心：iterator 清理 + 打标记 ★★★
    if body.event_id in app.iterators:
        async with app.lock:
            try:
                await safe_aclose_iterator(app.iterators[body.event_id])  # 优雅关闭生成器
            except Exception:
                pass
            del app.iterators[body.event_id]          # 从字典删除
            app.iterators_to_reset.add(body.event_id) # ★ 加入黑名单！
    return {"success": True}
```

**注意**：`/reset` 路由（[routes.py#L1189-L1193](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1189-L1193)）现在是空操作，所有逻辑都在 `/cancel` 中完成：
```python
@router.post("/reset")
async def reset_iterator(body: ResetBody):
    # No-op, all the cancelling/reset logic handled by /cancel
    return {"success": True}
```

### 11.3 恢复流程（下一次 call_process_api 调用）

`restore_session_state` 在 **每次** `call_process_api` 最开头被调用，负责判断是否应该恢复旧 iterator：

[route_utils.py#L320-L340](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/route_utils.py#L320-L340)

```python
def restore_session_state(app: App, body: PredictBodyInternal):
    event_id = body.event_id
    session_hash = getattr(body, "session_hash", None)
    if session_hash is not None:
        session_state = app.state_holder[session_hash]
        # ★★★ iterators_to_reset 检查 ★★★
        # 注释原文：The should_reset set keeps track of the fn_indices that have been cancelled.
        # When a job is cancelled, the /reset route will mark the jobs as having been reset.
        # That way if the cancel job finishes BEFORE the job being cancelled
        # the job being cancelled will not overwrite the state of the iterator.
        if event_id is None:
            iterator = None
        elif event_id in app.iterators_to_reset:   # ★ 在黑名单里？
            iterator = None                         # → 强制丢弃旧 iterator！
            app.iterators_to_reset.remove(event_id) # → 移除黑名单标记
        else:
            iterator = app.iterators.get(event_id)  # → 正常恢复
    else:
        session_state = SessionState(app.get_blocks())
        iterator = None
```

### 11.4 竞态条件防护

`iterators_to_reset` 的存在就是为了处理这种竞态：

```
时序 A（正常）：
  T1 用户生成 "Hel" → iterator 在 app.iterators
  T2 用户点取消 → /cancel → del iterators[id]; iterators_to_reset.add(id)
  T3 （没有后续调用）→ 没问题

时序 B（竞态）：
  T1 queueing.process_events while 循环准备 call_process_api()
  T2 用户点取消 → /cancel → del iterators[id]; iterators_to_reset.add(id)
  T3 process_events 继续执行 call_process_api()
       └─ restore_session_state()
           └─ 检测到 event_id in iterators_to_reset
               └─ iterator = None  ← ★ 不继续用旧生成器 ★
                   → blocks.process_api(iterator=None)
                       → call_function(iterator=None)
                           → 创建新 generator... 但这时 Event 已被标记 alive=False
```

### 11.5 其他清理场景

| 触发场景 | 处理方式 | 位置 |
|----------|----------|------|
| **客户端 SSE 连接断开** | `queue.clean_events(session_hash=...)` → 所有 Event.alive = False + 从队列移除 | [routes.py#L312-L314](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L312-L314) |
| **call_process_api 抛出异常** | `except BaseException:` → `safe_aclose_iterator(iterator)` + 结束所有 MediaStream | [route_utils.py#L404-L413](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/route_utils.py#L404-L413) |
| **`send_message` 投递前** | `if not event.alive: return` → 已死 Event 的消息直接丢弃 | [queueing.py#L245-L249](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L245-L249) |
| **生成器正常结束** | `StopAsyncIteration` → `iterator = None` → `del pending_diff_streams[session_hash][run]` | [blocks.py#L1668-L1676](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L1668-L1676) + [blocks.py#L2169-L2170](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L2169-L2170) |
| **前端正常 complete/error** | `delete event_callbacks[id]` + `delete pending_diff_streams[id]` | [submit.ts#L532-L536](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L532-L536) |

---

## 十二、边界点 3：前端协议分支（sse / sse_v1 / sse_v2 / sse_v3）完整对比

### 12.1 容易搞混的常量命名

先明确常量定义（[constants.ts#L4-L7](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/constants.ts#L4-L7)）：

| 常量名 | 值 | 用途 | 备注 |
|--------|----|------|------|
| `SSE_URL` | `"queue/data"` | 新版 SSE 流的 GET endpoint | **名字和实际用途相反**：SSE_URL 是数据流 URL |
| `SSE_DATA_URL` | `"queue/join"` | 新版入队的 POST endpoint | **名字和实际用途相反**：SSE_DATA_URL 是入队 URL |
| `SSE_URL_V0` | `"queue/join"` | 旧版（历史遗留） | 已不用 |
| `SSE_DATA_URL_V0` | `"queue/data"` | 旧版（历史遗留） | 已不用 |

> **注意**：`SSE_URL` 和 `SSE_DATA_URL` 的命名非常反直觉。记住：
> - `POST SSE_DATA_URL` → 入队 (`/queue/join`)
> - `GET SSE_URL` → 收流 (`/queue/data`)

### 12.2 分支入口

所有分支都在 [submit.ts#L77-L79](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L77-L79) 进入，基于 `config.protocol` 判断：

```typescript
let protocol = config.protocol ?? "ws";
if (protocol === "ws") {
    throw new Error("WebSocket protocol is not supported in this version");
}
// 之后两个大分支：sse / sse_v1~sse_v3
```

后端 `config.protocol` 由 [blocks.py#L2404](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L2404) 写死：`"protocol": "sse_v3"`（当前版本默认 sse_v3）。

---

### 12.3 路径 A：`protocol === "sse"` —— 第一代 SSE（单事件单连接，跳过 `/queue/join`）

**位置**：[submit.ts#L266-L394](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L266-L394)

#### 关键修正：旧版 SSE **不经过** `/queue/join`

```
流程（实际代码）：
  1. 直接构造 EventSource 连接：
     url = `/queue/data?fn_index=X&session_hash=Y`   ← ★ 没有 POST /queue/join！
     stream = this.stream(url)                        (submit.ts L279-L289)

  2. stream.onmessage 内联处理所有逻辑（没有全局分发层）：
     JSON.parse(event.data)
       → handle_message() → {type, status, data}
       → fire_event(status/data/log)                  (submit.ts L297-L392)

  3. 特殊流程：当 type === "data" 时：
     POST /queue/data {session_hash, event_id, ...}    (submit.ts L318-L325)
     ↑ 注：后端 routes.py 中无此 POST 路由，为历史遗留

  4. complete/error 时：
     stream.close() → 关闭本 EventSource              (submit.ts L314 / L390)
```

**核心特点**：
- ✅ **不经过 POST `/queue/join`**，直接 `new EventSource('/queue/data?fn_index=X&session_hash=Y')`
- ❌ 每个事件一条独立 SSE 连接（并发事件会开多个 EventSource）
- ❌ 没有全局流分发层，`onmessage` 内直接做 `handle_message` + `fire_event`
- ❌ 没有 `apply_diff_stream`（sse 协议不支持 diff）
- ❌ 代码中存在 `POST /queue/data` 调用（L318），但后端无对应路由，为历史遗留

---

### 12.4 路径 B：`protocol === "sse_v1"` —— 过渡协议

**位置**：和 sse_v2/v3 走同一块 [submit.ts#L395-L631](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L395-L631) 的分支入口。

```
流程：
  1. POST /queue/join → 拿 event_id                    (submit.ts L430-L440)
  2. 注册 event_callbacks[event_id] = callback
  3. 若 stream_status.open === false → open_stream() 建立全局 SSE
  4. stream.onmessage（stream.ts 中）只做 JSON.parse + 按 event_id 分发
  5. callback 里做 handle_message + fire_event
```

**关键差异**：`apply_diff_stream` 条件不包含 v1（[submit.ts#L551-L557](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L551-L557)）：
```typescript
if (
    data &&
    dependency.connection !== "stream" &&
    ["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)   // ★ sse_v1 不在此列表
) {
    apply_diff_stream(pending_diff_streams, event_id!, data);
}
```

**特点**：
- ✅ 用全局单连接 + `event_callbacks` 分发（和 sse_v2+ 一样）
- ✅ 经过 POST `/queue/join`
- ❌ 不做 diff，每次都是全量数据

---

### 12.5 路径 C：`protocol === "sse_v2" | "sse_v2.1" | "sse_v3"` —— 当前主流协议

**位置**：[submit.ts#L395-L631](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L395-L631)

```
流程：
  1. POST /queue/join → 拿 event_id                    (submit.ts L430-L440)
  2. 注册 event_callbacks[event_id] = callback
  3. 若 stream_status.open === false → open_stream() 建立全局 SSE
  4. stream.onmessage（stream.ts 中）只做 JSON.parse + 按 event_id 分发
  5. callback 里做 handle_message + apply_diff_stream + fire_event
```

**特点**：
- ✅ 经过 POST `/queue/join`
- ✅ 所有事件共享一条 `/queue/data` 连接（`event_callbacks` 多路复用）
- ✅ `onmessage`（分发层）和 `callback`（业务层）职责严格分离（见第六章）
- ✅ 中间 chunk 走 diff 压缩，`apply_diff_stream` 前端合并
- ✅ v3 独有：**只有后端发 `close_stream` 才关闭全局流**（v2/v2.1 可能在事件结束时关）

---

### 12.6 四协议完整特性对比矩阵

| 特性 | sse (v0) | sse_v1 | sse_v2/v2.1 | sse_v3 |
|------|----------|--------|-------------|--------|
| 经过 POST `/queue/join` | ❌ **不经过**，直接连 `/queue/data` | ✅ 经过 | ✅ 经过 | ✅ 经过 |
| 连接模型 | 每事件一条独立 EventSource | 全局单连接共享 | 全局单连接共享 | 全局单连接共享 |
| `event_callbacks` 分发 | ❌ 内联处理（没有分发层） | ✅ | ✅ | ✅ |
| SSE onmessage 职责 | JSON.parse + handle_message + fire_event | 仅 JSON.parse + 分发 | 仅 JSON.parse + 分发 | 仅 JSON.parse + 分发 |
| diff 压缩 | ❌ | ❌ | ✅ | ✅ |
| `type==="data"` 额外 POST | ✅ 代码存在（但后端无路由，遗留） | ❌ 不需要 | ❌ 不需要 | ❌ 不需要 |
| 关闭流的时机 | 事件结束立即 `stream.close()` | 事件结束 | 事件结束 | 后端发 `close_stream` |
| 典型使用场景 | 历史遗留（几乎不用） | 过渡版本 | 当前/近期版本 | 当前默认版本 |

### 12.7 前端分支判断代码位置汇总

| 判断点 | 位置 | 含义 |
|--------|------|------|
| 旧版协议分支 | [submit.ts#L266](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L266) `protocol == "sse"` | 进入旧版单连接路径（跳过 `/queue/join`） |
| 新版协议总分支 | [submit.ts#L395-L399](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L395-L399) `sse_v1 \|\| sse_v2 \|\| sse_v2.1 \|\| sse_v3` | 进入新版全局流路径（先 POST `/queue/join`） |
| diff 启用判断 | [submit.ts#L551-L557](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L551-L557) `["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)` | 是否调用 `apply_diff_stream` 合并后端 diff |
| `close_stream` 处理 | [stream.ts#L73-L76](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts#L73-L76) | 仅 v3 会收到并触发全局流关闭 |

---

## 十三、涉及文件索引

**后端（Python）：**
- [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py) — Queue 类、Event 类（`session_hash = session_hash or self._id` 关键 fallback）、process_events 流式循环、clean_events、remove_from_queue、send_message
- [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py) — 所有 API 入口：`/queue/join`、`/queue/data`、`/call/{api_name}`、`/call/v2/{api_name}`、`/call/{api}/{event_id}`、`/cancel`、`/reset`、`queue_data_helper` SSE 流式响应、App.iterators / App.iterators_to_reset
- [data_classes.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/data_classes.py) — `PredictBody`（`simple_format: bool = False`）、`SimplePredictBody`、`SimplePredictBodyV2`、`CancelBody`、`ResetBody`
- [server_messages.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/server_messages.py) — 所有 EventMessage 类型定义
- [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/route_utils.py) — `call_process_api` 桥接层、`restore_session_state`（iterators_to_reset 黑名单检查）
- [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py) — `process_api` 流程编排、`call_function` 生成器推进（`async_iteration` → `anext`）、`handle_streaming_diffs`（`simple_format` 控制是否做 diff）、`config.protocol = "sse_v3"`
- [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/utils.py) — `diff()` 算法、`async_iteration()`、`safe_aclose_iterator()`

**前端（TypeScript）：**
- [client/js/src/utils/submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts) — submit() 主流程、**sse/sse_v1/sse_v2~v3 四大协议分支**、cancel()、callback 业务处理层、旧版 SSE 直接连 `/queue/data` 的逻辑
- [client/js/src/utils/stream.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts) — `open_stream` 全局分发层、`apply_diff_stream` 前端 diff 合并、`readable_stream`
- [client/js/src/client.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/client.ts) — Client 类、全局状态字段定义
- [client/js/src/helpers/api_info.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/helpers/api_info.ts) — `handle_message` 消息类型转换
- [client/js/src/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/constants.ts) — **命名反直觉的常量**：`SSE_URL = "queue/data"`、`SSE_DATA_URL = "queue/join"`、`CANCEL_URL = "cancel"`、`RESET_URL = "reset"`
