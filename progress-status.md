# Gradio 进度条与状态 API 消息流向分析

## 总览

Gradio 的进度与状态系统是一条从 **用户函数** → **后端队列** → **SSE 通道** → **JS 客户端** → **Svelte 组件** 的单向消息流。整条链路的核心数据载体是 `EventMessage` 联合类型，其中与进度/状态直接相关的子类型有：

| 消息类型 (`msg` 字段) | 数据类 | 含义 |
|---|---|---|
| `estimation` | `EstimationMessage` | 队列排队位置与 ETA |
| `process_starts` | `ProcessStartsMessage` | 函数开始执行 |
| `progress` | `ProgressMessage` | 进度条更新 |
| `log` | `LogMessage` | 日志 Toast |
| `process_generating` | `ProcessGeneratingMessage` | 生成器/流式中间输出 |
| `process_completed` | `ProcessCompletedMessage` | 执行完成 |
| `heartbeat` | `HeartbeatMessage` | SSE 心跳 |

---

## 第一层：进度上下文的创建与注入

### 1.1 用户侧入口 —— `gr.Progress()`

用户在函数签名中声明 `progress=gr.Progress()` 即可启用进度跟踪。[helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/helpers.py#L672-L702)

```python
class Progress(Iterable):
    def __init__(self, track_tqdm=False):
        self.iterables: list[TrackedIterable] = []

    def __call__(self, progress, desc=None, total=None, unit="steps"):
        callback = self._progress_callback()
        if callback:
            callback(self.iterables + [TrackedIterable(...)])

    def tqdm(self, iterable, desc=None, ...):
        callback = self._progress_callback()
        if callback:
            self.iterables.append(TrackedIterable(iter(iterable), 0, length, ...))
            return self

    @staticmethod
    def _progress_callback():
        blocks = LocalContext.blocks.get(None)
        event_id = LocalContext.event_id.get(None)
        if not (blocks and event_id):
            return None
        return partial(blocks._queue.set_progress, event_id)
```

关键点：`_progress_callback()` 通过 `LocalContext`（线程/协程安全的 `ContextVar`）获取当前 `Blocks` 实例和 `event_id`，返回的 callback 直接绑定到 `Queue.set_progress`。[context.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/context.py#L22-L34)

### 1.2 识别 Progress 参数 —— `special_args()`

[helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/helpers.py#L918-L958)

```python
def special_args(fn, inputs, request, event_data, ...):
    for i, param in enumerate(positional_args):
        if isinstance(param.default, Progress):
            progress_index = i
            inputs.insert(i, param.default)   # 将 gr.Progress() 实例插入输入
```

当 `Blocks.call_function()` 调用 `special_args()` 时，它会检测函数参数中默认值为 `Progress` 实例的参数，并记录其位置索引 `progress_index`。

### 1.3 包装与上下文注入 —— `call_function()` → `create_tracker()`

[blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/blocks.py#L1636-L1649)

```python
processed_input, progress_index, _, _ = special_args(fn_to_analyze, processed_input, ...)
progress_tracker = processed_input[progress_index] if progress_index is not None else None

if progress_tracker is not None and progress_index is not None:
    progress_tracker, fn = create_tracker(fn, progress_tracker.track_tqdm)
    processed_input[progress_index] = progress_tracker
```

`create_tracker()` 做两件事（[helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/helpers.py#L905-L915)）：

1. 创建一个新的 `Progress` 实例
2. 如果 `track_tqdm=True`，用 `function_wrapper` 把函数包装，在执行前通过 `LocalContext.progress.set(progress)` 设置上下文，执行后通过 `LocalContext.progress.set(None)` 清除

此外，`get_function_with_locals()` 会设置更广泛的上下文（[utils.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/utils.py#L1077-L1099)）：

```python
def before_fn(blocks, event_id):
    LocalContext.blocks.set(blocks)
    LocalContext.in_event_listener.set(in_event_listener)
    LocalContext.event_id.set(event_id)
    LocalContext.request.set(request)
```

这样，当用户函数内部调用 `progress(0.5, desc="Processing")` 时，`_progress_callback()` 就能通过 `LocalContext` 拿到 `blocks` 和 `event_id`，从而调用 `blocks._queue.set_progress(event_id, iterables)`。

---

## 第二层：后端队列的消息生成与发送

### 2.1 `Queue.set_progress()` —— 将 TrackedIterable 写入 Event

[queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/queueing.py#L585-L608)

```python
def set_progress(self, event_id, iterables):
    for job in self.active_jobs:
        if job is None:
            continue
        for evt in job:
            if evt._id == event_id:
                progress_data = [ProgressUnit(
                    index=i.index, length=i.length,
                    unit=i.unit, progress=i.progress, desc=i.desc
                ) for i in iterables]
                evt.progress = ProgressMessage(progress_data=progress_data)
                evt.progress_pending = True
```

**重要设计**：进度更新并不立即发送，而是写入 `evt.progress` 并标记 `evt.progress_pending = True`。这是一个**节流（throttle）机制**——进度更新可能非常频繁，连续的更新之间只保留最新的一个。

### 2.2 `Queue.start_progress_updates()` —— 定时轮询发送

[queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/queueing.py#L565-L583)

```python
async def start_progress_updates(self):
    while not self.stopped:
        events = [evt for job in self.active_jobs if job is not None for evt in job]
        for event in events:
            if event.progress_pending and event.progress:
                event.progress_pending = False
                self.send_message(event, event.progress)
        await asyncio.sleep(self.progress_update_sleep_when_free)
        # Windows: 0.1s, 其他: 0.01s
```

以固定间隔检查所有活跃事件，有挂起的进度就发送。这确保了高频进度更新不会淹没 SSE 通道。

### 2.3 `Queue.send_message()` —— 写入会话消息队列

[queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/queueing.py#L240-L249)

```python
def send_message(self, event, event_message):
    if not event.alive:
        return
    event_message.event_id = event._id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```

所有消息（包括进度、状态、完成等）都写入 `pending_messages_per_session[session_hash]`——一个按会话哈希分组的 `AsyncQueue`。

### 2.4 其他状态消息的发送时机

| 消息 | 发送位置 | 触发条件 |
|---|---|---|
| `EstimationMessage` | `broadcast_estimations()` | 事件入队后、处理开始后、定时通知 |
| `ProcessStartsMessage` | `process_events()` | 事件开始执行时 |
| `ProgressMessage` | `start_progress_updates()` | 进度更新 pending 且定时轮询到 |
| `ProcessGeneratingMessage` | `process_events()` | 生成器/流式函数产出中间结果 |
| `ProcessCompletedMessage` | `process_events()` | 函数执行完成或出错 |
| `LogMessage` | `log_message()` | 后端主动调用 |

### 2.5 SSE 端点 —— 从 AsyncQueue 到 HTTP 流

[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/gradio/routes.py#L1490-L1557)

```python
async def sse_stream(request: fastapi.Request):
    while True:
        if await request.is_disconnected():
            await blocks._queue.clean_events(session_hash=session_hash)
            return
        messages = blocks._queue.pending_messages_per_session[session_hash]
        message = await asyncio.wait_for(messages.get(), timeout=10)
        if message:
            response = process_msg(message)   # 序列化为 SSE 格式
            if response is not None:
                yield response
```

对于内部前端（`/queue/data`），`process_msg` 直接用 `orjson` 序列化完整 `EventMessage`；对于外部 API（`/call/{api_name}/{event_id}`），仅提取简化的事件类型和数据。

SSE 数据格式：

```
data: {"msg":"progress","event_id":"abc123","progress_data":[{"index":5,"length":10,"unit":"steps","progress":null,"desc":"Loading"}]}\n\n
```

---

## 第三层：JS 客户端的消息解析与分发

### 3.1 `submit()` —— 建立 SSE 连接

[client/js/src/utils/submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/client/js/src/utils/submit.ts#L266-L394)

对于 `sse_v1/v2/v3` 协议（最新 API 格式），流程为：

1. `POST /queue/join` 获取 `event_id`
2. 打开 SSE 流 `/queue/data?session_hash=xxx`
3. 注册 `event_callbacks[event_id]` 回调

```typescript
event_callbacks[event_id] = callback;
```

SSE 消息到达时，通过 `handle_message()` 解析后调用对应回调。

### 3.2 `handle_message()` —— 消息类型映射

[client/js/src/helpers/api_info.ts](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/client/js/src/helpers/api_info.ts#L234-L404)

核心映射表：

| `data.msg` (后端) | 返回 `type` (前端) | 前端 `status.stage` |
|---|---|---|
| `estimation` | `"update"` | `"pending"`（保持当前） |
| `progress` | `"update"` | `"pending"` |
| `process_starts` | `"update"` | `"pending"` |
| `process_generating` | `"generating"` | `"generating"` |
| `process_streaming` | `"streaming"` | `"streaming"` |
| `process_completed` | `"complete"` | `"complete"` / `"error"` |
| `log` | `"log"` | — |
| `heartbeat` | `"heartbeat"` | — |

**关键**：`progress` 消息被映射为 `type: "update"`，`status.stage: "pending"`，`status.progress_data` 携带进度详情。

### 3.3 回调内 —— `fire_event()` 推入异步迭代器

[submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/client/js/src/utils/submit.ts#L489-L618)

```typescript
if (type === "update" && status && !complete) {
    fire_event({ type: "status", endpoint, fn_index, time, ...status });
}
```

`fire_event()` 将 `GradioEvent` 推入异步迭代器的值队列。前端 `DependencyManager` 通过 `for await (const result of dep_submission.data)` 消费这些事件。

---

## 第四层：前端 DependencyManager 与 LoadingStatus

### 4.1 DependencyManager 消费事件

[js/core/src/dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/core/src/dependency.ts#L431-L612)

```typescript
submit_loop: for await (const result of dep_submission.data) {
    if (result.type === "status") {
        if (result.stage === "complete") {
            this.loading_stati.update({
                ...status, status: status.stage,
                fn_index: dep.id, stream_state
            });
            this.update_loading_stati_state();
            break submit_loop;
        } else if (result.stage === "generating") {
            this.loading_stati.update({
                ...status, status: status.stage,
                fn_index: dep.id, stream_state
            });
            this.update_loading_stati_state();
        } else {
            // pending / estimation / progress 等
            this.loading_stati.update({
                ...status, status: status.stage,
                fn_index: dep.id, stream_state
            });
            this.update_loading_stati_state();
        }
    }
}
```

**所有** 状态更新（包括进度）都通过 `loading_stati.update()` + `update_loading_stati_state()` 统一处理。

### 4.2 LoadingStatus —— 将 fn_index 映射到组件 ID

[js/statustracker/static/state.svelte.ts](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/statustracker/static/state.svelte.ts#L1-L166)

```typescript
class LoadingStatus {
    fn_outputs: Record<number, number[]> = {};   // fn_index → output component IDs
    fn_inputs: Record<number, number[]> = {};    // fn_index → input component IDs
    current: Record<string, ILoadingStatus> = {}; // component_id → 当前状态

    register(dependency_id, outputs, inputs, show_progress) {
        this.fn_outputs[dependency_id] = outputs;
        this.fn_inputs[dependency_id] = inputs;
        this.show_progress[dependency_id] = show_progress;
    }

    update(args: LoadingStatusArgs) {
        const updates = this.resolve_args(args);
        updates.forEach(({ id, status, progress, ... }) => {
            this.current[id] = { status, progress, ... };
        });
    }
}
```

`resolve_args()` 将 `fn_index` 解析为具体的 input/output 组件 ID 列表，并为每个组件生成独立的状态更新。`progress_data` 直接透传到组件级别。

### 4.3 `update_loading_stati_state()` —— 推入组件状态

[dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/core/src/dependency.ts#L286-L298)

```typescript
async update_loading_stati_state() {
    for (const [component_id, loading_status] of Object.entries(this.loading_stati.current)) {
        this.update_state_cb(Number(component_id), { loading_status }, false);
    }
}
```

`update_state_cb` 是初始化时传入的回调，最终调用 `init.svelte.ts` 中的组件状态更新方法。注意 `loading_status` 被特殊处理——它不会缓存到 `pending_updates` 中（因为是瞬时状态）。

---

## 第五层：Svelte 组件的渲染

### 5.1 BaseColumn.svelte —— StatusTracker 的挂载点

[js/column/BaseColumn.svelte](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/column/BaseColumn.svelte#L32-L43)

```svelte
{#if loading_status && loading_status.show_progress}
    <StatusTracker
        autoscroll={props.autoscroll}
        i18n={props.i18n}
        {...loading_status}
        status={loading_status
            ? loading_status.status == "pending"
                ? "generating"
                : loading_status.status
            : null}
    />
{/if}
```

**关键转换**：当后端状态为 `"pending"` 时，前端显示为 `"generating"`（因为在用户视角，pending 意味着函数正在运行）。

### 5.2 StatusTracker —— 进度条与状态信息的渲染

[js/statustracker/static/index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/statustracker/static/index.svelte)

**进度条计算逻辑**（[index.svelte#L188-L222](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/statustracker/static/index.svelte#L188-L222)）：

```typescript
let progress_level = $derived.by(() => {
    if (progress != null) {
        _progress_level = progress.map(p => {
            if (p.index != null && p.length != null) {
                return p.index / p.length;    // 有 index+length → 比例
            } else if (p.progress != null) {
                return p.progress;            // 有 progress → 直接用 0~1
            }
            return undefined;
        });
    }
    // ...
});
```

**显示逻辑**（[index.svelte#L364-L431](file:///d:/fz/0601/solo-dogfeeding/code/259-gradio/js/statustracker/static/index.svelte#L364-L431)）：

```
status === "pending" (前端显示为 "generating"):
  ├─ 有 progress_data → 显示进度条 + 百分比 + 描述
  │   ├─ 每个 progress item: desc + (index/length 或 progress%)
  │   └─ 最内层进度条的宽度 = last_progress_level * 100%
  ├─ 无 progress_data 但有 ETA → 显示 ETA 进度条
  └─ 无 progress 且 show_progress="full" → 显示 Loader spinner
```

**ETA 计时器**：当 `status === "pending"` 时，启动 `requestAnimationFrame` 循环计时，显示经过时间与预估时间的比例。

---

## 完整消息流向图

```
用户函数
  │  progress(0.5, desc="Processing")
  │  或 for i in progress.tqdm(range(10)):
  ▼
Progress._progress_callback()
  │  通过 LocalContext.blocks / LocalContext.event_id 获取上下文
  │  → 调用 blocks._queue.set_progress(event_id, iterables)
  ▼
Queue.set_progress()
  │  将 TrackedIterable 列表转换为 ProgressUnit 列表
  │  写入 evt.progress = ProgressMessage(...)
  │  设置 evt.progress_pending = True（节流标记）
  ▼
Queue.start_progress_updates()  ← 异步循环，每 10~100ms 轮询
  │  检测 progress_pending=True 的事件
  │  调用 send_message(event, event.progress)
  ▼
Queue.send_message()
  │  写入 pending_messages_per_session[session_hash] AsyncQueue
  ▼
SSE 端点 /queue/data  ← FastAPI StreamingResponse
  │  asyncio.wait_for(messages.get(), timeout=10)
  │  序列化为 "data: {orjson_dumps}\n\n"
  ▼
JS Client: handle_message()
  │  data.msg === "progress"
  │  → type: "update", status: { stage: "pending", progress_data: [...] }
  ▼
JS Client: fire_event()
  │  推入 GradioEvent { type: "status", stage: "pending", progress_data: [...] }
  ▼
DependencyManager: for await (result of submission)
  │  result.type === "status"
  │  → loading_stati.update({ status, fn_index, progress_data })
  │  → update_loading_stati_state()
  ▼
LoadingStatus.update()
  │  将 fn_index 解析为 input/output 组件 ID
  │  写入 current[component_id] = { status, progress, ... }
  ▼
DependencyManager.update_loading_stati_state()
  │  遍历 current，逐组件调用 update_state_cb(id, { loading_status })
  ▼
init.svelte.ts: 组件状态更新
  │  loading_status 不缓存（瞬时属性），直接调用 _set_data()
  ▼
BaseColumn.svelte
  │  {loading_status} → <StatusTracker {...loading_status} />
  ▼
StatusTracker (index.svelte)
  │  status="generating" (pending→generating 转换)
  │  progress_data → 计算进度条宽度 + 显示百分比和描述
  │  无 progress_data → 显示 ETA 进度条 或 Loader spinner
  ▼
用户看到的进度条
```

---

## 其他状态消息的流程（简述）

### 队列排队状态

```
Queue.push() → broadcast_estimations()
  → EstimationMessage(rank, rank_eta, queue_size)
  → SSE → handle_message() → type:"update", stage:"pending", size/position/eta
  → LoadingStatus.update() → StatusTracker 显示 "queue: 2/5 | 3.2s"
```

### 函数开始

```
Queue.process_events() → ProcessStartsMessage(eta)
  → SSE → handle_message() → type:"update", stage:"pending", position:0
  → LoadingStatus.update() → StatusTracker 显示 "processing | 2.1s"
```

### 生成器中间输出

```
Queue.process_events() → ProcessGeneratingMessage(output, success)
  → SSE → handle_message() → type:"generating", stage:"generating"
  → DependencyManager 同时处理 data 更新和 loading_stati 更新
```

### 执行完成

```
Queue.process_events() → ProcessCompletedMessage(output, success, used_cache, ...)
  → SSE → handle_message() → type:"complete", stage:"complete"
  → LoadingStatus.update({ status:"complete" })
  → StatusTracker 隐藏（should_hide = true）
  → 若有缓存，显示 cache indicator（⚡ from cache: 0.3s）
```

### 日志消息

```
Queue.log_message() → LogMessage(log, level, title, duration, visible)
  → SSE → handle_message() → type:"log"
  → DependencyManager.handle_log() → log_cb() → Toast 组件显示
```

---

## 关键设计模式总结

1. **ContextVar 隔离**：`LocalContext.blocks` / `LocalContext.event_id` 使用 Python `ContextVar` 确保多线程/协程环境下每个请求的进度回调指向正确的 Queue 和 Event。

2. **进度节流**：`evt.progress_pending` 标志 + `start_progress_updates()` 定时轮询，避免高频进度更新撑爆 SSE 通道。连续更新之间只保留最新值。

3. **会话隔离**：`pending_messages_per_session[session_hash]` 按浏览器会话分组，SSE 端点仅推送该会话的消息。

4. **fn_index → component_id 映射**：`LoadingStatus` 将后端的 `fn_index` 维度转换为前端的 `component_id` 维度，使得一个函数的进度/状态可以映射到多个输出组件。

5. **瞬时属性分离**：`loading_status` 在 `init.svelte.ts` 中被排除在 `pending_updates` 缓存之外，防止组件延迟挂载时加载状态覆盖已完成状态。

6. **状态语义转换**：后端的 `"pending"` 在前端组件层被转换为 `"generating"`，更符合用户认知（函数正在运行而非等待）。

7. **show_progress 控制**：`BlockFunction.show_progress`（"full"/"minimal"/"hidden"）通过依赖配置传递到前端，控制进度条是完整显示、仅显示计时器还是完全隐藏。
