# Gradio 事件队列机制执行顺序梳理

本文档顺着代码梳理 Gradio 的事件队列系统中 **请求排队**、**并发限制**、**结果回传** 三者的协作方式。

---

## 1. 核心数据结构

### 1.1 Event（请求事件对象）

位置：[queueing.py:54-91](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L54-L91)

每个客户端请求被封装为一个 `Event` 对象：

| 字段 | 作用 |
|------|------|
| `_id` | 事件唯一标识（uuid） |
| `session_hash` | 会话哈希（前端生成），用于消息路由 |
| `fn` | 对应的 `BlockFunction`，含并发配置 |
| `concurrency_id` | 并发组 ID（从 fn 继承） |
| `data` | 请求体 `PredictBodyInternal` |
| `alive` / `closed` | 事件生命周期状态 |
| `signal` | `asyncio.Event`，流式事件的同步信号 |
| `enqueue_time` | 入队时间（单调时钟） |
| `run_time` | 累积运行时间（用于流式 time_limit） |
| `progress` / `progress_pending` | 进度更新状态 |

### 1.2 EventQueue（按并发组的子队列）

位置：[queueing.py:93-101](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L93-L101)

每个 `concurrency_id` 对应一个独立的 `EventQueue`：

| 字段 | 作用 |
|------|------|
| `queue` | `list[Event]`，FIFO 等待队列 |
| `concurrency_limit` | 该并发组的最大并行执行数 |
| `current_concurrency` | 当前正在执行的事件数 |
| `start_times_per_fn` | 按 fn 记录开始时间，用于 ETA 估算 |

### 1.3 Queue（全局队列管理器）

位置：[queueing.py:116-158](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L116-L158)

全局唯一的调度器，维护以下关键映射：

| 字段 | 类型 | 作用 |
|------|------|------|
| `event_queue_per_concurrency_id` | `dict[str, EventQueue]` | 按并发组分桶的等待队列 |
| `active_jobs` | `list[None \| list[Event]]` | 工作线程池，大小 = `max_thread_count` |
| `pending_messages_per_session` | `LRUCache[str, AsyncQueue[EventMessage]]` | 按 session 存储待发送消息（SSE 推送源） |
| `pending_event_ids_session` | `dict[str, set[str]]` | 每个 session 正在处理的 event_id 集合 |
| `event_ids_to_events` | `dict[str, Event]` | event_id → Event 的快速查找 |
| `process_time_per_fn` | `defaultdict[BlockFunction, ProcessTime]` | 每个 fn 的历史平均耗时，用于 ETA 估算 |

---

## 2. 请求排队流程

### 2.1 整体时序

```
前端 (Client.submit)                     后端 (Routes + Queue)
       |                                        |
       | POST /queue/data                       |  SSE_DATA_URL
       | { data, fn_index, session_hash }       |
       |--------------------------------------->|
       |                                        | queue_join_helper()
       |                                        |   → body → PredictBodyInternal
       |                                        |   → Queue.push(body, request, username)
       |                                        |
       |                             ┌──────────┴──────────┐
       |                             │   Queue.push()      │
       |                             └──────────┬──────────┘
       |                                        │
       |      1. 校验 max_size / fn_index       │
       |      2. 创建 Event + event_id          │
       |      3. 缓存探测 (ProbeCache)          │
       |      4. 入队 event_queue.queue         │
       |      5. broadcast_estimations()        │
       |                                        │
       |  { "event_id": "xxx" } (HTTP 200)      │
       |<---------------------------------------|
       |                                        |
       | 注册 event_callbacks[event_id]         |
       | 打开全局 SSE 连接 (open_stream)        |
       |                                        |
       | GET /queue/data?session_hash=xxx       |
       |--------------------------------------->|
       |                                        | sse_stream()
       |    (SSE 长连接保持)                    |   ← 从 pending_messages_per_session 取消息
       |    接收: EstimationMessage             |   ← heartbeat 协程保活
       |    接收: ProcessStartsMessage          |
       |    接收: ProcessGeneratingMessage      |
       |    接收: ProcessCompletedMessage       |
       |                                        |   全部事件完成 → CloseStreamMessage
```

