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

---

## 7. 边界路径的精确状态变化

三条边界路径（取消请求、连接断开、服务端停止）的核心差异在于**事件集合**、**消息队列**、**关闭条件**三者的变化时机和方式。先明确定义三个核心状态结构：

### 7.0 三个核心状态结构定义

| 状态类型 | 数据结构 | 含义 |
|---------|---------|------|
| **事件集合** | `pending_event_ids_session[session_hash]` | `set[str]`，该 session 所有未完成的 event_id。SSE 关闭的唯一判定依据 |
| | `event_ids_to_events` | `dict[str, Event]`，全局 event_id → Event 映射 |
| | `EventQueue.queue` | `list[Event]`，按并发组的**等待队列**（已取出执行的事件不在此） |
| **消息队列** | `pending_messages_per_session[session_hash]` | `AsyncQueue[EventMessage]`，该 session 的待发送消息缓冲。`send_message()` 的唯一目的地 |
| **关闭条件** | [routes.py:1543-1553](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1543-L1553) | `msg == server_stopped` **OR** `(msg == process_completed AND len(pending_event_ids_session) == 0)` |

---

### 7.1 路径一：POST /cancel（客户端主动取消）

位置：[routes.py:1401-1429](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1401-L1429)

```
客户端 iterator.cancel()
       │
       ▼
POST /cancel { event_id, session_hash, fn_index }
       │
       ├─ Step 1: cancel_tasks({f"{session_hash}_{fn_index}"})
       │     └─ 遍历 asyncio.Task，按 task name 匹配
       │        格式: "{session_hash}_{fn_index}<gradio-sep>{event_id}"
       │        匹配到的 .cancel() → asyncio.CancelledError
       │        【不修改】三个核心状态结构
       │
       ├─ Step 2: remove_from_queue(event_id)
       │     位置: queueing.py:470-479
       │     ├─ 【事件集合】EventQueue.queue.remove(event)  # 仅从等待队列移除
       │     ├─ 【事件集合】event_ids_to_events.pop(event_id, None)
       │     └─ 【注意】pending_event_ids_session 【完全不动!】
       │
       ├─ Step 3: if session_open AND event_running:
       │     ├─ session_open = session_hash in pending_messages_per_session
       │     ├─ event_running = event_id in pending_event_ids_session[session_hash]
       │     └─ 【消息队列】pending_messages_per_session[session_hash].put_nowait(
       │           ProcessCompletedMessage(output={}, success=True, event_id=event_id)
       │        )
       │
       └─ Step 4: iterator 清理（不影响核心结构）
```

#### 关键：后续异步变化（SSE 流中）

**`pending_event_ids_session` 的修改不在 `/cancel` 路由中，而是在 SSE 流消费消息时！**

位置：[routes.py:1525-1559](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1525-L1559)

```
SSE 流消费到注入的 ProcessCompletedMessage:
       │
       ├─ if isinstance(message, ProcessCompletedMessage) and message.event_id:
       │   │
       │   ├─ 【事件集合】
       │   │     if event_id in pending_event_ids_session[session_hash]:
       │   │         pending_event_ids_session[session_hash].remove(event_id)
       │   │     ← 【这里才真正修改 pending_event_ids_session!】
       │   │
       │   └─ 【关闭条件判定】
       │         if (message.msg == ServerMessage.process_completed
       │             and len(pending_event_ids_session[session_hash]) == 0):
       │             │
       │             ├─ 构造 CloseStreamMessage → yield
       │             ├─ heartbeat_task.cancel()
       │             └─ return  ← SSE 流优雅关闭
```

#### /cancel 完整状态链

```
POST /cancel
   │
   ├─ cancel_tasks() → 取消 Task
   ├─ remove_from_queue() → EventQueue.queue + event_ids_to_events
   │                                    (pending_event_ids_session 不动!)
   ├─ 注入 ProcessCompletedMessage → pending_messages_per_session
   │
   ▼ (异步，SSE 流消费)
pending_event_ids_session.remove(event_id)
   │
   └─ 若集合变空 → CloseStreamMessage → SSE return
```

---

### 7.2 路径二：SSE 连接断开（客户端失联/关闭标签页）

位置：[routes.py:1494-1497](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1494-L1497)

