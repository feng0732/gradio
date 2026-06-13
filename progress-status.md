# Gradio 进度条与状态 API 消息流向分析

> 本文档所有结论均附可在仓库中直接复核的代码引用（相对路径 + 精确行号），路径均相对于项目根目录。

---

## 总览

Gradio 的进度与状态系统是一条从 **用户函数** → **后端队列** → **SSE 通道** → **JS 客户端** → **前端组件树** 的单向消息流。整条链路的核心数据载体是 `EventMessage` 联合类型，后端在 [server_messages.py](gradio/server_messages.py) 中定义了全部消息类。

| 后端 `msg` 字段 | 对应数据类（定义位置） | 前端映射 | 含义 |
|---|---|---|---|
| `estimation` | `EstimationMessage` [server_messages.py#L40-L46](gradio/server_messages.py#L40-L46) | `type:"update", stage:"pending"` | 队列排队位置与 ETA |
| `process_starts` | `ProcessStartsMessage` [server_messages.py#L53-L56](gradio/server_messages.py#L53-L56) | `type:"update", stage:"pending"` | 函数开始执行 |
| `progress` | `ProgressMessage` [server_messages.py#L58-L62](gradio/server_messages.py#L58-L62) | `type:"update", stage:"pending"` | 进度条更新 |
| `log` | `LogMessage` [server_messages.py#L64-L72](gradio/server_messages.py#L64-L72) | `type:"log"` | 日志 Toast |
| `process_generating` | `ProcessGeneratingMessage` [server_messages.py#L74-L80](gradio/server_messages.py#L74-L80) | `type:"generating"` | 生成器/流式中间输出 |
| `process_completed` | `ProcessCompletedMessage` [server_messages.py#L82-L94](gradio/server_messages.py#L82-L94) | `type:"complete"` | 执行完成 |
| `heartbeat` | `HeartbeatMessage` [server_messages.py#L129-L131](gradio/server_messages.py#L129-L131) | `type:"heartbeat"` | SSE 心跳 |

---

## 第一层：进度上下文的创建与注入

### 1.1 用户侧入口 —— `gr.Progress()`

用户在函数签名中声明 `progress=gr.Progress()` 即可启用进度跟踪。`Progress` 类定义在 [helpers.py#L672-L823](gradio/helpers.py#L672-L823)，关键方法：

**构造函数** —— [helpers.py#L691-L702](gradio/helpers.py#L691-L702)：
```python
def __init__(self, track_tqdm: bool = False):
    if track_tqdm:
        patch_tqdm()
    self.track_tqdm = track_tqdm
    self.iterables: list[TrackedIterable] = []
```

**`__call__` 触发进度更新** —— [helpers.py#L736-L764](gradio/helpers.py#L736-L764)：
```python
def __call__(self, progress, desc=None, total=None, unit="steps", _tqdm=None):
    callback = self._progress_callback()
    if callback:
        if isinstance(progress, tuple):
            index, total = progress
            progress = None
        else:
            index = None
        callback(self.iterables + [TrackedIterable(None, index, total, desc, unit, _tqdm, progress)])
```

**`_progress_callback` 核心** —— [helpers.py#L804-L823](gradio/helpers.py#L804-L823)：
```python
@staticmethod
def _progress_callback():
    blocks = LocalContext.blocks.get(None)
    event_id = LocalContext.event_id.get(None)
    if not (blocks and event_id):
        return None
    return partial(blocks._queue.set_progress, event_id)
```

通过 `LocalContext`（Python `ContextVar`，线程/协程安全）获取当前 `Blocks` 实例和 `event_id`，返回的 callback 直接绑定到 `Queue.set_progress`。`LocalContext` 定义在 [context.py#L22-L34](gradio/context.py#L22-L34)：
```python
class LocalContext:
    blocks: ContextVar = ContextVar("blocks", default=None)
    event_id: ContextVar = ContextVar("event_id", default=None)
    progress: ContextVar = ContextVar("progress", default=None)
    request: ContextVar = ContextVar("request", default=None)
    ...
```

**`tqdm()` 包装迭代器** —— [helpers.py#L766-L802](gradio/helpers.py#L766-L802)：
```python
def tqdm(self, iterable, desc=None, total=None, unit="steps", _tqdm=None):
    callback = self._progress_callback()
    if callback:
        length = len(iterable) if hasattr(iterable, "__len__") else total
        new_iterable = TrackedIterable(iter(iterable), 0, length, desc, unit, _tqdm)
        self.iterables.append(new_iterable)
        callback(self.iterables)
        return self
```

`TrackedIterable` 数据结构 —— [helpers.py#L649-L669](gradio/helpers.py#L649-L669)：
```python
@dataclass
class TrackedIterable:
    iterable: Iterable | None
    index: int | float | None
    length: int | float | None
    desc: str | None
    unit: str | None
    _tqdm = None
    progress: float | None = None
```

### 1.2 识别 Progress 参数 —— `special_args()`

[helpers.py#L918-L962](gradio/helpers.py#L918-L962) 中 `special_args()` 扫描函数参数，检测默认值为 `Progress` 实例的参数：
```python
def special_args(fn, inputs, request, event_data, ...):
    for i, param in enumerate(positional_args):
        if isinstance(param.default, Progress):
            progress_index = i
            inputs.insert(i, param.default)   # 将 gr.Progress() 实例插入输入列表
```

### 1.3 包装与上下文注入 —— `call_function()` → `create_tracker()`

[blocks.py#L1582-L1740](gradio/blocks.py#L1582-L1740) 的 `call_function()` 是调用用户函数的入口：

**识别并注入 Progress** —— [blocks.py#L1633-L1653](gradio/blocks.py#L1633-L1653)：
```python
processed_input, progress_index, _, _ = special_args(fn_to_analyze, processed_input, ...)
progress_tracker = processed_input[progress_index] if progress_index is not None else None

if progress_tracker is not None and progress_index is not None:
    progress_tracker, fn = create_tracker(fn, progress_tracker.track_tqdm)
    processed_input[progress_index] = progress_tracker
```

**`create_tracker()` 包装逻辑** —— [helpers.py#L895-L916](gradio/helpers.py#L895-L916)：
```python
def create_tracker(func, track_tqdm: bool):
    progress = Progress()
    if track_tqdm:
        def function_wrapper(*args, **kwargs):
            LocalContext.progress.set(progress)
            try:
                return func(*args, **kwargs)
            finally:
                LocalContext.progress.set(None)
        return progress, function_wrapper
    return progress, func
```

**设置 LocalContext —— `get_function_with_locals()`** —— [utils.py#L1070-L1103](gradio/utils.py#L1070-L1103)：
```python
def before_fn(blocks, event_id, request=None, in_event_listener=False):
    LocalContext.blocks.set(blocks)
    LocalContext.in_event_listener.set(in_event_listener)
    LocalContext.event_id.set(event_id)
    LocalContext.request.set(request)
```

**`_id` 与 `session_hash` 设置** —— [route_utils.py#L362-L512](gradio/route_utils.py#L362-L512) 的 `call_process_api()`：
```python
async def call_process_api(...):
    ...
    with set_space_token(space_token):
        output = await blocks.process_api(
            fn_index, inputs, request, username, session_hash, event_id, ...
        )
```
最终在 `blocks.process_api()` 中通过 `get_function_with_locals()` 设置好 `LocalContext.event_id`，`event_id` 即事件的唯一 ID，由 [queueing.py Event 类](gradio/queueing.py) `_id` 字段保存。

---

## 第二层：后端队列的消息生成与发送

### 2.1 `Queue.set_progress()` —— 将 TrackedIterable 写入 Event（节流）

[queueing.py#L585-L608](gradio/queueing.py#L585-L608)：
```python
def set_progress(self, event_id: str, iterables: list[TrackedIterable]) -> None:
    for job in self.active_jobs:
        if job is None:
            continue
        for evt in job:
            if evt._id == event_id:
                progress_data = [
                    ProgressUnit(
                        index=i.index, length=i.length,
                        unit=i.unit, progress=i.progress, desc=i.desc
                    )
                    for i in iterables
                ]
                evt.progress = ProgressMessage(progress_data=progress_data)
                evt.progress_pending = True   # ⚠️ 节流标记：不立即发送
```

**关键设计（节流机制）**：进度更新并不立即发送，而是写入 `evt.progress` 并标记 `evt.progress_pending = True`。如果用户 1ms 内调用 100 次 `progress()`，只保留最后一次。

`ProgressUnit` 数据类定义在 [server_messages.py#L114-L122](gradio/server_messages.py#L114-L122)，`ProgressMessage` 在 [server_messages.py#L58-L62](gradio/server_messages.py#L58-L62)：
```python
@dataclass
class ProgressMessage(EventMessage):
    msg: Literal["progress"] = "progress"
    progress_data: list[ProgressUnit] = field(default_factory=list)
```

### 2.2 `Queue.start_progress_updates()` —— 定时轮询发送

[queueing.py#L555-L583](gradio/queueing.py#L555-L583)：
```python
async def start_progress_updates(self) -> None:
    while not self.stopped:
        events = [evt for job in self.active_jobs if job is not None for evt in job]
        for event in events:
            if event.progress_pending and event.progress:
                event.progress_pending = False
                self.send_message(event, event.progress)   # 真正发送
        await asyncio.sleep(self.progress_update_sleep_when_free)
        # Windows: 0.1s，其他平台: 0.01s（Queue 构造函数中设置）
```

### 2.3 `Queue.send_message()` —— 写入会话消息队列

[queueing.py#L240-L249](gradio/queueing.py#L240-L249)：
```python
def send_message(self, event: Event, event_message: EventMessage):
    if not event.alive:
        return
    event_message.event_id = event._id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```

所有消息（进度、状态、完成、日志等）统一写入 `pending_messages_per_session[session_hash]`——一个按会话哈希分组的 `asyncio.Queue`。该字典在 [queueing.py Queue.__init__](gradio/queueing.py) 中初始化为 `defaultdict(asyncio.Queue)`。

### 2.4 其他状态消息的发送时机（附精确行号）

| 消息类 | 发送函数/位置 | 触发条件 |
|---|---|---|
| `EstimationMessage` | `Queue.broadcast_estimations()` [queueing.py#L410-L476](gradio/queueing.py#L410-L476) | 事件入队后、处理开始后、定时通知 |
| `ProcessStartsMessage` | `Queue.process_events()` [queueing.py#L947-L962](gradio/queueing.py#L947-L962) | 事件出队开始执行时 |
| `ProgressMessage` | `Queue.start_progress_updates()` [queueing.py#L555-L583](gradio/queueing.py#L555-L583) | 进度 `progress_pending=True` 且定时轮询到 |
| `ProcessGeneratingMessage` | `Queue.process_events()` [queueing.py#L995-L1008](gradio/queueing.py#L995-L1008) | 生成器/流式函数 `yield` 中间结果 |
| `ProcessCompletedMessage` | `Queue.process_events()` [queueing.py#L1059-L1107](gradio/queueing.py#L1059-L1107) | 函数执行完成或异常返回 |
| `LogMessage` | `Queue.log_message()` [queueing.py#L611-L625](gradio/queueing.py#L611-L625) | 后端主动调用 |

### 2.5 SSE 端点 —— 从 AsyncQueue 到 HTTP 流

[routes.py#L1490-L1557](gradio/routes.py#L1490-L1557)：

```python
async def sse_stream(request: fastapi.Request):
    ...
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

对于内部前端（`/queue/data`），`process_msg` 直接用 `orjson.dumps()` 序列化完整 `EventMessage`（见 [routes.py 中 process_msg 函数](gradio/routes.py) `sse_stream` 内的 `process_msg` 闭包）。

**实际 SSE 数据格式**：
```
data: {"msg":"progress","event_id":"abc123","progress_data":[{"index":5,"length":10,"unit":"steps","progress":null,"desc":"Loading"}]}

data: {"msg":"process_completed","event_id":"abc123","output":{"data":[{"data":"result"}]}, "success": true}

```

对于外部 API `/call/{api_name}/{event_id}`，消息会被转换为简化格式（见 `routes.py` 中 `/call` 路由内的 `process_msg_api` 闭包），仅包含 `msg` 字段和核心数据。

---

## 第三层：JS 客户端的消息解析与分发

### 3.1 `submit()` —— 建立 SSE 连接

[client/js/src/utils/submit.ts#L266-L394](client/js/src/utils/submit.ts#L266-L394)

对于 `sse_v1/v2/v3` 协议（最新 API 格式），流程为：

1. `POST /queue/join` —— [submit.ts#L333-L370](client/js/src/utils/submit.ts#L333-L370)，获取 `event_id`
2. 打开 SSE 流 `/queue/data?session_hash=xxx` —— [submit.ts#L372-L394](client/js/src/utils/submit.ts#L372-L394)
3. 注册 `event_callbacks[event_id]` 回调 —— [submit.ts#L385](client/js/src/utils/submit.ts#L385)：
```typescript
event_callbacks[event_id] = callback;
```

### 3.2 `handle_message()` —— 消息类型映射

[client/js/src/helpers/api_info.ts#L234-L404](client/js/src/helpers/api_info.ts#L234-L404)

核心映射代码（进度消息处理位于 [api_info.ts#L252-L280](client/js/src/helpers/api_info.ts#L252-L280)）：
```typescript
case "progress": {
    status = {
        ...status,
        stage: "pending" as const,
        progress_data: data.progress_data,
        ...
    };
    break;
}
```

完整映射表：

| 后端 `data.msg` | 返回 `type` | 前端 `status.stage` | 处理位置 |
|---|---|---|---|
| `estimation` | `"update"` | `"pending"` | [api_info.ts#L240-L251](client/js/src/helpers/api_info.ts#L240-L251) |
| `progress` | `"update"` | `"pending"` | [api_info.ts#L252-L280](client/js/src/helpers/api_info.ts#L252-L280) |
| `process_starts` | `"update"` | `"pending"` | [api_info.ts#L281-L304](client/js/src/helpers/api_info.ts#L281-L304) |
| `process_generating` | `"generating"` | `"generating"` | [api_info.ts#L305-L351](client/js/src/helpers/api_info.ts#L305-L351) |
| `process_streaming` | `"streaming"` | `"streaming"` | [api_info.ts#L352-L361](client/js/src/helpers/api_info.ts#L352-L361) |
| `process_completed` | `"complete"` | `"complete"`/`"error"` | [api_info.ts#L362-L404](client/js/src/helpers/api_info.ts#L362-L404) |
| `log` | `"log"` | — | [api_info.ts#L405-L414](client/js/src/helpers/api_info.ts#L405-L414) |
| `heartbeat` | `"heartbeat"` | — | [api_info.ts#L415-L418](client/js/src/helpers/api_info.ts#L415-L418) |

### 3.3 回调内 —— `fire_event()` 推入异步迭代器

[submit.ts#L489-L618](client/js/src/utils/submit.ts#L489-L618)

进度消息（`type === "update"`）触发的 `fire_event` 在 [submit.ts#L500-L510](client/js/src/utils/submit.ts#L500-L510)：
```typescript
if (type === "update" && status && !complete) {
    fire_event({ type: "status", endpoint, fn_index, time, ...status });
}
```

`fire_event()` 是 `make_promise_with_events()` 返回的异步迭代器的入队函数，定义在 [submit.ts#L62-L111](client/js/src/utils/submit.ts#L62-L111)。前端 `DependencyManager` 通过 `for await (const result of dep_submission.data)` 消费这些事件。

---

## 第四层：前端 DependencyManager → LoadingStatus → AppTree → 组件

### 4.1 DependencyManager 消费异步事件

**提交循环位置** —— [dependency.ts#L431-L612](js/core/src/dependency.ts#L431-L612)：

```typescript
submit_loop: for await (const result of dep_submission.data) {
    if (result.type === "status") {
        const { fn_index, ...status } = result;

        // ✅ 完成：[dependency.ts#L461-L480](js/core/src/dependency.ts#L461-L480)
        if (result.stage === "complete") {
            this.loading_stati.update({
                ...status, status: status.stage,
                fn_index: dep.id, stream_state: "closed"
            });
            this.update_loading_stati_state();
            break submit_loop;
        }

        // ✅ 生成中：[dependency.ts#L481-L490](js/core/src/dependency.ts#L481-L490)
        else if (result.stage === "generating") {
            this.loading_stati.update({
                ...status, status: status.stage, fn_index: dep.id, stream_state
            });
            this.update_loading_stati_state();
        }

        // ✅ error: [dependency.ts#L491-L558](js/core/src/dependency.ts#L491-L558)
        else if (result.stage === "error") { ... }

        // ✅ pending（进度、排队、开始）：[dependency.ts#L559-L567](js/core/src/dependency.ts#L559-L567)
        else {
            this.loading_stati.update({
                ...status, status: status.stage, fn_index: dep.id, stream_state
            });
            this.update_loading_stati_state();
        }
    }
}
```

**所有** 状态更新（进度/排队/开始/完成/错误）都统一走 `loading_stati.update()` → `update_loading_stati_state()` 流程。

### 4.2 LoadingStatus 注册 —— fn_index → component_id 映射

**注册依赖信息** —— 在 `DependencyManager.register_loading_stati()` [dependency.ts#L271-L279](js/core/src/dependency.ts#L271-L279)：
```typescript
register_loading_stati(deps: Map<number, Dependency>): void {
    for (const [_, dep] of deps) {
        this.loading_stati.register(
            dep.id,
            dep.show_progress_on || dep.outputs,   // 哪些 output 组件显示进度
            dep.inputs,                              // 哪些 input 组件显示进度
            dep.show_progress                        // "full" | "minimal" | "hidden"
        );
    }
}
```

**LoadingStatus.register()** —— [state.svelte.ts#L18-L42](js/statustracker/static/state.svelte.ts#L18-L42)：
```typescript
register(
    dependency_id: number,
    outputs: number[],
    inputs: number[],
    show_progress: boolean | "full" | "minimal"
): void {
    this.fn_outputs[dependency_id] = outputs;
    this.fn_inputs[dependency_id] = inputs;
    this.show_progress[dependency_id] = show_progress;
}
```

`LoadingStatus` 类完整定义在 [state.svelte.ts#L1-L222](js/statustracker/static/state.svelte.ts#L1-L222)，字段结构：
```typescript
class LoadingStatus {
    fn_outputs: Record<number, number[]> = {};    // fn_index → output 组件 IDs
    fn_inputs: Record<number, number[]> = {};     // fn_index → input 组件 IDs
    show_progress: Record<number, ...> = {};      // fn_index → 显示模式
    current: Record<string, ILoadingStatus> = {}; // component_id → 状态对象
}
```

**LoadingStatus.update() → resolve_args()** —— [state.svelte.ts#L44-L92](js/statustracker/static/state.svelte.ts#L44-L92)：

`resolve_args()` 将后端维度的 `fn_index` 转换为前端维度的组件 ID 集合（遍历 `fn_outputs[fn_index]` 和 `fn_inputs[fn_index]`），并为每个组件生成独立的 `ILoadingStatus` 对象（包含 `status`、`progress_data`、`eta`、`size`、`position`、`duration`、`message`、`progress`、`queue`、`show_progress` 等字段）。

**ILoadingStatus 接口定义** —— [types.ts#L1-L52](js/statustracker/static/types.ts#L1-L52)：
```typescript
export interface ILoadingStatus {
    status: "pending" | "error" | "complete" | "generating" | null;
    message?: string;
    queue?: boolean;
    size?: number;
    position?: number;
    eta?: number;
    progress?: number;
    progress_data?: ProgressUnit[];
    ...
    show_progress?: boolean | "full" | "minimal";
}
```

### 4.3 `update_loading_stati_state()` —— 逐组件调用 update_state_cb

[dependency.ts#L286-L298](js/core/src/dependency.ts#L286-L298)：
```typescript
async update_loading_stati_state() {
    for (const [component_id, loading_status] of Object.entries(
        this.loading_stati.current
    )) {
        this.update_state_cb(
            Number(component_id),
            { loading_status: loading_status },
            false   // check_visibility=false，跳过可见性遍历
        );
    }
}
```

### 4.4 Blocks.svelte 绑定 update_state_cb → AppTree.update_state

**DependencyManager 初始化** —— [Blocks.svelte#L232-L241](js/core/src/Blocks.svelte#L232-L241)：
```svelte
let dep_manager = new DependencyManager(
    dependencies,
    app,
    app_tree.update_state.bind(app_tree),   // ← update_state_cb
    app_tree.get_state.bind(app_tree),
    app_tree.rerender.bind(app_tree),
    new_message,
    add_to_api_calls,
    handle_connection_lost
);
```

reload 时的绑定 —— [Blocks.svelte#L255-L261](js/core/src/Blocks.svelte#L255-L261)。

### 4.5 AppTree.update_state() —— 写入组件树节点的 shared_props

[init.svelte.ts#L439-L513](js/core/src/init.svelte.ts#L439-L513)：

**核心路径**（组件已注册回调时走 `_set_data` 分支）：
```typescript
async update_state(id, new_state, check_visibility=true) {
    // ... 处理可见性（略）...

    const _set_data = this.#set_callbacks.get(id);

    if (!_set_data) {
        // 组件未挂载：直接修改 tree 的 props
        const new_props = create_props_shared_props(new_state);  // ← 拆分为 shared_props / props
        for (const key in new_props.shared_props) {
            node!.props.shared_props[key] = new_props.shared_props[key];  // ← 就地修改
        }
        // ⚠️ 瞬时属性分离：loading_status 不缓存
        const { loading_status: _ls, ...rest_new_state } = new_state;  // [init.svelte.ts#L482](js/core/src/init.svelte.ts#L482)
        if (Object.keys(rest_new_state).length > 0) {
            const existing = this.#pending_updates.get(id) || {};
            this.#pending_updates.set(id, { ...existing, ...rest_new_state });
        }
    } else if (_set_data) {
        // ✅ 组件已挂载：直接调用 _set_data（通过 Gradio 类注册的回调）
        _set_data(new_state);   // [init.svelte.ts#L501-L503](js/core/src/init.svelte.ts#L501-L503)
    }
    // ...
}
```

**瞬时属性分离设计说明** —— [init.svelte.ts#L477-L490](js/core/src/init.svelte.ts#L477-L490) 的注释明确说明了为什么 `loading_status` 要排除在 `pending_updates` 缓存之外：

> Exclude loading_status because it is a transient real-time prop managed independently by the loading status store. Storing it would cause a stale "pending" update to be applied after the correct "complete" status has already been received, trapping the component in an infinite loading state.

**create_props_shared_props()** —— [init.svelte.ts#L691-L710](js/core/src/init.svelte.ts#L691-L710)：
```typescript
function create_props_shared_props(props) {
    for (const key in props) {
        if (allowed_shared_props.includes(key as keyof SharedProps)) {
            _shared_props[_key] = props[key];   // loading_status 是 allowed_shared_props 一员
        } else {
            _props[key] = props[key];
        }
    }
}
```

`allowed_shared_props` 中包含 `loading_status` —— [utils.svelte.ts#L293-L324](js/utils/src/utils.svelte.ts#L293-L324)（`loading_status` 在第 314 行）。

**初始值**：`gather_props()` 为所有组件初始化 `loading_status = {}` —— [init.svelte.ts#L760-L764](js/core/src/init.svelte.ts#L760-L764)。

### 4.6 MountComponents.svelte —— 将 shared_props 传入 Svelte 组件

[MountComponents.svelte#L9-L31](js/core/src/MountComponents.svelte#L9-L31)：
```svelte
{#if node && component}
    {#if node.props.shared_props.visible && !node.runtime}
        <svelte:component
            this={component.default}
            shared_props={node.props.shared_props}   // ← loading_status 包含在这里
            props={node.props.props}
        >
            {#each node.children as _node}
                <Self node={_node} />
            {/each}
        </svelte:component>
    {:else if ...}
        <MountCustomComponent {...rest} {node} .../>
    {/if}
{/if}
```

### 4.7 Gradio 类 —— shared_props → gradio.shared.loading_status

每个组件构造时创建 `new Gradio(_props)`，**构造函数中将 `_props.shared_props` 全量赋值到 `this.shared`**：

[utils.svelte.ts#L380-L406](js/utils/src/utils.svelte.ts#L380-L406)：
```typescript
constructor(_props: { shared_props: SharedProps; props: U }, default_values?: Partial<U>) {
    for (const key in _props.shared_props) {
        this.shared[key] = _props.shared_props[key];   // ← loading_status 赋值到 this.shared
    }
    // ...
}
```

**响应式同步**：`$effect` 在每次 `_props.shared_props` 变化时重新同步 —— [utils.svelte.ts#L437-L462](js/utils/src/utils.svelte.ts#L437-L462)：
```typescript
$effect(() => {
    for (const key in _props.shared_props) {
        this.shared[key] = _props.shared_props[key];   // ← loading_status 变化立即同步
    }
    // ...
});
```

**`set_data()` 路径**（组件挂载后通过 `_set_data` 回调）—— [utils.svelte.ts#L520-L556](js/utils/src/utils.svelte.ts#L520-L556)：
```typescript
set_data(data: Partial<U & SharedProps>): void {
    for (const key in data) {
        const value = data[key];
        if (this.shared_props.includes(key as keyof SharedProps)) {
            // @ts-ignore
            this.shared[key] = value;    // ← loading_status 更新到 this.shared
        } else {
            // @ts-ignore
            this.props[key] = value;
        }
    }
}
```

`set_data` 在构造函数中通过 `register_component` 注册为 AppTree 的 `#set_callbacks[id]` —— [utils.svelte.ts#L430-L435](js/utils/src/utils.svelte.ts#L430-L435)：
```typescript
this.register_component(
    _props.shared_props.id,
    this.set_data.bind(this),   // ← 注册为 _set_data 回调
    this.get_data.bind(this)
);
```

---

## 第五层：组件内部直接渲染 StatusTracker（两条路径）

组件展示进度有两种模式，**所有组件都是直接读取 `gradio.shared.loading_status` 来渲染 `<StatusTracker>`**。

### 路径 A：叶子组件（如 Textbox、Slider）自行渲染

以 **Textbox** 为例：

[textbox/Index.svelte#L52-L110](js/textbox/Index.svelte#L52-L110)：
```svelte
<Block ...>
    <!-- ✅ 直接读取 gradio.shared.loading_status，存在则渲染 StatusTracker -->
    {#if gradio.shared.loading_status}
        <StatusTracker
            autoscroll={gradio.shared.autoscroll}
            i18n={gradio.i18n}
            {...gradio.shared.loading_status}
            show_validation_error={false}
            on_clear_status={() =>
                gradio.dispatch("clear_status", gradio.shared.loading_status)}
        />
    {/if}

    <!-- 实际组件内容 -->
    <TextBox
        ...
        validation_error={gradio.shared?.loading_status?.validation_error ||
            gradio.shared?.validation_error}
        ...
    />
</Block>
```

**注意**：Textbox 还读取 `gradio.shared.loading_status.validation_error` 作为验证错误优先值 —— [textbox/Index.svelte#L92-L93](js/textbox/Index.svelte#L92-L93)。

其他叶子组件同样的模式（摘录）：

| 组件 | StatusTracker 渲染位置 |
|---|---|
| Textbox | [textbox/Index.svelte#L62-L71](js/textbox/Index.svelte#L62-L71) |
| SimpleTextbox | [simpletextbox/Index.svelte#L44-L51](js/simpletextbox/Index.svelte#L44-L51) |
| Slider | [slider/Index.svelte#L100-L103](js/slider/Index.svelte#L100-L103) |
| SimpleImage | [simpleimage/Index.svelte#L48-L50](js/simpleimage/Index.svelte#L48-L50)、[#L76-L78](js/simpleimage/Index.svelte#L76-L78) |
| SimpleDropdown | [simpledropdown/Index.svelte#L40-L47](js/simpledropdown/Index.svelte#L40-L47) |
| Video | [video/Index.svelte#L97-L99](js/video/Index.svelte#L97-L99)、[#L144-L146](js/video/Index.svelte#L144-L146) |
| Sidebar | [sidebar/Index.svelte#L15](js/sidebar/Index.svelte#L15) |

### 路径 B：布局组件（Row、Column、BaseColumn）自行渲染

以 **Row** 为例：

[row/Index.svelte#L59-L70](js/row/Index.svelte#L59-L70)：
```svelte
{#if gradio.shared.loading_status && gradio.shared.loading_status.show_progress && gradio}
    <StatusTracker
        autoscroll={gradio.shared.autoscroll}
        i18n={gradio.i18n}
        {...gradio.shared.loading_status}
        <!-- ✅ 关键语义转换：pending → generating -->
        status={gradio.shared.loading_status
            ? gradio.shared.loading_status.status == "pending"
                ? "generating"
                : gradio.shared.loading_status.status
            : null}
    />
{/if}
```

**BaseColumn 同样的模式**（ChatInterface 等复合组件用）：[column/BaseColumn.svelte#L32-L45](js/column/BaseColumn.svelte#L32-L45)：
```svelte
{#if gradio.shared.loading_status && gradio.shared.loading_status.show_progress}
    <StatusTracker
        autoscroll={gradio.shared.autoscroll}
        i18n={gradio.shared.i18n}
        {...gradio.shared.loading_status}
        status={gradio.shared.loading_status
            ? gradio.shared.loading_status.status == "pending"
                ? "generating"
                : gradio.shared.loading_status.status
            : null}
    />
{/if}
```

### 5.3 StatusTracker 组件 —— 最终渲染进度条

**导入与实例化**：所有组件通过 `import StatusTracker from "@gradio/statustracker"` 引用，该包的导出在 [statustracker/index.ts](js/statustracker/index.ts)。

**StatusTracker 核心渲染逻辑** —— [statustracker/static/index.svelte](js/statustracker/static/index.svelte)：

**进度条计算（progress_level 派生变量）** —— [index.svelte#L188-L222](js/statustracker/static/index.svelte#L188-L222)：
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
    return _progress_level;
});
```

**进度条宽度（last_progress_level）** —— [index.svelte#L224-L237](js/statustracker/static/index.svelte#L224-L237)：
```typescript
let last_progress_level = $derived.by(() => {
    const last = progress_level?.[progress_level.length - 1];
    return last != null && last >= 0 ? Math.min(last * 100, 100) : undefined;
});
```

**显示条件分支** —— [index.svelte#L320-L445](js/statustracker/static/index.svelte#L320-L445)：

```
should_hide 为 true（status=="complete" 且无 message 无 error）→ 隐藏
├─ status=="pending"（即前端显示的 "generating"）
│   ├─ progress_data 存在且 last_progress_level 有值 → 完整进度条
│   │   ├─ 每层 progress: desc + (index/length% 或 progress%)
│   │   └─ 最内层进度条 style.width = last_progress_level + "%"
│   ├─ 否则 ETA 存在 → 显示 ETA 计时器 + 伪进度条（按时间推进）
│   └─ 否则 show_progress=="full" → 仅 Loader spinner
├─ status=="error" → 红色错误条 + message
└─ status=="complete" → 成功消息 or 缓存提示 or 空
```

**ETA 计时启动** —— [index.svelte#L276-L295](js/statustracker/static/index.svelte#L276-L295)：当 `status === "pending"` 时，通过 `$effect` 启动 `requestAnimationFrame` 循环，计算实际流逝时间占预估 ETA 的比例。

**缓存命中指示** —— [index.svelte#L174-L186](js/statustracker/static/index.svelte#L174-L186)：当 `success === true && used_cache === true` 时，显示 `⚡ from cache: {duration}s`。

### 5.4 清除状态路径

用户点击 StatusTracker 上的关闭按钮 → 组件 dispatch `"clear_status"` 事件：
- 如 Textbox 中 `on_clear_status={() => gradio.dispatch("clear_status", ...)}` —— [textbox/Index.svelte#L68-L69](js/textbox/Index.svelte#L68-L69)

该事件在 Blocks.svelte 中处理 —— [Blocks.svelte#L127-L138](js/core/src/Blocks.svelte#L127-L138)：
```typescript
else if (event == "clear_status") {
    app_tree.update_state(id, { loading_status: {} }, false);
    dep_manager.clear_loading_status(id);  // → loading_stati.clear(id)
}
```

---

## 完整消息流向图（每步附可复核的代码位置）

```
用户函数 demo/progress/run.py
  │  progress(0.5, desc="Processing") [helpers.py#L736-L764]
  │  或 for i in progress.tqdm(range(10)): [helpers.py#L766-L802]
  ▼
Progress._progress_callback() [helpers.py#L804-L823]
  │  LocalContext.blocks.get() → ContextVar [context.py#L22-L34]
  │  LocalContext.event_id.get()
  │  → 返回 partial(blocks._queue.set_progress, event_id)
  ▼
Queue.set_progress(event_id, iterables) [queueing.py#L585-L608]
  │  TrackedIterable → ProgressUnit [server_messages.py#L114-L122]
  │  evt.progress = ProgressMessage(progress_data) [server_messages.py#L58-L62]
  │  evt.progress_pending = True (⚠️ 节流标记)
  ▼
Queue.start_progress_updates() 定时轮询 [queueing.py#L555-L583]
  │  await asyncio.sleep(0.01s/0.1s)
  │  evt.progress_pending = False
  │  → send_message(event, event.progress)
  ▼
Queue.send_message() [queueing.py#L240-L249]
  │  event_message.event_id = event._id
  │  pending_messages_per_session[session_hash].put_nowait(msg)
  ▼
SSE 端点 sse_stream() [routes.py#L1490-L1557]
  │  asyncio.wait_for(messages.get(), timeout=10)
  │  process_msg() → orjson.dumps(): "data: {json}\n\n"
  ▼
JS Client handle_message() [api_info.ts#L234-L404]
  │  msg=="progress" → type:"update", stage:"pending" [api_info.ts#L252-L280]
  │  progress_data 保留在 status 对象中
  ▼
JS Client fire_event() [submit.ts#L500-L510]
  │  { type:"status", stage:"pending", progress_data:[...] }
  │  推入 make_promise_with_events 异步迭代器队列
  ▼
DependencyManager for await 循环 [dependency.ts#L431-L612]
  │  result.type === "status" → stage: "pending" 分支 [dependency.ts#L559-L567]
  │  → loading_stati.update({...status, fn_index, status:stage, ...})
  │  → update_loading_stati_state() [dependency.ts#L286-L298]
  ▼
LoadingStatus.update() → resolve_args() [state.svelte.ts#L44-L92]
  │  fn_index → fn_outputs[] + fn_inputs[] (component_ids)
  │  current[component_id] = ILoadingStatus{progress_data, eta, ...}
  ▼
DependencyManager.update_loading_stati_state() [dependency.ts#L286-L298]
  │  for component_id, loading_status:
  │     update_state_cb(id, {loading_status}, false)
  ▼
AppTree.update_state() [init.svelte.ts#L439-L513]
  │  组件未挂载 → node.props.shared_props.loading_status = xxx（就地修改）
  │               #pending_updates 排除 loading_status [init.svelte.ts#L482]
  │  组件已挂载 → _set_data(new_state) [init.svelte.ts#L501-L503]
  ▼
MountComponents.svelte 重新渲染 [MountComponents.svelte#L9-L31]
  │  <svelte:component shared_props={node.props.shared_props} ... />
  │  包含 loading_status 字段
  ▼
Gradio 类 constructor / $effect [utils.svelte.ts#L380-L462]
  │  for key in _props.shared_props: this.shared[key] = value
  │  → gradio.shared.loading_status 响应式更新
  ▼
┌───────────────────────────────────────────────────────┐
│  组件 Index.svelte 直接读取 gradio.shared.loading_status  │
├───────────────────────────────────────────────────────┤
│ 叶子组件模式（Textbox 等）：                           │
│  {#if gradio.shared.loading_status}                   │
│     <StatusTracker {...gradio.shared.loading_status}/>│
│  {/if}  [textbox/Index.svelte#L62-L71]               │
│                                                       │
│ 布局组件模式（Row 等）：                               │
│  {#if gradio.shared.loading_status?.show_progress}    │
│     <StatusTracker                                    │
│       status={status=="pending" ? "generating"        │
│                 : status} />   [row/Index.svelte#L59-L70]│
└───────────────────────────────────────────────────────┘
  ▼
StatusTracker 最终渲染 [statustracker/static/index.svelte]
  │  progress_level 派生 [index.svelte#L188-L222]
  │  last_progress_level → style.width [index.svelte#L224-L237]
  │  ETA 计时器 rAF 循环 [index.svelte#L276-L295]
  ▼
用户看到进度条
```

---

## 关键设计模式总结（附代码证据）

| 设计模式 | 说明 | 代码证据位置 |
|---|---|---|
| **ContextVar 隔离** | `LocalContext` 使用 Python `ContextVar`，多线程/协程环境下每个请求的 `blocks`、`event_id` 互不干扰 | [context.py#L22-L34](gradio/context.py#L22-L34)、[utils.py#L1070-L1103](gradio/utils.py#L1070-L1103) |
| **进度节流** | `evt.progress_pending` 标志 + `start_progress_updates()` 定时轮询（10-100ms），高频更新只保留最新值 | [queueing.py#L555-L608](gradio/queueing.py#L555-L608) |
| **会话隔离** | `pending_messages_per_session[session_hash]` 按浏览器会话分组，SSE 端点仅推送该会话消息 | [queueing.py#L240-L249](gradio/queueing.py#L240-L249) |
| **fn_index → component_id 映射** | `LoadingStatus` 将后端 `fn_index` 维度转换为前端 `component_id` 维度，支持一个函数映射到多个输入/输出组件 | [state.svelte.ts#L18-L92](js/statustracker/static/state.svelte.ts#L18-L92) |
| **瞬时属性分离** | `loading_status` 被排除在 `#pending_updates` 缓存之外，防止组件延迟挂载时过期 pending 覆盖已完成状态 | [init.svelte.ts#L477-L490](js/core/src/init.svelte.ts#L477-L490) |
| **状态语义转换** | 后端 `"pending"` 在前端渲染层转换为 `"generating"`，更符合用户"正在处理"的认知 | [row/Index.svelte#L64-L68](js/row/Index.svelte#L64-L68)、[column/BaseColumn.svelte#L35-L42](js/column/BaseColumn.svelte#L35-L42) |
| **响应式同步** | 组件 `shared_props` 的变化通过 `$effect` 立即同步到 `gradio.shared`，无需手动订阅 | [utils.svelte.ts#L437-L462](js/utils/src/utils.svelte.ts#L437-L462) |
| **双路径状态更新** | 组件未挂载时就地修改 tree node 的 `shared_props`；已挂载时通过注册的 `_set_data` 回调直接更新组件内部状态 | [init.svelte.ts#L457-L503](js/core/src/init.svelte.ts#L457-L503) |
| **直接读取模式** | 所有组件（叶子/布局）统一从 `gradio.shared.loading_status` 读取并自行决定是否渲染 `<StatusTracker>`，无额外数据流 | 见 [textbox/Index.svelte#L62-L71](js/textbox/Index.svelte#L62-L71) 等多处 |
| **show_progress 控制** | `BlockFunction.show_progress` 通过依赖配置传递，控制进度条显示级别（"full"/"minimal"/"hidden"） | [block_function.py#L131-L138](gradio/block_function.py#L131-L138) → [dependency.ts#L273-L278](js/core/src/dependency.ts#L273-L278) |