### 2.2 前端入口

**Client.submit()** 位置：[submit.ts:32-40](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L32-L40)

根据 `protocol` 选择不同路径，当前主流为 `sse_v2 / sse_v2.1 / sse_v3`：

1. 处理 blob / 文件上传 → `handle_blob()`
2. 将 payload POST 到 `/queue/data`
3. 响应返回 **event_id**
4. 创建 `callback` 函数，注册到 `event_callbacks[event_id]`
5. 若 SSE 连接未打开，调用 `open_stream()`

关键点：[submit.ts:619-628](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L619-L628)
- 若该 event_id 已有**暂存消息**（SSE 先到、回调后注册的竞态），先回放
- `unclosed_events` 跟踪未关闭事件

### 2.3 后端入队：Queue.push()

位置：[queueing.py:279-468](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L279-L468)

**步骤详解：**

**Step 1：前置校验**
```python
if self.max_size is not None and len(self) >= self.max_size:
    return (False, "Queue is full...", "queue_full")
```
检查总队列大小是否超限。

**Step 2：解析 BlockFunction 并创建专属 EventQueue**
```python
fn = route_utils.get_fn(self.blocks, None, body)
self.create_event_queue_for_fn(fn)  # 若该 concurrency_id 首次出现
```
`create_event_queue_for_fn()` 位置：[queueing.py:216-235](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L216-L235)
- 按 `concurrency_id` 建立 `EventQueue`
- 同一 concurrency_id 下取 **最低** 的 `concurrency_limit`

**Step 3：Validator 前置验证**（可选）
- 若 `fn.validator` 存在，先同步调用验证函数
- 验证失败直接返回 `validator_error`，不入队

**Step 4：缓存探测（ProbeCache）**（可选）
```python
if hasattr(fn.fn, "cache"):
    try:
        with ProbeCache():
            response = await route_utils.call_process_api(...)
            while response and response.get("is_generating"):
                ...  # 发送流式缓存消息
        # 缓存全命中 → 直接发送 ProcessCompletedMessage，不走排队
        return True, event._id, "success"
    except CacheMissError:
        pass  # 回退到正常排队路径
```

**Step 5：创建 Event 并注册到 Session**
```python
event = Event(body.session_hash, fn, request, username)
event.data = body
# 初始化该 session 的 AsyncQueue（若不存在）
if body.session_hash not in self.pending_messages_per_session:
    self.pending_messages_per_session[body.session_hash] = AsyncQueue()
self.pending_event_ids_session[body.session_hash].add(event._id)
self.event_ids_to_events[event._id] = event
```

**Step 6：入队 + 广播 ETA**
```python
event_queue.queue.append(event)
self.event_analytics[event._id] = {...}
self.broadcast_estimations(event.concurrency_id, len(event_queue.queue) - 1)
return True, event._id, "success"
```

---

## 3. 并发限制机制（两层门控）

### 3.1 第一层：全局工作线程池

**主循环 start_processing()** 位置：[queueing.py:521-563](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L521-L563)

```
while 循环:
  ├─ 队列为空 → sleep(sleep_when_free)
  ├─ active_jobs 无空位 → sleep(sleep_when_free)
  └─ 有空位:
      ├─ delete_lock 保护下调用 get_events()
      ├─ 将取出的 events 放入 active_jobs 槽位
      ├─ event_queue.current_concurrency += 1
      └─ run_coro_in_background(process_events, events, batch, start_time)
```

- `active_jobs = [None] * max_thread_count`（由 `Blocks.queue(concurrency_count=...)` 设置）
- 这是**全局硬上限**，任何时刻并行执行的协程数不超过该值
- Windows 平台 `sleep_when_free = 0.05s`，其他平台 `0.001s`