```
sse_stream() while 循环开头检测:
       │
       if await request.is_disconnected():
       │
       ├─ clean_events(session_hash=session_hash)
       │
       ├─ heartbeat_task.cancel()
       └─ return  ← SSE 流直接终止，不发任何消息
```

#### clean_events() 的精确操作

位置：[queueing.py:631-658](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L631-L658)

```python
async def clean_events(self, *, session_hash: str | None = None):
    # Part A: 标记正在执行的事件
    for job_set in self.active_jobs:
        if job_set:
            for job in job_set:
                if job.session_hash == session_hash:
                    job.alive = False  # 【仅标记! 不修改任何集合】
    
    # Part B: 收集并移除"等待中的"事件
    async with self.delete_lock:
        events_to_remove = []
        for event_queue in event_queue_per_concurrency_id.values():
            for event in event_queue.queue:  # 【只遍历等待队列!】
                if event.session_hash == session_hash:
                    events_to_remove.append(event)
        
        for event in events_to_remove:
            event_queue.queue.remove(event)       # 【事件集合】
            event_ids_to_events.pop(event._id, None)  # 【事件集合】
        
        # Part C: 只移除"等待中"的事件从 pending_event_ids_session
        if session_hash in self.pending_event_ids_session:
            removed_ids = {e._id for e in events_to_remove}  # 只含等待中的!
            self.pending_event_ids_session[session_hash] -= removed_ids
            if not self.pending_event_ids_session[session_hash]:
                self.pending_event_ids_session.pop(session_hash, None)
```

#### 精确状态变化（按事件状态分类）

| 事件状态 | EventQueue.queue | event_ids_to_events | pending_event_ids_session | event.alive |
|---------|------------------|---------------------|--------------------------|-------------|
| **正在执行** (在 active_jobs 中) | 不修改（已不在队列） | 不修改 | **不修改（保留）** | 设为 False |
| **等待中** (在 EventQueue.queue) | 移除 | 移除 | 移除 | —（未执行）|

| 其他状态 | 变化 |
|---------|------|
| **消息队列** (pending_messages_per_session) | **完全不碰!** AsyncQueue 和里面的消息都原封不动 |
| **关闭条件** | 不触发 — SSE 直接 `return`，不发送 CloseStreamMessage |
| **asyncio Task** | 不取消 — 正在执行的协程会自然完成，finally 块释放槽位 |

#### 后续影响
- `event.alive = False`：`process_events` 后续 `send_message()` 会直接 return（`if not event.alive: return`）
- `pending_event_ids_session` 仍包含正在执行的 event_id，但 SSE 已断开，没人消费了
- `pending_messages_per_session` 仍存在，但生产者（process_events）被 `alive=False` 阻挡，消费者（SSE 流）已退出，形成**僵尸队列**

---

### 7.3 路径三：服务端停止（Queue.stopped = True）

有两个并行变化链：

#### 变化链 A：start_processing 主循环退出

位置：[queueing.py:561-563](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L561-L563)

```python
finally:
    self.stopped = True
    self._cancel_asyncio_tasks()  # 取消所有 process_events 协程
```

- `_cancel_asyncio_tasks()` 遍历 `self._asyncio_tasks`，每个 `.cancel()`
- 被取消的 `process_events` 触发 `CancelledError`，进入 finally 块：
  - `current_concurrency--`
  - `active_jobs[slot] = None`
  - `reset_iterators()`

#### 变化链 B：SSE 流检测到 stopped 标志

位置：[routes.py:1516-1520](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1516-L1520)

```
while True:
    ...
    if blocks._queue.stopped:
        message = UnexpectedErrorMessage(
            message="Server stopped unexpectedly.",
            success=False,
        )
    ...
    yield process_msg(message)
    ...
    # 检查关闭条件
    if message.msg == ServerMessage.server_stopped or (...):
```

#### 关键发现：关闭条件的设计不匹配

