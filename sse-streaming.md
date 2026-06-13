# Gradio SSE 流式通信代码走向分析

本文档梳理 Gradio 中基于 Server-Sent Events (SSE) 的流式通信完整链路，覆盖**消息生成 → 队列转发 → SSE 输出 → 前端接收渲染**四个阶段。

---

## 一、整体架构概览

```
用户触发事件
     │
     ▼
[前端] submit() ──POST──► /queue/join (routes.py)
                                    │
                                    ▼
                         [后端] Queue.push() 创建 Event 入队
                                    │
                                    ▼
                         Queue.start_processing() 后台协程
                                    │
                                    ▼
                         call_process_api() ──► blocks.process_api()
                                    │
                                    │  生成输出 output (含is_generating)
                                    ▼
                         Queue.send_message() 写入 AsyncQueue
                                    │
                                    ▼
                         /queue/data SSE endpoint
                                    │
                         (StreamingResponse yield)
                                    │
     ┌──────────────────────────────┘
     ▼
[前端] EventSource.onmessage 逐条接收
     │
     ▼
  callback() → fire_event() → 组件渲染
```

---

## 二、阶段 1：消息生成（后端 Python 层）

### 2.1 入口：`call_process_api`

位置：[route_utils.py#L362-L417](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/route_utils.py#L362-L417)

```python
async def call_process_api(app, body, gr_request, fn, root_path):
    session_state, iterator = restore_session_state(app=app, body=body)
    ...
    output = await app.get_blocks().process_api(
        block_fn=fn, inputs=inputs, request=gr_request,
        state=session_state, iterator=iterator,
        session_hash=session_hash, event_id=event_id, ...
    )
    iterator = output.pop("iterator", None)       # 保存生成器状态
    if event_id is not None:
        app.iterators[event_id] = iterator         # 挂到 app.iterators 字典
    return output
```

核心作用：
- 恢复会话状态和上次迭代的 `iterator`（生成器函数的上下文）
- 调用 `blocks.process_api()` 执行业务函数
- 将新的 iterator 存回 `app.iterators[event_id]`，供下次迭代使用

### 2.2 核心执行：`blocks.process_api`

位置：[blocks.py#L2174-L2270](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py#L2174-L2270)

对于 generator / streaming 函数，process_api 的返回字典结构：

| 字段 | 含义 |
|------|------|
| `data` | 本批次产出的输出数据 |
| `is_generating` | **关键标记**：True 表示还有后续数据，False 表示生成结束 |
| `iterator` | 生成器对象，下一次迭代继续使用 |
| `duration` | 单次迭代耗时 |
| `average_duration` | 平均耗时 |
| `render_config` | 动态渲染配置（可选） |

```python
# blocks.process_api 对生成器函数的逻辑（简化）:
if inspect.isasyncgenfunction(block_fn.fn):
    if iterator is None:
        iterator = block_fn.fn(*inputs, **kwargs)   # 首次调用创建生成器
    try:
        prediction = await iterator.__anext__()     # 取下一个值
        is_generating = True
    except StopAsyncIteration:
        prediction = <final value>
        is_generating = False
        iterator = None
return {"data": ..., "is_generating": is_generating, "iterator": iterator, ...}
```

> **设计要点**：每次调用只取生成器的 `__anext__()` 一次，产出一个 chunk；`is_generating` 控制是否继续循环。

---

## 三、阶段 2：队列转发（Queue 核心）

### 3.1 消息类型体系

位置：[server_messages.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/server_messages.py)

所有流经队列的消息都继承自 `BaseMessage`，统一带有 `msg` 类型标记和 `event_id`：

| 消息类 | msg 常量 | 触发时机 |
|--------|----------|----------|
| `EstimationMessage` | `estimation` | 入队后/排队中，告知排队位置、ETA |
| `ProcessStartsMessage` | `process_starts` | 事件出队、开始执行前 |
| `ProgressMessage` | `progress` | 用户代码中 `gr.Progress()` 更新 |
| `LogMessage` | `log` | 用户调用 `gr.Info()`/`gr.Warning()` 等 |
| `ProcessGeneratingMessage` | `process_generating` / `process_streaming` | 每次生成器产出一个 chunk 且 `is_generating=True` |
| `ProcessCompletedMessage` | `process_completed` | 生成结束 / 普通函数返回 |
| `HeartbeatMessage` | `heartbeat` | 保活心跳（定时插入） |
| `CloseStreamMessage` | `close_stream` | 所有事件处理完，关闭 SSE 流 |
| `UnexpectedErrorMessage` | `unexpected_error` | 异常或断连 |

### 3.2 入队：`Queue.push()`

位置：[queueing.py#L279-L468](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L279-L468)

关键步骤：
1. 根据 `fn_index` 找到对应的 `BlockFunction`
2. 创建 `Event` 对象（带 `session_hash`、`_id`）
3. **为 session 创建 AsyncQueue**：`pending_messages_per_session[session_hash] = AsyncQueue()`
4. 将 `event_id` 登记到 `pending_event_ids_session`
5. Event 按 `concurrency_id` 放入 `event_queue_per_concurrency_id[id].queue`
6. 立即调用 `broadcast_estimations()` 发送排队位置信息

```python
# queueing.py L382-L388
async with self.pending_message_lock:
    if body.session_hash not in self.pending_messages_per_session:
        self.pending_messages_per_session[body.session_hash] = AsyncQueue()  # ★ 每个 session 一个队列
    if body.session_hash not in self.pending_event_ids_session:
        self.pending_event_ids_session[body.session_hash] = set()
self.pending_event_ids_session[body.session_hash].add(event._id)
self.event_ids_to_events[event._id] = event
```

### 3.3 出队与处理循环：`Queue.start_processing()` + `process_events()`

位置：[queueing.py#L521-L1081](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L521-L1081)

#### 3.3.1 调度循环（`start_processing`）

```python
async def start_processing(self):
    while not self.stopped:
        if len(self) == 0 or None not in self.active_jobs:
            await asyncio.sleep(self.sleep_when_free)
            continue
        event_batch = self.get_events()   # 从 event_queue_per_concurrency_id 取队首
        if event_batch:
            events, batch, concurrency_id = event_batch
            self.active_jobs[self.active_jobs.index(None)] = events
            run_coro_in_background(self.process_events, events, batch, start_time)
```

#### 3.3.2 处理流程（`process_events`）——流式循环的核心

```
process_events(events, batch, begin_time)
    │
    ├─► 发送 ProcessStartsMessage
    │
    ├─► 第一次调用 call_process_api()
    │    返回 response = {data:..., is_generating: True/False, ...}
    │
    ├─► 如果 response.is_generating=True：
    │    │
    │    └─► while 循环：
    │         ├─► 发送 ProcessGeneratingMessage (调用 send_message)
    │         ├─► 若是 streaming 连接：wait_for_batch() 等前端发来新数据
    │         ├─► 再次调用 call_process_api()  —— 复用上次的 iterator
    │         └─► 更新 event.run_time，检查 time_limit
    │
    └─► 循环结束后，发送 ProcessCompletedMessage
```

对应代码（queueing.py L898-L1006 核心循环）：

```python
if response and response.get("is_generating", False):
    while response and response.get("is_generating", False):
        # --- 发送中间 chunk ---
        for event in awake_events:
            self.send_message(event, ProcessGeneratingMessage(
                msg=ServerMessage.process_generating if not event.streaming
                    else ServerMessage.process_streaming,
                output=old_response, success=True, time_limit=...
            ))
        # --- streaming 模式等待前端输入 ---
        if awake_events[0].streaming:
            awake_events, closed_events = await Queue.wait_for_batch(awake_events, timeouts)
        # --- 取下一次迭代 ---
        response = await route_utils.call_process_api(...)
    # --- 发送最终结果 ---
    for event in awake_events:
        self.send_message(event, ProcessCompletedMessage(
            output=output, success=success, used_cache=...
        ))
```

### 3.4 消息投递：`Queue.send_message()`

位置：[queueing.py#L240-L249](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py#L240-L249)

```python
def send_message(self, event: Event, event_message: EventMessage):
    if not event.alive:
        return
    event_message.event_id = event._id                         # 打上 event_id 标记
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)                        # ★ 写入 session 的 AsyncQueue
```

> **关键**：`pending_messages_per_session[session_hash]` 是一个 `asyncio.Queue`，所有发给同一会话的消息按序入此队列，等待 SSE endpoint 取出。

---

## 四、阶段 3：SSE Endpoint（HTTP 流式响应层）

### 4.1 主要路由

位置：[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py)

| 路由 | 方法 | 作用 |
|------|------|------|
| `/queue/join` | POST | **入队请求**：创建 Event，返回 `event_id`（L1357-L1399） |
| `/queue/data` | GET | **主 SSE 流**：按 session 持续输出所有消息（L1463-L1471） |
| `/call/{api_name}/{event_id}` | GET | **外部 API SSE 流**：单事件视角，输出 `complete/generating/heartbeat`（L1431-L1461） |
| `/stream/{event_id}` | POST | 双向流时前端推回数据给后端（L1088-L1094） |
| `/stream/{event_id}/close` | POST | 前端主动关闭流式事件（L1096-L1102） |

### 4.2 SSE 核心实现：`queue_data_helper`

位置：[routes.py#L1473-L1577](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py#L1473-L1577)

```python
async def queue_data_helper(request, session_hash, process_msg):
    async def heartbeat():
        while blocks.is_running:
            await asyncio.sleep(heartbeat_rate)
            queue = blocks._queue.pending_messages_per_session.get(session_hash)
            if queue:
                await queue.put(HeartbeatMessage())       # 定时塞心跳消息

    async def sse_stream(request):
        heartbeat_task = asyncio.create_task(heartbeat())
        try:
            while True:
                # (1) 检查客户端是否断开
                if await request.is_disconnected():
                    await blocks._queue.clean_events(session_hash=session_hash)
                    return

                # (2) 从 AsyncQueue 取一条消息（10s 超时）
                message = None
                try:
                    messages = blocks._queue.pending_messages_per_session[session_hash]
                    message = await asyncio.wait_for(messages.get(), timeout=10)
                except TimeoutError:
                    pass

                # (3) 转换为 SSE 文本格式并 yield
                if message:
                    response = process_msg(message)
                    if response is not None:
                        yield response                                # ★ 流式输出点

                    # (4) ProcessCompletedMessage 之后检查是否关闭流
                    if isinstance(message, ProcessCompletedMessage) and message.event_id:
                        blocks._queue.pending_event_ids_session[session_hash].remove(message.event_id)
                        # 如果所有 pending 事件都完成，则发 close_stream 并退出
                        if len(blocks._queue.pending_event_ids_session[session_hash]) == 0:
                            message = CloseStreamMessage()
                            yield process_msg(message)                # ★ 关闭流标记
                            return
        finally:
            heartbeat_task.cancel()

    return StreamingResponse(sse_stream(request), media_type="text/event-stream")
```

#### 两种 `process_msg` 转换函数

**A. `/queue/data`（内部前端使用）**：完整消息序列化
```python
# routes.py L1468-L1469
def process_msg(message: EventMessage) -> str:
    return f"data: {orjson.dumps(message.model_dump(), default=str).decode('utf-8')}\n\n"
```
输出标准 SSE 格式：每行 `data: <JSON>\n\n`，前端通过 `event.data` 解析。

**B. `/call/{api}/{event_id}`（外部 API 使用）**：简化事件流
```python
# routes.py L1439-L1459
def process_msg(message):
    if isinstance(message, ProcessCompletedMessage):
        event = "complete" if message.success else "error"
        data = msg["output"].get("data")
    elif isinstance(message, ProcessGeneratingMessage):
        event = "generating" if message.success else "error"
        data = msg["output"].get("data")
    ...
    return f"event: {event}\ndata: {json.dumps(data)}\n\n"
```
使用 `event:` 行区分事件类型，便于 curl / EventSource `addEventListener` 使用。

---

## 五、阶段 4：前端接收与渲染

### 5.1 协议版本区分

前端 `submit.ts` 中根据 `config.protocol` 选择不同接收路径（[submit.ts#L266-L631](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L266-L631)）：

| 协议值 | 路径 | 特点 |
|--------|------|------|
| `"sse"` | 旧版（L266-L394） | 单独 EventSource 连 `/queue/join`，自己处理 onmessage |
| `"sse_v1"` / `"sse_v2"` / `"sse_v2.1"` / `"sse_v3"` | 新版（L395-L631） | ★ **复用单条 SSE 连接**：所有事件共享一个 `/queue/data` 流 |

新版（sse_v2+）流程：
1. `POST /queue/join` → 返回 `event_id`
2. 注册 `event_callbacks[event_id] = callback`
3. 若 `stream_status.open == false` → 调用 `open_stream()` 建立全局 SSE 连接
4. 所有回调通过 `event_id` 从全局流中分发给对应的订阅者

### 5.2 全局 SSE 连接：`open_stream`

位置：[stream.ts#L5-L89](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts#L5-L89)

```typescript
export async function open_stream(this: Client): Promise<void> {
    const url = new URL(`${config.root}${this.api_prefix}/${SSE_URL}?session_hash=${this.session_hash}`);
    // SSE_URL = "queue/data"  (constants.ts L6)

    stream = this.stream(url);     // 内部调用 readable_stream() → fetch-event-stream

    stream.onmessage = async function (event: MessageEvent) {
        let _data = JSON.parse(event.data);

        if (_data.msg === "close_stream") {
            close_stream(stream_status, that.abort_controller);
            return;
        }

        const event_id = _data.event_id;

        if (!event_id) {
            // 无 event_id → 广播给所有回调（如 estimation 早期消息）
            await Promise.all(Object.keys(event_callbacks).map(id => event_callbacks[id](_data)));
        } else if (event_callbacks[event_id]) {
            // 有 event_id 且有回调 → 定向分发
            if (_data.msg === "process_completed") unclosed_events.delete(event_id);
            setTimeout(event_callbacks[event_id], 0, _data);   // setTimeout 避免阻塞 UI
        } else {
            // 有 event_id 但回调未注册 → 暂存 pending_stream_messages
            if (!pending_stream_messages[event_id]) pending_stream_messages[event_id] = [];
            pending_stream_messages[event_id].push(_data);
        }
    };
}
```

> **关键设计**：`setTimeout(fn, 0, _data)` 将处理推到事件循环末尾，浏览器有机会刷新 UI，避免高频生成消息导致主线程冻结。

### 5.3 事件回调处理（submit.ts 中的 callback）

位置：[submit.ts#L489-L617](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts#L489-L617)

`handle_message()` 将后端原始消息（带 `msg` 字段）映射为前端内部事件类型：

| 后端 msg | 前端 type | 处理 |
|----------|-----------|------|
| `estimation` | `update` | status.stage = "queued" / "pending" |
| `process_starts` | `update` | status.stage = "pending" |
| `progress` | `update` | status.stage = "running"，带进度条 |
| `process_generating` | `generating` | status.stage = "running"，**data 可能为增量 diff** |
| `process_streaming` | `streaming` | status.stage = "running" |
| `process_completed` | `complete` + data | 最后输出，stage = "complete" |
| `log` | `log` | 触发日志事件 |
| `unexpected_error` / `broken_connection` | 同左 | stage = "error" |
| `heartbeat` | — | 直接忽略 |

#### 增量 Diff：`apply_diff_stream`

对于普通 generator（非 `streaming` 连接 + sse_v2+ 协议），中间 chunk 只发送 diff 而非全量：

```typescript
// stream.ts L101-L178
if (type === "generating" && dependency.connection !== "stream") {
    apply_diff_stream(pending_diff_streams, event_id!, data);  // ★ 增量合并
}
```

diff 格式为 `[action, path, value][]`，支持的操作：
- `replace`：`target[path] = value`
- `append`：`target[path] += value`（字符串拼接，常用）
- `add`：数组插入 / 对象赋值
- `delete`：数组移除 / 对象删除

> 这样 LLM 文本流式输出时每个 SSE 消息只有几个字节，大大减轻网络压力。

### 5.4 最终渲染：`fire_event` → 组件更新

```
callback(_data)
    └─ handle_message(_data) → {type, status, data}
         ├─ fire_event({type:"status", ...})    → UI 状态条更新
         ├─ fire_event({type:"log", ...})       → Toast 提示
         └─ fire_event({type:"data", data})
               └─ data.data 经过 handle_payload 后，更新对应 output 组件的值
                     └─ Svelte 响应式绑定 → DOM 重渲染
```

---

## 六、双向 Streaming 特殊路径（connection="stream"）

当 `BlockFunction.connection == "stream"`（如实时语音识别、WebRTC 等）时，流程有额外分支：

### 前端 → 后端推送（routes.py L1088-L1102）

```
POST /stream/{event_id}       → event.data = body; event.signal.set()
POST /stream/{event_id}/close → event.closed = True; signal.set()
```

### 后端等待前端数据（queueing.py L926-L935）

```python
# process_events 循环内部
if awake_events[0].streaming:
    awake_events, closed_events = await Queue.wait_for_batch(
        awake_events,
        [fn.time_limit or 30 - first_iteration] * len(awake_events)
    )
    # wait_for_batch 内部 await event.signal.wait()
    # 直到前端 POST /stream/{event_id} 触发 signal.set()
```

### 前端发送方法

submit.ts L690-L703 返回的 iterator 带：
```typescript
send_chunk: (payload) => {
    this.post_data(`/stream/${event_id_final}`, {...payload, session_hash})
},
close_stream: () => {
    this.post_data(`/stream/${event_id_final}/close`, {})
}
```

---

## 七、完整时序图（以 `yield` generator + sse_v3 协议为例）

```
前端 (JS)                                  后端 (Python)
   │                                           │
   │  POST /queue/join {fn_index, data,...}    │
   │──────────────────────────────────────────►│  Queue.push() → 创建 Event
   │                                           │   入队 + 广播 EstimationMessage
   │◄──────────────────────────────────────────│
   │       {event_id: "abc123"}                │  放入 AsyncQueue
   │                                           │
   │  register callback[abc123]                │
   │  open_stream() 若未打开                   │
   │                                           │
   │  GET /queue/data?session_hash=xxx         │
   │◄──────────────────────────────────────────│
   │   (SSE 长连接建立)                        │
   │                                           │  sse_stream() 开始循环：
   │                                           │    await messages.get() → EstimationMessage
   │◄── data: {msg:"estimation", rank:3, ...} │    yield → 前端 onmessage
   │                                           │
   │  (event_callbacks[abc123] 分发)           │
   │  → status: queued                         │
   │                                           │
   │                ... 排队中 ...             │
   │                                           │
   │◄── data: {msg:"process_starts", ...}     │  ProcessStartsMessage
   │  → status: pending → running              │
   │                                           │
   │                                           │  call_process_api() → generator.__anext__()
   │                                           │  send_message(ProcessGeneratingMessage)
   │◄── data: {msg:"process_generating",      │
   │           output:{data:["Hel"],...}}      │
   │  → apply_diff → fire "data" event         │
   │  → Textbox 显示 "Hel"                     │
   │                                           │
   │◄── data: {msg:"process_generating",      │  再次 __anext__()
   │           output:{diff:[[append,[0],"lo"]...}}│
   │  → apply_diff("Hel"+"lo") → "Hello"       │
   │                                           │
   │                ... 更多 chunk ...         │
   │                                           │
   │                                           │  StopAsyncIteration → is_generating=False
   │                                           │  send_message(ProcessCompletedMessage)
   │◄── data: {msg:"process_completed",       │
   │           output:{data:["Hello World!"],...}}│
   │  → fire "complete" + "data"               │
   │  → Textbox 显示最终结果                    │
   │                                           │
   │  pending_event_ids_session[abc] 清空      │
   │◄── data: {msg:"close_stream"}             │  CloseStreamMessage
   │  → close_stream() → abort_controller      │
   │  → SSE 连接关闭                            │
```

---

## 八、关键数据结构速查表

| 结构 | 位置 | 作用 |
|------|------|------|
| `Queue.pending_messages_per_session` | queueing.py L126-L128 | `{session_hash: AsyncQueue[EventMessage]}` 每个会话的待发送消息队列 |
| `Queue.pending_event_ids_session` | queueing.py L129 | `{session_hash: {event_id,...}}` 判断 SSE 是否可关闭 |
| `Queue.event_ids_to_events` | queueing.py L130 | `{event_id: Event}` 事件全局索引 |
| `Queue.event_queue_per_concurrency_id` | queueing.py L132 | `{concurrency_id: EventQueue}` 按并发组排队 |
| `App.iterators` | routes.py L238 | `{event_id: AsyncIterator}` 生成器状态持久化 |
| `Client.event_callbacks` | client.ts L65 | `{event_id: callback}` 前端按 event_id 分发消息 |
| `Client.pending_stream_messages` | client.ts L63 | `{event_id: [msg,...]}` 回调未注册时暂存消息 |
| `Client.pending_diff_streams` | client.ts L64 | `{event_id: [accumulated_data,...]}` 增量 diff 合并状态 |

---

## 九、涉及文件索引

**后端（Python）：**
- [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/queueing.py) — Queue 类、Event 生命周期、消息循环
- [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/routes.py) — `/queue/join`、`/queue/data`、`/call/...`、SSE StreamingResponse
- [server_messages.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/server_messages.py) — 所有 EventMessage 类型定义
- [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/route_utils.py) — `call_process_api` 桥接
- [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/gradio/blocks.py) — `process_api` 实际执行用户函数 + generator 迭代

**前端（TypeScript）：**
- [client/js/src/utils/submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/submit.ts) — submit() 主流程、多协议分支、callback 逻辑
- [client/js/src/utils/stream.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/utils/stream.ts) — `open_stream`、`readable_stream`、`apply_diff_stream`
- [client/js/src/client.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/client.ts) — Client 类、全局流状态、session_hash
- [client/js/src/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/248-gradio/client/js/src/constants.ts) — SSE_URL = `queue/data`、SSE_DATA_URL = `queue/join`