### 3.2 第二层：按 concurrency_id 限并发

**get_events()** 位置：[queueing.py:496-519](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L496-L519)

```python
def get_events(self):
    random.shuffle(concurrency_ids)  # 公平性：打乱并发组的遍历顺序
    for concurrency_id in concurrency_ids:
        event_queue = self.event_queue_per_concurrency_id[concurrency_id]
        # 关键条件：该组未达到并发上限
        if len(event_queue.queue) and (
            event_queue.concurrency_limit is None
            or event_queue.current_concurrency < event_queue.concurrency_limit
        ):
            first_event = event_queue.queue[0]
            events = [first_event]
            
            # 批处理：若启用 batch，收集同函数的事件
            if first_event.fn.batch:
                events += [同 fn 的后续事件][: max_batch_size - 1]
            
            # 从等待队列移除
            for event in events:
                event_queue.queue.remove(event)
            return events, batch, concurrency_id
    return None  # 所有并发组都无可用名额
```

**并发限制释放**（process_events 的 finally 块）位置：[queueing.py:1054-1058](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L1054-L1058)
```python
event_queue.current_concurrency -= 1
start_times.remove(begin_time)
self.active_jobs[self.active_jobs.index(events)] = None
```

### 3.3 concurrency_limit 的解析链

优先级从高到低：

1. **BlockFunction 个体设置**（`gr.Button().click(..., concurrency_limit=3)`）
2. **若值为 "default"** → 查 `Queue.default_concurrency_limit`
3. **Queue.default_concurrency_limit** 来源：
   - `Blocks.queue(default_concurrency_limit=...)` 参数
   - 环境变量 `GRADIO_DEFAULT_CONCURRENCY_LIMIT`（支持 `"none"` 表示无限制）
   - **默认值 1**（即默认单并发）

代码位置：[queueing.py:251-271](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L251-L271)

### 3.4 concurrency_id（并发分组）