`ServerMessage.server_stopped` 的值是 **字符串 `"Server stopped unexpectedly."`**（见 [utils.py:142](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/python/gradio_client/utils.py#L142)），而 `UnexpectedErrorMessage` 的 `msg` 字段是 `ServerMessage.unexpected_error = "unexpected_error"`。

所以条件 `message.msg == ServerMessage.server_stopped` **永远不会为 True**！

#### 精确状态变化

| 状态结构 | 变化 |
|---------|------|
| **EventQueue.queue** | 不直接修改（process_events finally 自然释放） |
| **event_ids_to_events** | 不直接修改 |
| **pending_event_ids_session** | 不修改（继续保留所有 event_id） |
| **pending_messages_per_session** | 注入 `UnexpectedErrorMessage` |
| **关闭条件** | 不触发 — msg 是 `"unexpected_error"`，不匹配 `server_stopped`，也不是 `process_completed` |
| **asyncio Task** | `_cancel_asyncio_tasks()` 批量取消所有后台任务 |

#### 实际效果
- SSE 流会不断循环，每次都注入 `UnexpectedErrorMessage` 发给客户端
- 客户端收到 `unexpected_error` 消息后，由前端逻辑自行处理关闭（`handle_message()` → `fire_event(stage="error")` → `close()`）
- 服务端最终会因为外层连接断开或进程退出而终止

---

### 7.4 三条路径状态对比总表

| 状态结构 | /cancel 路径 | SSE 断开路径 | 服务端停止路径 |
|---------|-------------|-------------|---------------|
| **EventQueue.queue** | `remove_from_queue()` 移除 | `clean_events()` 移除**等待中的** | 不直接修改（finally 释放） |
| **event_ids_to_events** | `remove_from_queue()` pop | `clean_events()` pop 等待中的 | 不直接修改 |
| **pending_event_ids_session** | **不动**（SSE 消费 ProcessCompletedMessage 时才 remove） | 仅移除**等待中的**，**执行中的保留** | 不修改（全部保留） |
| **pending_messages_per_session** | 注入 `ProcessCompletedMessage` | **完全不碰** | 注入 `UnexpectedErrorMessage` |
| **event.alive** | 不修改（cancel_tasks 触发 CancelledError） | 正在执行的标记为 False | 不修改 |
| **asyncio Task** | `cancel_tasks()` 精准取消（按 task name） | 不取消（自然完成） | `_cancel_asyncio_tasks()` 批量取消所有 |
| **SSE 关闭方式** | `CloseStreamMessage`（当 pending 变空时） | 直接 `return`（无 CloseStreamMessage） | 注入 `UnexpectedErrorMessage`（关闭条件永不触发，前端自行关闭） |
| **关闭条件触发** | `process_completed + pending == 0` | 不触发（直接 return） | `server_stopped` 条件永不触发 |
| **消息队列是否残留** | 无（优雅关闭） | 有（僵尸 AsyncQueue） | 有（不断注入错误消息） |
| **pending_event_ids_session 是否残留** | 无（清空后 pop） | 有（执行中的 event_id 残留） | 有（全部残留） |

---

## 8. 异常消息回传流程

异常发生在 `process_events()` 的多个位置，每类异常有不同的回传策略。

### 8.1 用户函数执行异常（首次调用）

位置：[queueing.py:882-897](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L882-L897)

```
call_process_api() 抛出异常 e
       │
       ├─ traceback.print_exc()  (非 Error 类 或 e.print_exception=True)
       │
       ├─ error_payload(err, show_error)
       │     → 若 show_error 或 AppError: content["error"] = str(e)
       │     → 否则: content["error"] = None  (隐藏内部错误)
       │     位置: utils.py:1711-1725
       │
       ├─ 对每个 awake_event 发送 ProcessCompletedMessage:
       │     output=content, success=False, title=content.get("title", "Error")
       │
       └─ await run_sync(compute_analytics_summary)
            → 更新分析统计
```

**状态**：异常后 `response = None`，跳过 `is_generating` 分支和正常完成分支，直接进入 finally 块。

### 8.2 生成式/流式函数的迭代异常

位置：[queueing.py:969-973](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L969-L973)

```
在 while is_generating 循环内:
  call_process_api() 抛出异常 e
       │
       ├─ traceback.print_exc()
       ├─ response = None, err = e
       │
       └─ 退出 while 循环 → 进入后续的完成处理:
            if response:
                ...  (不执行，response=None)
            else:
                success = False
                error = err or old_err
                output = error_payload(error, show_error)
                → 对每个 awake_event 发送 ProcessCompletedMessage(success=False)
              位置: queueing.py:975-1006
```

**与首次调用异常的区别**：迭代异常可以利用 `old_err`（上一轮的 err）作为兜底错误信息，因为在流式模式下 `err` 可能在上一次迭代中被设置。

### 8.3 UnexpectedErrorMessage（服务端意外错误）

位置：[server_messages.py:73-77](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/server_messages.py#L73-L77)

这不是用户函数抛出的，而是**框架层**注入的错误消息，出现场景：

| 场景 | 触发位置 | message |
|------|---------|---------|
| 队列停止 | [routes.py:1516-1520](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1516-L1520) | "Server stopped unexpectedly." |
| SSE 流异常 | [routes.py:1561-1564](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1561-L1564) | str(e) |
| session_not_found | 同上 | HTTPException 时 session_not_found=True |

前端处理：
```typescript
// submit.ts:512-528
type == "unexpected_error" || type == "broken_connection"
  → fire_event({ type: "status", stage: "error", message, broken, session_not_found })
```

### 8.4 BrokenConnection（SSE 连接中断）

位置：[stream.ts:78-88](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/stream.ts#L78-L88)

```
SSE 连接 onerror:
  → 遍历所有 event_callbacks，注入 { msg: "broken_connection", message: BROKEN_CONNECTION_MSG }
  → 每个 callback 收到后 → handle_message() → type="broken_connection"
  → fire_event({ stage: "error", broken: true })
```

**关键**：broken_connection 是前端 SSE 层面检测到的连接问题，不是服务端主动发送的。它会广播给该 session 下的**所有**未关闭事件。

### 8.5 异常回传消息类型对比

| 消息类型 | 触发者 | event_id | 发送方式 | 前端 stage |
|---------|--------|----------|---------|-----------|
| ProcessCompletedMessage(success=False) | 用户函数异常 | 有 | send_message() → AsyncQueue | "error" |
| UnexpectedErrorMessage | 框架/路由层 | 无或有 | 直接注入 AsyncQueue | "error" |
| broken_connection (前端构造) | SSE onerror | 无 | 前端直接回调 | "error" |

---

## 9. 全部事件结束后的收尾状态

当一个 session 的所有事件处理完毕后，涉及多层状态清理。

### 9.1 SSE 通道的优雅关闭

位置：[routes.py:1525-1559](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1525-L1559)

```
sse_stream() 收到 ProcessCompletedMessage(event_id=xxx):
       │
       ├─ if isinstance(message, ProcessCompletedMessage) and message.event_id:
       │   │
       │   ├─ 从 pending_event_ids_session[session_hash] 移除该 event_id
       │   │   (重复 cancel 安全：若已被移除则跳过)
       │   │
       │   └─ 判断是否应该关闭 SSE:
       │       条件: (message.msg == process_completed
       │              AND len(pending_event_ids_session[session_hash]) == 0)
       │       │
       │       ├─ YES → 构造 CloseStreamMessage
       │       │        yield process_msg(CloseStreamMessage)
       │       │        heartbeat_task.cancel()
       │       │        return  ← SSE 流结束
       │       │
       │       └─ NO  → 继续循环，等待其他事件完成
       │
       └─ 其他消息类型 → 只 yield，不移除 event_id，不判断关闭
```

**核心判定逻辑（精确说明）**：

关闭条件代码上写的是 `OR`，但实际上只有一个分支会触发：
- **分支 1**：`message.msg == ServerMessage.server_stopped` → **永不触发**（设计不匹配：`ServerMessage.server_stopped` = `"Server stopped unexpectedly."` 是字符串内容，而消息的 `msg` 字段是枚举值 `"unexpected_error"`）
- **分支 2**：`message.msg == process_completed AND len(pending_event_ids_session) == 0` → **正常路径唯一触发条件**

所以**实际生效的关闭条件只有一个**：当且仅当收到 `ProcessCompletedMessage` 且 `pending_event_ids_session[session_hash]` 变为空集时，才发送 `CloseStreamMessage` 关闭 SSE。

**关闭触发时机**：
- 正常完成：process_events 发送 ProcessCompletedMessage → SSE 消费 → 移除 event_id → 集合变空 → CloseStreamMessage
- 主动取消：/cancel 注入 ProcessCompletedMessage → SSE 消费 → 移除 event_id → 集合变空 → CloseStreamMessage
- 连接断开：不触发 CloseStreamMessage，直接 return
- 服务端停止：不触发 CloseStreamMessage，前端收到 unexpected_error 自行关闭

前端收到 `close_stream` 后：
```typescript
// stream.ts:43-46
if (_data.msg === "close_stream") {
    close_stream(stream_status, that.abort_controller);
    return;  // 停止 onmessage 处理
}
```
位置：[stream.ts:43-46](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/stream.ts#L43-L46)

### 9.2 process_events 的 finally 块：资源释放

位置：[queueing.py:1046-1080](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/queueing.py#L1046-L1080)

无论事件是成功、失败还是被取消，finally 块**必定执行**：

```
finally:
  │
  ├─ [Profiling] 若启用，收集 trace
  │
  ├─ event_queue.current_concurrency -= 1        # 释放并发组名额
  │    → 下一个 start_processing() 循环可以调度该组的新事件
  │
  ├─ start_times.remove(begin_time)               # 移除本次起始时间
  │    → 影响 broadcast_estimations() 的 ETA 计算
  │
  ├─ active_jobs[slot] = None                     # 释放全局工作槽位
  │    → start_processing() 可以分配新任务到该槽
  │    (可能 ValueError → 该 events 从未被放入 active_jobs 的特殊情况)
  │
  ├─ 对每个 event:
  │   ├─ reset_iterators(event._id)               # 清理生成器/迭代器
  │   │   → safe_aclose_iterator() 关闭异步迭代器
  │   │   → del app.iterators[event_id]
  │   │   → app.iterators_to_reset.add(event_id)  # 标记需要重置
  │   │   位置: queueing.py:1082-1097
  │   │
  │   ├─ 更新 event_analytics:
  │   │   event in awake_events → "success" 或 "failed"
  │   │   event not in awake_events → "cancelled"  (被 clean_events 标记 alive=False)
  │   │
  │   └─ compute_analytics_summary()              # 更新统计摘要
  │
  └─ [结束]
```

**reset_iterators 的设计意义**：
- 正常完成：迭代器已自然耗尽，reset 无副作用
- 被取消：迭代器可能还在中间状态，必须 `aclose` 后标记重置
- 若不重置：下一次调用同函数时可能从上次中断的位置继续，产生不可预期的行为

### 9.3 SSE 流的异常退出与清理

位置：[routes.py:1560-1572](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1560-L1572)

```
sse_stream() 的 except BaseException 分支:
       │
       ├─ 构造 UnexpectedErrorMessage(message=str(e), session_not_found=isinstance(HTTPException))
       ├─ yield 错误消息给客户端
       │
       ├─ if isinstance(e, asyncio.CancelledError):
       │     del pending_messages_per_session[session_hash]  # 直接删除消息队列
       │     await clean_events(session_hash=session_hash)    # 清理该 session 所有事件
       │
       └─ heartbeat_task.cancel()
            raise e  ← 重新抛出，让 FastAPI 处理
```

**注意**：`CancelledError` 是特殊处理——直接删除 session 的消息队列（不等 `pending_event_ids_session` 自然收缩），然后主动清理所有事件。这是最激进的清理路径。

### 9.4 session_not_found 场景

当 SSE 连接建立时，`pending_messages_per_session` 中可能已经没有该 session（例如服务端重启后客户端重连）：

```
sse_stream() 循环内:
  if session_hash not in pending_messages_per_session:
      raise HTTPException(404)
           │
           └─ 被 except BaseException 捕获
              → UnexpectedErrorMessage(session_not_found=True)
              → 前端收到 session_not_found 标记
              → 可据此判断需要重新建立 session
```

位置：[routes.py:1499-1505](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/gradio/routes.py#L1499-L1505)

### 9.5 前端收尾：iterator.close() 与状态重置

前端 `SubmitIterable` 的 close 函数：
```typescript
// submit.ts:640-647
function close(): void {
    done = true;
    while (resolvers.length > 0)
        (resolvers.shift())({ value: undefined, done: true });
}
```

触发 close() 的所有路径：

| 路径 | 代码位置 | 触发条件 |
|------|---------|---------|
| 正常完成 | [submit.ts:577-588](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L577-L588) | complete 状态 + data 已发送 |
| 错误完成 | [submit.ts:512-528](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L512-L528) | unexpected_error / broken_connection |
| SSE 流关闭 | [submit.ts:339-341](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L339-L341) | stream.onmessage 中 type=complete |
| HTTP 错误 | [submit.ts:444-486](file:///d:/fz/0601/solo-dogfeeding/code/239-gradio/client/js/src/utils/submit.ts#L444-L486) | POST /queue/data 返回 503/422/非200 |

close() 后，`done=true` 使后续的 `next()` 调用直接返回 `{done: true}`，所有 for-await-of 循环正常终止。

### 9.6 完整收尾状态变迁图

```
事件执行完毕 (成功/失败/取消)
       │
       ▼
process_events() finally 块
  ├─ current_concurrency--          → 并发组有空位
  ├─ active_jobs[slot] = None       → 全局槽位有空位
  ├─ start_times.remove()           → ETA 计算更新
  ├─ reset_iterators()              → 生成器归零
  └─ analytics 更新
       │
       ▼
send_message(ProcessCompletedMessage) → AsyncQueue
       │
       ▼
sse_stream() 取出消息
  ├─ yield SSE 数据给客户端
  ├─ pending_event_ids_session[session_hash].remove(event_id)
  └─ 判断: pending_event_ids_session[session_hash] 是否为空?
       │
       ├─ 不为空 → 继续等待其他事件
       │
       └─ 为空 → 发送 CloseStreamMessage
                 heartbeat_task.cancel()
                 return  ← SSE 流结束
                    │
                    ▼
              前端 close_stream()
                ├─ stream_status.open = false
                ├─ abort_controller.abort()
                └─ close() → done = true
                     │
                     ▼
              for-await-of 循环终止
              event_callbacks[event_id] 删除
              pending_diff_streams[event_id] 删除
              unclosed_events.delete(event_id)
```

---

## 10. 边界场景速查表

| 场景 | 触发路径 | pending_event_ids_session | pending_messages_per_session | SSE 关闭方式 | 消息回传 |
|------|---------|--------------------------|-----------------------------|-------------|---------|
| 用户点击取消 | POST /cancel | **SSE 消费时才 remove** | 注入 ProcessCompletedMessage(success=True) | CloseStreamMessage（集合变空时） | 伪造的 success=True 完成消息 |
| 关闭浏览器标签 | SSE is_disconnected | 仅移除等待中的，**执行中的保留** | **完全不碰**（僵尸队列） | 直接 return（无 CloseStreamMessage） | 无（连接已断） |
| 用户函数 raise | process_events except | SSE 消费 ProcessCompletedMessage 时 remove | 注入 ProcessCompletedMessage(success=False) | CloseStreamMessage（若集合变空） | error 内容（受 show_error 控制） |
| 生成式函数迭代异常 | while 循环内 except | SSE 消费时 remove | 注入 ProcessCompletedMessage(success=False) | CloseStreamMessage（若集合变空） | 使用 old_err 兜底 |
| 服务端停止 | Queue.stopped | **不修改（全部保留）** | 不断注入 UnexpectedErrorMessage | 不触发（前端自行关闭） | "Server stopped unexpectedly." |
| SSE 连接异常 | except BaseException | CancelledError 触发 clean_events | 非 CancelledError 保留，CancelledError 直接 del | 异常 return | UnexpectedErrorMessage(str(e)) |
| SSE CancelledError | except CancelledError | clean_events 移除 | **直接 del** 整个 AsyncQueue | 异常 return + re-raise | UnexpectedErrorMessage 后 re-raise |
| Session 不存在 | HTTPException(404) | 无（本就无状态） | 无 | 异常 return | UnexpectedErrorMessage(session_not_found=True) |
| 前端 SSE 断连 | stream.onerror | 不修改（服务端不知情） | 不修改（服务端不知情） | 前端 close_stream() | 前端构造 broken_connection，广播给所有 callbacks |
| 队列满 | push() 返回 queue_full | 不入队，不修改 | 不入队，不修改 | 无（不入队） | HTTP 503 + 前端 fire_error |
| 验证失败 | push() 返回 validator_error | 不入队，不修改 | 不入队，不修改 | 无（不入队） | HTTP 422 + 前端 fire_error |