来源：[block_function.py:74](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/block_function.py#L74)
```python
self.concurrency_id = concurrency_id or str(id(fn))
```

- 默认：用 Python 对象的内存地址 `id(fn)`，即**每个函数独立一个并发组**
- 手动设置：多个函数设置相同 `concurrency_id`，共享并发额度
- 生效策略：同组取最低 `concurrency_limit`（木桶效应）

### 3.5 批处理 (batch)

当 `BlockFunction.batch=True` 时：
- 取队首事件后，继续向后收集**同 fn** 的事件
- 最多 `max_batch_size - 1` 个（加上队首共 max_batch_size）
- 整批事件共享 **一个 active_job 槽位**，一同执行
- 输入数据在 [queueing.py:819-827](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L819-L827) 转置打包，
  输出在 [queueing.py:1011-1014](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L1011-L1014) 拆分还原

---

## 4. 结果回传的协作方式

### 4.1 消息路由核心：send_message()

位置：[queueing.py:240-249](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L240-L249)

```python
def send_message(self, event: Event, event_message: EventMessage):
    if not event.alive:
        return
    event_message.event_id = event._id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```

所有回传消息**统一入口**：写入对应 session 的 `AsyncQueue`，由 SSE 连接统一消费。

### 4.2 服务端 SSE 推送：/queue/data

位置：[routes.py:1473-1576](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1473-L1576)

```
sse_stream() 循环:
  ├─ 检测 request.is_disconnected()
  │   └─ 是 → clean_events(session_hash) + cancel heartbeat
  ├─ 10s 超时从 pending_messages_per_session[session_hash].get() 取消息
  ├─ 队列停止 → 注入 UnexpectedErrorMessage
  ├─ process_msg() 序列化为 "data: {...}\n\n" 格式
  ├─ yield 响应
  └─ 若是 ProcessCompletedMessage:
      ├─ 从 pending_event_ids_session[session_hash] 移除该 event_id
      └─ 若该 session 所有事件都完成 → yield CloseStreamMessage + 退出
```

**心跳保活**：[routes.py:1481-1488](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1481-L1488)
- 独立 `heartbeat()` 协程定期 `await queue.put(HeartbeatMessage())`
- 防止连接被中间代理因空闲而断开

### 4.3 前端 SSE 多路分发

**open_stream()** 位置：[stream.ts:5-89](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/stream.ts#L5-L89)

前端**全局共享一个 SSE 连接**（按 session_hash），收到消息后按 event_id 分发：

```
stream.onmessage:
  ├─ msg === "close_stream" → 关闭连接
  ├─ 无 event_id → 广播给 ALL event_callbacks（如心跳）
  ├─ 存在 event_callbacks[event_id]:
  │   └─ 浏览器环境: setTimeout(fn, 0, _data)  // 让出渲染机会
  │   └─ Node 环境: 直接 fn(_data)
  └─ 回调未就绪 → 暂存 pending_stream_messages[event_id]
```

**竞态处理**：SSE 消息可能在 event_callbacks 注册前就到达，此时暂存；在 [submit.ts:619-621](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L619-L621) 注册时回放。

### 4.4 流式事件的双向协作（connection="stream"）

流式事件是服务端**等待客户端**推送数据的双向交互模式。

**客户端 → 服务端（数据块推送）**：

- `POST /stream/{event_id}` 位置：[routes.py:1088-1094](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1088-L1094)
  ```python
  event.data = body  # 更新最新输入数据
  event.signal.set()  # 唤醒正在等待的处理协程
  ```

- `POST /stream/{event_id}/close` 位置：[routes.py:1096-1102](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1096-L1102)
  ```python
  event.run_time = math.inf   # 触发 is_finished = True
  event.closed = True
  event.signal.set()
  ```

**服务端等待信号**：[queueing.py:751-785](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L751-L785)

```python
@staticmethod
async def wait_for_event_or_timeout(event, timeout):
    t1 = wait_for_event(event)        # 等待 event.signal
    t2 = sleep(timeout)               # 超时保护
    done, _ = wait([t1, t2], FIRST_COMPLETED)
    event.signal.clear()              # 重置，供下次等待
    return done[0].result()
```

在 `process_events()` 的流式循环中：
1. 执行一次 fn 调用，获得中间结果
2. 发送 `ProcessGeneratingMessage`
3. 调用 `wait_for_batch(awake_events, timeouts)`，等待所有流式事件的客户端新数据
4. 循环直到 `event.is_finished`（达到 `time_limit` 或收到 close）

### 4.5 进度更新（节流）

位置：[queueing.py:565-609](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L565-L609)

为避免高频进度条更新挤压带宽，采用**定期刷新**策略：

- 用户代码调用 `gr.Progress()` → 内部调 `set_progress()` → 设置 `evt.progress_pending = True`
- 独立协程 `start_progress_updates()` 每隔 `progress_update_sleep_when_free` 扫描一次
- 只发送**最后一次**进度（连续多次 set_progress 会被合并）

### 4.6 排队估算与广播

**broadcast_estimations()** 位置：[queueing.py:669-734](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L669-L734)

遍历 EventQueue 中所有等待事件，计算每个事件的：

```
rank              # 队列中的位置（0 = 队首）
queue_size        # 该并发组的总队列长度
rank_eta          # 预计等待时间（秒）
```

**ETA 计算公式**：
```
rank_eta = process_time_for_fn   # 自身执行耗时（历史均值）
         + wait_so_far           # 轮到自己需要的等待（累积分摊）
         + time_till_available_worker  # 首个空闲worker的剩余时间
```

触发时机：
1. 新事件入队时：`push()` 末尾立即广播（从当前 rank 开始）
2. 主循环取走事件时：`start_processing()` 中分配完任务后广播（`live_updates=True`）
3. 轮询兜底：`notify_clients()` 协程按 `update_intervals` 周期性广播

---

## 5. 完整请求生命周期流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CLIENT (JS / Browser)                              │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         │ 1. submit() → POST /queue/data  (fn_index + data + session_hash)
         │    └─ skip_queue() 检查 → 决定是否走队列
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          /queue/data  ROUTE                                 │
│                   queue_join_helper() → Queue.push()                       │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ├─ [Validator 阶段] 同步执行 fn.validator → 失败返回 422
         ├─ [Cache 阶段]    ProbeCache 尝试 → 命中则直接回传结果 + 结束
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  入队 EventQueue[concurrency_id].queue                                      │
│  广播 EstimationMessage(rank, eta) → pending_messages_per_session          │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         │ 2. 返回 { event_id } → 前端注册 callback + 打开 SSE 连接
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              SSE 通道建立 (GET /queue/data?session_hash=)                   │
│   ┌─ heartbeat 协程：定期塞 HeartbeatMessage                                │
│   └─ 消费循环：pending_messages_per_session.get() → yield SSE               │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         │ 3. Queue.start_processing() 主循环调度
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  get_events() 两层门控:                                                     │
│    Layer1: active_jobs 有空位?                                              │
│    Layer2: EventQueue[concurrency_id].current_concurrency < limit?          │
│  → 满足则取出 events（可能批量）→ 放入 active_jobs 槽位                      │
│  → current_concurrency++ → 启动 process_events 协程                         │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         │ 4. process_events() 执行
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ProcessStartsMessage(eta)  → send_message()                                │
│  call_process_api() 调用用户 fn                                             │
│  ┌─────────────────────────────────────────────────────┐                   │
│  │ is_generating == true? (流式/生成式函数)              │                   │
│  │   YES → ProcessGeneratingMessage → send_message()    │                   │
│  │      → [stream 模式] wait_for_batch() 等待 client    │                   │
│  │         推送 POST /stream/{event_id} → signal.set()  │                   │
│  │      → 循环迭代直到 is_finished / 超时 / 断开        │                   │
│  │   NO  → ProcessCompletedMessage → send_message()     │                   │
│  └─────────────────────────────────────────────────────┘                   │
│  finally: current_concurrency--                                            │
│           active_jobs[slot] = None                                         │
│           process_time_per_fn[fn].add(duration)  # 更新 ETA 统计           │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         │ 5. SSE 通道推送所有累积消息
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  open_stream() onmessage: 按 event_id 分发 callback                         │
│    callback → fire_event(status/data) → UI 更新                            │
│    process_completed → 若该 session 无待处理事件 → CloseStreamMessage       │
│    浏览器端: setTimeout(fn, 0) 分帧，避免消息风暴阻塞渲染                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键状态与协作原语总结

| 原语 | 位置 | 作用 |
|------|------|------|
| `AsyncQueue`（per session） | [queueing.py:126-128](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L126-L128) | send_message 与 SSE 流的解耦缓冲区 |
| `asyncio.Event`（per Event） | [queueing.py:76](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L76) | 流式事件的客户端数据到达信号 |
| `current_concurrency` 计数 | [queueing.py:98](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L98) | 并发组内并行数的信号量语义 |
| `active_jobs` 槽位池 | [queueing.py:136](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L136) | 全局工作协程上限（令牌桶语义） |
| `delete_lock` | [queueing.py:137](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L137) | 保护队列列表遍历+删除的原子性 |
| `pending_message_lock` | [queueing.py:131](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L131) | 保护 session AsyncQueue 的初始化竞态 |
| `event.signal` + `asyncio.wait` | [queueing.py:751-785](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L751-L785) | 流式事件"等待数据/超时"二选一的模式 |
| 单 SSE + event_id 分发 | [stream.ts:41-77](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/stream.ts#L41-L77) | 多路复用，避免每个事件独立建连接 |
