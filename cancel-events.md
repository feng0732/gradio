# Gradio 取消事件中断路径：五条路径的差异分析

取消事件在 Gradio 中有五条独立的中断路径，它们对队列事件的失效方式、取消信号的传递方式、以及资源回收的时机各不相同。本文从代码出发，逐一拆解每条路径的具体行为，并对比哪些路径让队列事件真正失效、哪些只发送取消信号。

---

## 核心概念：队列事件"失效"的三个层次

在分析五条路径之前，先明确队列事件失效有三个递进的层次：

| 层次 | 操作 | 效果 |
|------|------|------|
| **L1 等待队列移除** | 从 `EventQueue.queue` 列表中删除 Event 对象 | 事件永远不会被 `get_events()` 选中执行 |
| **L2 存活标志置假** | 将 `Event.alive` 设为 `False` | `send_message()` 直接返回不发送；`process_events()` 中 alive 检查点跳过该事件 |
| **L3 asyncio 任务取消** | 对 task 调用 `task.cancel()` | 在下一个 await 点注入 `CancelledError`，强制中断协程执行 |

**关键区别**：L1 和 L2 是**数据结构层面的失效**（事件从队列消失或被标记为死亡），L3 是**执行层面的中断**（运行中的代码被强制打断）。三者的组合决定了取消的实际效果。

---

## 路径一：用户主动取消（`cancels` 参数配置）

### 触发场景

开发者在事件监听器中通过 `cancels` 参数声明事件间的取消关系：

```python
click_event = btn.click(long_running_fn, inputs, outputs)
stop_btn.click(lambda: None, cancels=[click_event])
```

### 前端执行：`DependencyManager.cancel()`

入口在 [dependency.ts L338](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/js/core/src/dependency.ts#L338)：

```typescript
// dispatch() 中，事件执行前首先执行取消
this.cancel(dep.cancels);
```

[dependency.ts L811-L849](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/js/core/src/dependency.ts#L811-L849) 的 `cancel()` 方法执行四件事：

1. **`submission.cancel()`**：向 `/cancel` 发送 HTTP 请求
2. **`loading_stati.update({ status: "complete" })`**：立即更新 UI，用户感知上事件已结束
3. **`this.submissions.delete(id)`**：从前端 submissions Map 中删除
4. **触发 `.failure()` 和 `.then()` 链**：被取消事件的后续链式事件继续执行

### 后端执行：`/cancel` 路由

[routes.py L1401-L1429](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1401-L1429) 执行五个步骤：

```python
@router.post("/cancel")
async def cancel_event(body: CancelBody):
    # ① L3: 取消 asyncio 任务（仅发信号）
    await cancel_tasks({f"{body.session_hash}_{body.fn_index}"})
    
    # ② 检查会话和事件状态
    session_open = body.session_hash in blocks._queue.pending_messages_per_session
    event_running = body.event_id in blocks._queue.pending_event_ids_session.get(body.session_hash, {})
    
    # ③ L1: 从等待队列移除
    await blocks._queue.remove_from_queue(body.event_id)
    
    # ④ 发送完成消息让客户端正常断开
    if session_open and event_running:
        blocks._queue.pending_messages_per_session[body.session_hash].put_nowait(
            ProcessCompletedMessage(output={}, success=True, event_id=body.event_id)
        )
    
    # ⑤ 迭代器清理
    if body.event_id in app.iterators:
        async with app.lock:
            await safe_aclose_iterator(app.iterators[body.event_id])
            del app.iterators[body.event_id]
            app.iterators_to_reset.add(body.event_id)
```

### 本路径覆盖的失效层次

| 层次 | 是否覆盖 | 代码位置 |
|------|---------|---------|
| L1 等待队列移除 | ✅ | `remove_from_queue()` L1413 |
| L2 alive 置假 | ❌ | **不设置** `event.alive = False` |
| L3 任务取消信号 | ✅ | `cancel_tasks()` L1403 |

### 关键发现：`/cancel` 不设置 `event.alive = False`

这是最容易被忽略的细节。`/cancel` 路由中调用的是 `remove_from_queue()`，而非 `clean_events()`。两者的核心区别：

- `remove_from_queue()`：仅从等待队列列表和 `event_ids_to_events` 字典中删除，**不修改 `alive` 标志**
- `clean_events()`：既从队列删除，**又将 `alive` 设为 `False`**

这意味着：如果事件已经在执行中（已从等待队列移出、进入 `active_jobs`），`remove_from_queue()` 的 L1 操作实际上无效（事件已不在等待队列中），而 L2 操作（`alive = False`）又没有执行。此时唯一生效的是 L3 的 `cancel_tasks()` 信号。

**但** L3 信号对普通同步函数无效——`task.cancel()` 只在 `await` 点生效，如果 `call_process_api()` 内部正在运行一个不 yield 的同步函数，`CancelledError` 会被延迟到该函数执行完毕。

### `cancel_tasks()` 的信号机制

[utils.py L1102-L1115](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/utils.py#L1102-L1115)：

```python
async def cancel_tasks(task_ids: set[str]) -> list[str]:
    tasks = [(task, task.get_name()) for task in asyncio.all_tasks()]
    event_ids: list[str] = []
    matching_tasks = []
    for task, name in tasks:
        if "<gradio-sep>" not in name:
            continue
        task_id, event_id = name.split("<gradio-sep>")
        if task_id in task_ids:
            matching_tasks.append(task)
            event_ids.append(event_id)
            task.cancel()
    await asyncio.gather(*matching_tasks, return_exceptions=True)
    return event_ids
```

- 通过 task 命名约定 `{session_hash}_{fn_index}<gradio-sep>{event_id}` 定位目标任务
- `task.cancel()` 是**纯信号**：仅在下一个 `await` 点注入 `CancelledError`
- `asyncio.gather(..., return_exceptions=True)` 等待任务真正结束（吸收异常）

---

## 路径二：等待队列移除

### 触发场景

事件已进入 `EventQueue.queue` 排队，尚未被 `get_events()` 选中执行。此时取消只需要从队列列表中移除即可。

### 执行方法：`remove_from_queue()`

[queueing.py L470-L479](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L470-L479)：

```python
async def remove_from_queue(self, event_id: str):
    event = self.event_ids_to_events.get(event_id)
    if event:
        async with self.delete_lock:
            q = self.event_queue_per_concurrency_id[event.concurrency_id]
            try:
                q.queue.remove(event)
                self.event_ids_to_events.pop(event_id, None)
            except ValueError:
                pass
```

### 本路径覆盖的失效层次

| 层次 | 是否覆盖 | 说明 |
|------|---------|------|
| L1 等待队列移除 | ✅ | 从 `EventQueue.queue` 列表中删除 |
| L2 alive 置假 | ❌ | 不修改 `alive` |
| L3 任务取消信号 | ❌ | 无需取消（任务尚未创建） |

### 为什么不需要 L2 和 L3

事件在等待队列中时尚未创建 asyncio task（task 在 `start_processing()` 中才通过 `run_coro_in_background()` 创建），所以：
- 没有 task 可以 cancel
- `alive` 标志只在 `process_events()` 中被检查，而该函数不会被执行到这个事件

### 调用者

`remove_from_queue()` 只在 `/cancel` 路由中被直接调用。`clean_events()` 内部也做了类似的队列移除操作，但走的是不同的代码路径。

---

## 路径三：执行中任务取消

### 触发场景

事件已被 `get_events()` 取出、进入 `active_jobs`、对应的 asyncio task 正在运行 `process_events()`。

### `cancel_tasks()` 如何中断执行中的任务

`cancel_tasks()` 对执行中任务调用 `task.cancel()`，`CancelledError` 会在以下 await 点之一被注入：

**注入点 1**：[queueing.py L867](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L867)
```python
response = await route_utils.call_process_api(...)
```
这是首次调用用户函数的 await 点。如果用户函数是同步函数且正在 CPU 上执行，`CancelledError` 会**延迟**到函数执行完毕后的下一个 await 点。

**注入点 2**：[queueing.py L927](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L927)
```python
awake_events, closed_events = await Queue.wait_for_batch(awake_events, ...)
```
流式事件在等待新数据时的 await 点。

**注入点 3**：[queueing.py L957](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L957)
```python
response = await route_utils.call_process_api(...)
```
生成器/流式事件的后续调用。

### CancelledError 的传播路径

```
task.cancel()
    │
    ▼ (在 await 点注入 CancelledError)
    │
process_events() 内部
    │
    ├─► 内层 except Exception (L882/L969) 捕获
    │       └─► 发送错误消息、设置 response = None
    │
    └─► 外层 except Exception (L1043) 捕获
            └─► traceback.print_exc()
    
    finally (L1046):
        ├─► event_queue.current_concurrency -= 1  ─── 释放并发槽位
        ├─► active_jobs[index] = None             ─── 释放工作线程
        ├─► await reset_iterators(event._id)      ─── 迭代器重置
        └─► event_analytics 状态标记
```

**注意**：在 Python 3.9+ 中 `CancelledError` 是 `BaseException` 而非 `Exception`，此时 `except Exception` 不会捕获它，`CancelledError` 会直接穿透到 `finally` 块。无论哪种情况，`finally` 块**始终执行**，确保资源回收。

### `alive` 检查点：协作式退出的补充机制

除了 `CancelledError` 强制中断外，`process_events()` 还在两个位置检查 `alive` 标志：

**检查点 1**：[queueing.py L794-L806](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L794-L806)
```python
for event in events:
    if event.alive:
        self.send_message(event, ProcessStartsMessage(...))
        awake_events.append(event)
if not awake_events:
    return
```

**检查点 2**：[queueing.py L921-L923](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L921-L923)
```python
awake_events = [event for event in awake_events if event.alive]
if not awake_events:
    return
```

但如前所述，`/cancel` 路径**不设置 `alive = False`**，所以这两个检查点在用户主动取消场景下**不会触发**。执行中任务的中断完全依赖 `cancel_tasks()` 的 `CancelledError`。

### 本路径覆盖的失效层次

| 层次 | 是否覆盖 | 说明 |
|------|---------|------|
| L1 等待队列移除 | ❌（无效） | 事件已不在等待队列中 |
| L2 alive 置假 | ❌ | `/cancel` 不调用 `clean_events()` |
| L3 任务取消信号 | ✅ | `cancel_tasks()` 发出 `CancelledError` |

### 限制：同步函数无法立即中断

如果用户函数是一个长时间运行的同步函数（如 `time.sleep(60)`），`CancelledError` 只能等到该函数执行完毕、控制权回到事件循环后才能生效。这是 Python 协作式多任务的根本限制，不是 Gradio 的设计缺陷。

---

## 路径四：客户端断开清理

### 触发场景

浏览器标签页关闭、网络断开等导致 SSE 连接中断。

### SSE 流检测断开

[routes.py L1493-L1497](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1493-L1497)：

```python
async def sse_stream(request: fastapi.Request):
    # ...
    while True:
        if await request.is_disconnected():
            await blocks._queue.clean_events(session_hash=session_hash)
            heartbeat_task.cancel()
            return
```

### `clean_events()` 的完整行为

[queueing.py L631-L657](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L631-L657)：

```python
async def clean_events(
    self, *, session_hash: str | None = None, event_id: str | None = None
) -> None:
    # ① L2: 将 active_jobs 中匹配的事件标记为死亡
    for job_set in self.active_jobs:
        if job_set:
            for job in job_set:
                if job.session_hash == session_hash or job._id == event_id:
                    job.alive = False

    # ② L1: 从所有等待队列中移除匹配的事件
    async with self.delete_lock:
        events_to_remove: list[Event] = []
        for event_queue in self.event_queue_per_concurrency_id.values():
            for event in event_queue.queue:
                if event.session_hash == session_hash or event._id == event_id:
                    events_to_remove.append(event)

        for event in events_to_remove:
            self.event_queue_per_concurrency_id[event.concurrency_id].queue.remove(event)
            self.event_ids_to_events.pop(event._id, None)

        # ③ 清理 pending_event_ids_session
        if session_hash and session_hash in self.pending_event_ids_session:
            removed_ids = {e._id for e in events_to_remove}
            self.pending_event_ids_session[session_hash] -= removed_ids
            if not self.pending_event_ids_session[session_hash]:
                self.pending_event_ids_session.pop(session_hash, None)
```

### 本路径覆盖的失效层次

| 层次 | 是否覆盖 | 说明 |
|------|---------|------|
| L1 等待队列移除 | ✅ | 清除该会话所有等待中的事件 |
| L2 alive 置假 | ✅ | 标记所有执行中的事件为死亡 |
| L3 任务取消信号 | ❌ | **不调用 `cancel_tasks()`** |

### 关键差异：不发送取消信号，也不发送完成消息

客户端断开路径与用户主动取消路径有三个重要差异：

**差异 1：不调用 `cancel_tasks()`**

`clean_events()` 将 `alive` 设为 `False`，但不向 asyncio task 发送 `CancelledError`。执行中的任务会继续运行，直到：

- 遇到 `process_events()` 内的 `alive` 检查点（生成器循环中 L921），主动退出
- 用户函数自然执行完毕

此时 `send_message()` 被 `alive` 标志截断：

[queueing.py L240-L249](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L240-L249)：
```python
def send_message(self, event, event_message):
    if not event.alive:
        return                    # alive=False 时直接返回，不发送任何消息
    event_message.event_id = event._id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```

**差异 2：不发送 `ProcessCompletedMessage`**

`/cancel` 路由会向 SSE 流写入一个 `ProcessCompletedMessage`，让客户端优雅地关闭连接。客户端断开时 SSE 流已经断开，无需也无法发送完成消息。

**差异 3：不清理迭代器**

`clean_events()` 不清理 `app.iterators`。迭代器的清理由 `process_events()` 的 `finally` 块负责——当任务最终结束（自然完成或通过 alive 检查点退出）时，`finally` 块中的 `reset_iterators()` 会清理迭代器。

### CancelledError 路径中的二次清理

[routes.py L1560-L1568](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1560-L1568)：

```python
except BaseException as e:
    # ...
    if isinstance(e, asyncio.CancelledError):
        del blocks._queue.pending_messages_per_session[session_hash]
        await blocks._queue.clean_events(session_hash=session_hash)
```

当 SSE 流本身被取消（如服务器关闭时），会再次调用 `clean_events()`，并删除整个会话的消息队列。

### 会话级 vs 事件级清理

`clean_events()` 支持两种粒度：
- `session_hash`：清理该会话的**所有**事件（客户端断开场景）
- `event_id`：清理**单个**事件（但目前没有代码使用 `event_id` 参数调用 `clean_events()`）

---

## 路径五：迭代器释放

### 两种迭代器清理时机

迭代器释放不是一条独立的"取消路径"，而是取消发生后的**资源回收步骤**。它有两个触发时机：

### 时机 1：`/cancel` 路由中的立即清理

[routes.py L1421-L1428](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1421-L1428)：

```python
if body.event_id in app.iterators:
    async with app.lock:
        try:
            await safe_aclose_iterator(app.iterators[body.event_id])
        except Exception:
            pass
        del app.iterators[body.event_id]
        app.iterators_to_reset.add(body.event_id)
```

这是**即时清理**：取消请求到达后立即关闭迭代器。对于生成器函数，这意味着生成器的 `aclose()` 方法被调用，生成器内的 `finally` 块得以执行。

### 时机 2：`process_events()` finally 块中的兜底清理

[queueing.py L1067-L1072](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L1067-L1072)：

```python
finally:
    for event in events:
        await self.reset_iterators(event._id)
```

[queueing.py L1082-L1097](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L1082-L1097)：

```python
async def reset_iterators(self, event_id: str):
    app = self.server_app
    if event_id not in app.iterators:
        return                    # 如果 /cancel 已清理过，直接返回
    async with app.lock:
        try:
            await safe_aclose_iterator(app.iterators[event_id])
        except Exception:
            pass
        del app.iterators[event_id]
        app.iterators_to_reset.add(event_id)
```

这是**兜底清理**：无论任务是成功完成、被取消、还是异常退出，`finally` 块都会执行。如果 `/cancel` 已经清理了迭代器，`reset_iterators()` 发现 `event_id not in app.iterators` 就会直接返回。

### 两阶段清理的必要性

| 场景 | 时机1（/cancel立即清理） | 时机2（finally兜底清理） |
|------|------------------------|------------------------|
| 用户主动取消 + 生成器正在 yield | ✅ 立即关闭生成器 | ✅ 兜底确认（已清理则跳过） |
| 用户主动取消 + 普通函数执行中 | ❌ 无迭代器可清理 | ✅ 函数结束后清理 |
| 客户端断开 | ❌ 不清理迭代器 | ✅ 任务结束后清理 |
| 任务正常完成 | ❌ 无需取消 | ✅ 清理迭代器 |

**用户主动取消时立即清理迭代器**的原因：生成器函数可能正在 `yield` 中等待，`safe_aclose_iterator()` 调用生成器的 `aclose()`，触发生成器内部的 `finally` 块，使其能够释放内部资源（如数据库连接、文件句柄等）。

---

## 五条路径对比总表

| 维度 | 路径一：用户主动取消 | 路径二：等待队列移除 | 路径三：执行中任务取消 | 路径四：客户端断开 | 路径五：迭代器释放 |
|------|-------------------|-------------------|-------------------|----------------|----------------|
| **触发方式** | `cancels` 参数 | 事件在等待队列中 | 事件在 `active_jobs` 中 | SSE 连接断开 | 取消/完成后的资源回收 |
| **L1 队列移除** | ✅ `remove_from_queue()` | ✅ 队列列表删除 | ❌（已不在队列） | ✅ `clean_events()` 内部 | — |
| **L2 alive 置假** | ❌ | ❌（无需） | ❌ | ✅ `clean_events()` 内部 | — |
| **L3 取消信号** | ✅ `cancel_tasks()` | ❌（无 task） | ✅ `cancel_tasks()` | ❌ | — |
| **发送完成消息** | ✅ `ProcessCompletedMessage` | — | — | ❌ | — |
| **迭代器清理** | ✅ 立即清理 | — | — | ❌（由 finally 兜底） | ✅ 两阶段 |
| **对同步函数** | 延迟生效 | N/A | 延迟生效 | 延迟生效 | N/A |
| **对生成器函数** | 立即中断 | N/A | 在 yield 点中断 | 在 yield 点中断 | 立即关闭生成器 |

---

## 哪些路径让队列事件失效，哪些只发信号

### 让队列事件失效的路径（数据结构层面移除或标记死亡）

**`remove_from_queue()`**（路径一、二的底层操作）：
- 从 `EventQueue.queue` 列表中删除 Event 对象
- 从 `event_ids_to_events` 字典中删除映射
- **效果**：事件从队列数据结构中彻底消失，`get_events()` 永远不会选中它

**`clean_events()`**（路径四的底层操作）：
- 包含 `remove_from_queue()` 的全部效果
- **额外**将 `active_jobs` 中匹配事件的 `alive` 设为 `False`
- **额外**清理 `pending_event_ids_session`
- **效果**：事件不仅从数据结构中消失，执行中的事件也被标记为"死亡"，`send_message()` 和 `alive` 检查点都会跳过它

### 只发取消信号的路径（不修改队列数据结构）

**`cancel_tasks()`**（路径一、三的信号操作）：
- 遍历所有 asyncio task，按名称匹配目标
- 调用 `task.cancel()` 注入 `CancelledError`
- **不修改** `EventQueue.queue`、`event_ids_to_events`、`Event.alive` 中的任何一个
- **效果**：仅在 asyncio 协程的 await 点生效，如果任务不在 await 点则无效

### 信号与失效的组合关系

```
路径一（用户主动取消）= cancel_tasks()（纯信号）+ remove_from_queue()（L1失效）
                          │                        │
                          │                        └─ 事件在等待队列：有效移除
                          │                        └─ 事件在执行中：无效（已不在队列）
                          │
                          └─ 事件在执行中：CancelledError 注入
                          └─ 事件在等待中：无 task 可取消，信号无目标

路径四（客户端断开）= clean_events()（L1+L2失效，无信号）
                          │
                          ├─ 事件在等待队列：有效移除
                          ├─ 事件在执行中：alive=False，send_message() 被截断
                          └─ 无 CancelledError，同步函数继续运行直到自然结束
```

---

## 流式事件的特殊取消路径

流式事件（`connection="stream"`）有一条额外的取消路径，不经过 `/cancel` 路由：

[routes.py L1096-L1102](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1096-L1102)：

```python
@router.post("/stream/{event_id}/close")
async def _(event_id: str):
    event = app.get_blocks()._queue.event_ids_to_events[event_id]
    event.run_time = math.inf
    event.closed = True
    event.signal.set()
    return {"msg": "success"}
```

### 本路径覆盖的失效层次

| 层次 | 是否覆盖 | 说明 |
|------|---------|------|
| L1 等待队列移除 | ❌ | 不修改队列 |
| L2 alive 置假 | ❌ | 不修改 alive |
| L3 任务取消信号 | ❌ | 不调用 task.cancel() |

这条路径**不使队列事件失效，也不发送取消信号**。它通过设置 `event.closed = True` 和 `event.signal.set()` 让流式事件的 `wait_for_batch()` 超时返回，使 `process_events()` 在下一次循环判断 `event.is_finished` 为 `True` 而正常结束。

这是唯一一条**不中断、不失效，只引导事件自然结束**的路径。

---

## `send_message()` 的 alive 守卫

所有路径中，`send_message()` 是消息到达客户端的必经之路。它有一个关键的 `alive` 守卫：

[queueing.py L240-L249](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L240-L249)：

```python
def send_message(self, event, event_message):
    if not event.alive:
        return
    # ...
```

这意味着：
- **路径一**（用户取消）没有设置 `alive = False`，但通过 `cancel_tasks()` 使任务在 await 点中断，`process_events()` 不会再调用 `send_message()`
- **路径四**（客户端断开）设置了 `alive = False`，即使任务仍在运行，所有 `send_message()` 调用都会被截断——这是"静默杀死"执行中任务的方式，任务继续运行但输出被丢弃

---

## `process_events()` finally 块：所有路径的统一终点

无论通过哪条路径取消，只要 `process_events()` 的 task 最终结束（无论是被 CancelledError 中断、alive 检查点退出、还是自然完成），`finally` 块都会执行统一的资源回收：

[queueing.py L1046-L1080](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L1046-L1080)：

```python
finally:
    # 释放并发槽位
    event_queue.current_concurrency -= 1
    # 清除启动时间记录
    start_times = event_queue.start_times_per_fn[fn]
    if begin_time in start_times:
        start_times.remove(begin_time)
    # 释放工作线程
    self.active_jobs[self.active_jobs.index(events)] = None
    # 迭代器重置（兜底清理）
    for event in events:
        await self.reset_iterators(event._id)
        # 分析标记
        if event in awake_events:
            self.event_analytics[event._id]["status"] = "success" if success else "failed"
        else:
            self.event_analytics[event._id]["status"] = "cancelled"
```

这保证了无论取消路径如何，并发控制（`current_concurrency`）、工作线程（`active_jobs`）和迭代器资源都能被正确回收。

---

## 附录：事件映射与待处理集合生命周期

取消路径之所以让人难以理解，一个重要原因是 Gradio 用了三套独立的、生命周期各不相同的集合/映射。同一个"事件"在不同数据结构中被添加、移除、标记的时机完全不一致。本节按代码执行顺序，逐一拆解六个关键场景下，三套数据结构的增删改。

### 三套核心数据结构

| 数据结构 | 类型 | 作用 | 存活周期 |
|---------|------|------|---------|
| `event_ids_to_events` | `dict[str, Event]` | event_id → Event 对象的全局映射 | 最长：入队时添加，仅取消/断开时移除，**正常完成不删除** |
| `pending_event_ids_session` | `dict[str, set[str]]` | 每个会话待处理事件 ID 集合 | 入队时添加，SSE 消费端消费完成消息时移除 |
| `pending_messages_per_session` | `LRUCache[str, AsyncQueue[EventMessage]]` | 每个会话的 SSE 消息队列 | 会话首次入队时创建，SSE 流收到 CancelledError 时删除 |

> **反直觉的一点**：事件正常完成后，`event_ids_to_events` 中的映射不会被删除。只有取消路径（`remove_from_queue` 或 `clean_events`）才会从映射中移除事件。

---

### 场景一：事件入队（`Queue.push()`）

代码位置：[queueing.py L373-L468](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L373-L468)

**执行顺序**：

```
① 创建 Event 对象 (L373-L379)
    │
    ▼
② pending_messages_per_session[session_hash] = AsyncQueue()  (L383-L384)
    └─ 仅当该会话首次入队时创建，后续复用
    │
    ▼
③ pending_event_ids_session[session_hash].add(event._id)  (L387)
    └─ 加入待处理事件 ID 集合
    │
    ▼
④ event_ids_to_events[event._id] = event  (L388)
    └─ 加入全局事件映射
    │
    ▼
⑤ event_queue.queue.append(event)  (L458)
    └─ 加入等待队列
    │
    ▼
⑥ event_analytics[event._id] = {status: "queued", ...}  (L459-L465)
```

**各数据结构的变化**：

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | ✅ 添加（append） | L458 |
| `event_ids_to_events` | ✅ 添加 | L388 |
| `pending_event_ids_session` | ✅ 添加（add） | L387 |
| `pending_messages_per_session` | ✅ 创建（会话首次） | L383-L384 |
| `Event.alive` | — | 初始值 True |

> **要点**：事件一创建就加入了三套数据结构，而非等到进入执行阶段。这就是为什么 `/cancel` 路由可以通过 event_id 立即找到事件。

---

### 场景二：取出执行（`start_processing()` → `get_events()` → `process_events()`）

代码位置：[queueing.py L496-L556](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L496-L556)

**执行顺序**：

```
get_events() (L496-L519):
  ① 从 event_queue.queue.remove(event)  (L517)
      └─ 出等待队列
  ② 返回 events 列表

start_processing():
  ③ active_jobs[index] = events  (L538)
      └─ 入执行中列表
  ④ event_queue.current_concurrency += 1  (L540)
      └─ 并发计数+1
  ⑤ event_analytics[event._id]["status"] = "processing"  (L544)
      └─ 分析状态更新
  ⑥ 创建 asyncio task 运行 process_events()  (L545-L553)
```

**各数据结构的变化**：

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | ❌ 移除 | L517 |
| `event_ids_to_events` | — 保留 | 不修改 |
| `pending_event_ids_session` | — 保留 | 不修改 |
| `pending_messages_per_session` | — 保留 | 不修改（后续 `send_message()` 向其中投递消息） |
| `active_jobs` | ✅ 加入 | L538 |
| `Event.alive` | — | 仍为 True |

> **关键发现**：事件从等待队列取出后，仍然保留在 `event_ids_to_events` 和 `pending_event_ids_session` 中。这两套数据结构在执行阶段**只增不减**，除非遇到取消或断开。

---

### 场景三：用户取消（`/cancel` 路由）

代码位置：[routes.py L1401-L1429](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1401-L1429)

**执行顺序**：

```
① cancel_tasks({session_hash}_{fn_index})  (L1403)
    └─ 向匹配的 asyncio task 发送 CancelledError 信号
    │
    ▼
② remove_from_queue(event_id)  (L1413)
    │  ├─ 从 event_queue.queue 移除事件   ← 仅当事件仍在等待队列
    │  └─ event_ids_to_events.pop(event_id)  ← 从全局映射删除
    │
    ▼
③ pending_messages_per_session[session_hash].put_nowait(ProcessCompletedMessage)
    │  (L1418-L1420)
    ├─ 塞入一条空的完成消息，让 SSE 消费端正常断开
    └─ 仅当 session_open 且 event_running 时执行
    │
    ▼
④ 迭代器清理 (L1421-L1428)
    ├─ safe_aclose_iterator(app.iterators[event_id])
    ├─ del app.iterators[event_id]
    └─ app.iterators_to_reset.add(event_id)
```

**各数据结构的变化**（分两种情况）：

#### 情况 A：事件仍在等待队列

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | ❌ 移除 | `remove_from_queue` L476 |
| `event_ids_to_events` | ❌ 移除（pop） | `remove_from_queue` L477 |
| `pending_event_ids_session` | — 保留 | **不修改** |
| `pending_messages_per_session` | ✅ 塞入完成消息 | L1418-L1420 |
| `Event.alive` | — 保留 | 不修改，仍为 True |

#### 情况 B：事件已在执行中

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | —（已不在队列） | `ValueError` 被静默捕获 |
| `event_ids_to_events` | — 保留 | **不修改** |
| `pending_event_ids_session` | — 保留 | 不修改 |
| `pending_messages_per_session` | ✅ 塞入完成消息 | L1418-L1420 |
| `Event.alive` | — 保留 | 不修改，仍为 True |

> **最反直觉的发现**：
> - `/cancel` 路由不修改 `pending_event_ids_session`，事件的 ID 仍然留在待处理集合中
> - 对于执行中事件，`event_ids_to_events` 也不会被删除
> - 这两套数据结构的真正清理，发生在 **SSE 消费端消费掉那条被塞入的完成消息时**

---

### 场景四：客户端断开（SSE `is_disconnected`）

代码位置：[routes.py L1493-L1497](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1493-L1497)

**执行顺序**：

```
① request.is_disconnected() 检测到断开
    │
    ▼
② clean_events(session_hash=session_hash)  (L1495)
    │
    ├─ 遍历 active_jobs，匹配事件设 alive=False  (L634-L638)
    │   └─ 标记执行中事件为死亡
    │
    ├─ 遍历所有 event_queue.queue，收集匹配事件到 events_to_remove (L642-L645)
    │   └─ 仅收集等待队列中的事件，不包含执行中的
    │
    ├─ 对 events_to_remove 中的每个事件：
    │   ├─ event_queue.queue.remove(event)  ← 出等待队列
    │   └─ event_ids_to_events.pop(event._id)  ← 出全局映射
    │
    └─ pending_event_ids_session[session_hash] -= removed_ids  (L655)
        ├─ 仅减去等待队列中的事件 ID
        └─ 若集合为空则删除整个会话条目
    │
    ▼
③ heartbeat_task.cancel()  (L1496)
    │
    ▼
④ return —— 函数返回，SSE 流结束
```

**各数据结构的变化**（分两部分）：

#### 等待队列中的事件

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | ❌ 移除 | `clean_events` L648 |
| `event_ids_to_events` | ❌ 移除（pop） | `clean_events` L651 |
| `pending_event_ids_session` | ❌ 移除（集合减法） | `clean_events` L655 |

#### 执行中的事件

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | —（已不在队列） | — |
| `event_ids_to_events` | — 保留 | **不修改** |
| `pending_event_ids_session` | — 保留 | **不修改** |
| `Event.alive` | ❌ 设为 False | `clean_events` L638 |
| `active_jobs` | — 仍在列表中 | 不删除，仅 alive 标志变化 |

> **重要发现**：`clean_events()` 只清理等待队列中的事件。执行中的事件仅被标记 `alive=False`，但它们的 ID 仍然保留在 `event_ids_to_events` 和 `pending_event_ids_session` 中——事件被"杀死"了，但映射还在。
>
> 原因在于 `clean_events()` 的 `events_to_remove` 只从 `event_queue.queue` 列表中收集，而执行中的事件已被 `get_events()` 取出，不在此列表内。

---

### 场景五：流式关闭（`/stream/{event_id}/close`）

代码位置：[routes.py L1096-L1102](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1096-L1102)

**执行顺序**：

```
① event_ids_to_events[event_id]  (L1098)
    └─ 从全局映射取出事件
    │
    ▼
② event.run_time = math.inf  (L1099)
    └─ 运行时间设为无穷大
    │
    ▼
③ event.closed = True  (L1100)
    └─ 标记流已关闭
    │
    ▼
④ event.signal.set()  (L1101)
    └─ 唤醒等待中的 process_events 协程
```

**各数据结构的变化**：

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `EventQueue.queue` | — | 不修改 |
| `event_ids_to_events` | — 保留 | 不删除 |
| `pending_event_ids_session` | — 保留 | 不修改 |
| `Event.closed` | ✅ 设为 True | L1100 |
| `Event.signal` | ✅ set() | L1101 |
| `Event.alive` | — | 仍为 True |

这是最"轻量级"的取消方式，只修改 Event 对象的内部状态，不触碰任何集合或映射。事件在所有数据结构中仍然保留。

---

### 场景六：完成消息消费（SSE 消费端）

代码位置：[routes.py L1525-L1559](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1525-L1559)

这是最容易被忽略的清理点：`pending_event_ids_session` 的真正移除发生在这里，而非队列或 `process_events()` 端。

**执行顺序**：

```
sse_stream() 循环消费 pending_messages_per_session[session_hash].get()
    │
    ▼
收到 ProcessCompletedMessage（携带 event_id）
    │
    ▼
① 检查 event_id 是否在 pending_event_ids_session[session_hash] 中  (L1532-L1538)
    │  └─ 注释说明：可能已被移除，例如重复发送 /cancel 请求
    │
    ▼
② pending_event_ids_session[session_hash].remove(event_id)  (L1540-L1542)
    └─ 从待处理集合中移除
    │
    ▼
③ 若 pending_event_ids_session[session_hash] 为空，或收到 server_stopped
    │  (L1543-L1553)
    │
    ▼
④ 发送 CloseStreamMessage + heartbeat_task.cancel() + return  (L1554-L1559)
    └─ 关闭 SSE 流
```

**各数据结构的变化**：

| 数据结构 | 操作 | 代码位置 |
|---------|------|---------|
| `event_ids_to_events` | — 保留 | **不删除** |
| `pending_event_ids_session` | ❌ 移除（remove） | L1540-L1542 |
| `pending_messages_per_session` | —（消息被消费取出） | `messages.get()` |
| `Event.alive` | — | 不修改 |

> **最关键的发现**：`event_ids_to_events` 在事件正常完成后**永远不会被删除**！它只在取消路径（`remove_from_queue` 和 `clean_events`）中被删除。
>
> 这意味着：
> - 正常完成的事件，其 Event 对象引用会一直保留在 `event_ids_to_events` 中
> - 只有被取消的事件，才会从 `event_ids_to_events` 中被移除
>
> 这是 Gradio 事件生命周期管理中非常反直觉的设计。

---

### 生命周期全景总表

下表汇总六个场景下三套核心数据结构的变化：

| 场景 | `EventQueue.queue` | `event_ids_to_events` | `pending_event_ids_session` | `Event.alive` |
|------|---------------------|------------------------|---------------------------|---------------|
| **事件入队** | ✅ 添加 | ✅ 添加 | ✅ 添加 | True |
| **取出执行** | ❌ 移除 | — 保留 | — 保留 | True |
| **用户取消（等待中）** | ❌ 移除 | ❌ 移除 | — 保留 | 不修改，仍为 True |
| **用户取消（执行中）** | —（已不在队列） | — 保留 | — 保留 | 不修改，仍为 True |
| **客户端断开（等待中）** | ❌ 移除 | ❌ 移除 | ❌ 移除 | —（对象已被移除） |
| **客户端断开（执行中）** | —（已不在队列） | — 保留 | — 保留 | ❌ False |
| **流式关闭** | — | — 保留 | — 保留 | True，`closed=True` |
| **完成消息消费** | — | — 保留（关键） | ❌ 移除 | — |

符号说明：✅ 添加 / ❌ 移除 / — 不修改

---

### 为什么 `event_ids_to_events` 不清理？

从代码来看，`event_ids_to_events` 中键的删除只有两条路径：

1. `remove_from_queue()` — 仅被 `/cancel` 路由调用
2. `clean_events()` — 仅被客户端断开检测逻辑调用

事件正常完成后，`event_ids_to_events` 中的 Event 对象**不会被删除**。这可能是设计上的权衡：

- 在流式事件（`connection="stream"`）场景下，`/stream/{event_id}` 和 `/stream/{event_id}/close` 路由需要通过 event_id 查找 Event 对象
- 事件完成后，客户端可能还有后续的流操作需要访问事件信息

> **潜在问题**：从代码看，正常完成后的 `event_ids_to_events` 条目没有明确的清理机制，也不依赖 LRU 或其他间接清理。在低并发、短会话的场景下可以接受，但在高并发长运行环境下可能成为内存泄漏点。

---

## 补充：心跳断开时的会话清理流程

除了客户端通过 SSE 流检测断开之外，Gradio 还有一条独立的心跳机制用于会话清理路径——`/heartbeat/{session_hash}` 长连接。当浏览器标签页关闭或网络中断时，心跳连接同样会触发 CancelledError，进而执行一套更完整的会话清理流程。

### 心跳路由入口：[/heartbeat/{session_hash}](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1195-L1264)

### 心跳连接的生命周期

```
正常心跳循环（routes.py L1207-L1220）
  │
  ├─ yield "data: ALIVE\n\n"               ← 定期发送心跳包
  ├─ asyncio.sleep(heartbeat_rate)         ← 等待下一次心跳
  └─ 同时等待 stop_stream_task             ← 监听服务器关闭信号
       │
       └─ 若服务器关闭 → 主动 raise CancelledError
```

当客户端断开连接时，`iterator()` 生成器被垃圾回收或 `yield` 操作失败，`asyncio.CancelledError` 被抛出，进入 `except` 块执行清理流程：

### 完整清理步骤（[routes.py L1221-L1264](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1221-L1264)）

```
① stop_stream_task.cancel()  (L1222-L1223)
    └─ 停止监听服务器关闭信号
    │
    ▼
② 执行 unload 事件（L1234-L1249）
    ├─ 遍历 app.get_blocks().fns.items()，筛选出 targets == "unload" 的函数
    │    └─ 即开发者通过 gr.Interface(unload_fn=...) 注册的会话卸载函数
    └─ 用 background_tasks.add_task() 在后台执行每个 unload 函数
    │
    ▼
③ 标记会话数据为关闭状态（L1251-L1252）
    └─ app.state_holder.session_data[session_hash].is_closed = True
        └─ 注释说明：标记状态将在一小时后被删除
    │
    ▼
④ 清空会话缓存（L1253）
    └─ caching.clear_session_caches(session_hash)
        └─ 遍历 _per_session_stores，对每个 store.clear_session(session_hash)
    │
    ▼
⑤ 唤醒所有待处理事件（L1254-L1263）
    ├─ 遍历 pending_event_ids_session[session_hash] 中所有待处理 event_id
    ├─ event.run_time = math.inf   ← 设运行时间为无穷大
    └─ event.signal.set()          ← 唤醒正在 wait_for_batch 的协程
```

### 各步骤的详细说明

#### 步骤 ②：执行 unload 事件

[routes.py L1234-L1249](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1234-L1249)

`unload_fn_indices` 的筛选逻辑：

```python
unload_fn_indices = [
    i
    for i, dep in app.get_blocks().fns.items()
    if any(t for t in dep.targets if t[1] == "unload")
]
```

- 这是开发者通过 `gr.Interface(unload_fn=...)` 或 `block.unload(fn)` 注册的会话卸载钩子
- 使用 `background_tasks.add_task()` 在后台执行，不阻塞心跳路由立即返回
- **关键**：心跳连接本身已经被 CancelledError 打断，但 unload 函数会被放入 FastAPI 的 BackgroundTasks 异步执行

#### 步骤 ③：关闭会话状态

[routes.py L1251-L1252](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1251-L1252)

```python
if session_hash in app.state_holder.session_data:
    app.state_holder.session_data[session_hash].is_closed = True
```

- 这只是**标记**为关闭，而非立即删除
- 注释说明"这会标记状态在一小时后被删除"——具体的过期清理由另一个定时任务负责
- 在状态被真正删除前，`session_data` 对象仍然存在，只是 `is_closed` 标志位为 True

#### 步骤 ④：清空会话缓存

[routes.py L1253](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1253) → [caching.py L339-L341](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/caching.py#L339-L341)

```python
def clear_session_caches(session_hash: str | None) -> None:
    for store in list(_per_session_stores):
        store.clear_session(session_hash)
```

- `_per_session_stores` 是所有使用 `@gr.cache()` 装饰器创建的缓存存储集合
- 遍历所有缓存存储，移除该 session_hash 的缓存条目
- 这是**立即生效**的操作，与状态标记不同

#### 步骤 ⑤：唤醒所有待处理事件

[routes.py L1254-L1263](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/routes.py#L1254-L1263)

```python
for event_id in app.get_blocks()._queue.pending_event_ids_session.get(session_hash, []):
    event = app.get_blocks()._queue.event_ids_to_events[event_id]
    event.run_time = math.inf
    event.signal.set()
```

这是最值得注意的一步：

- **所有待处理事件**（不仅仅是取消或执行中的）都被设置了 `run_time = math.inf` 和 `event.signal.set()`
- 设置 `run_time = math.inf` 的目的是让该事件在 `process_events()` 的 `wait_for_batch()` 中立即超时
- `event.signal.set()` 唤醒任何正在 `await event.signal.wait()` 的协程
- 配合起来的效果：所有该会话的事件都会立即被 `process_events()` 在下一次循环检查 `event.is_finished` 判断为 `True`，从而正常退出
- 这是一条**不发送取消信号，不删除事件，也不修改 `alive` 标志**的**软终止**路径

### 心跳断开 vs SSE 断开的对比

| 维度 | SSE 断开 | 心跳断开 |
|------|---------|---------|
| 触发点 | `request.is_disconnected()` 检测 | 心跳长连接 CancelledError |
| 清理 unload 事件 | ❌ 不执行 | ✅ 后台执行所有 unload 函数 |
| 会话状态 | ❌ 不标记 | ✅ `is_closed = True` |
| 会话缓存 | ❌ 不清空 | ✅ `clear_session_caches()` |
| 唤醒事件 | ❌ 不唤醒 | ✅ 所有待处理事件全部被 signal.set() |
| `clean_events()` | ✅ 调用 | ❌ 不调用 |
| `alive=False` | ✅ 标记执行中事件 | ❌ 不修改 alive |
| `CancelledError` 发送 | ❌ 不发送 | ❌ 不发送（通过 `run_time=math.inf` 软终止） |

> **关键点**：心跳断开和 SSE 断开是两条**独立且互补**的路径。
> - SSE 断开通过 `clean_events()` 标记 `alive=False` 并从队列中删除事件
> - 心跳断开通过 unload、缓存清理、run_time=math.inf 让事件自然退出
> - 实际浏览器关闭标签页时，两条路径都会被触发（SSE 和 心跳是两个独立的长连接）

---

## 补充：任务命名限制与批处理取消失效

`cancel_tasks()` 的核心机制是通过 asyncio task 的**名称**来匹配目标任务。但这套命名规则有一个关键限制：**批处理（batch=True）的任务永远不会被取消**。

### 任务命名规则

#### 命名设置函数：[set_task_name()](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/utils.py#L1118-L1120)

```python
def set_task_name(task, session_hash: str, fn_index: int, event_id: str, batch: bool):
    if not batch:
        task.set_name(f"{session_hash}_{fn_index}<gradio-sep>{event_id}")
```

调用位置：[queueing.py L548-L553](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L548-L553)

```python
set_task_name(
    process_event_task,
    events[0].session_hash,
    events[0].fn._id,
    events[0]._id,
    batch,  # ← 关键参数
)
```

#### 命名匹配规则：[cancel_tasks()](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/utils.py#L1102-L1115)

```python
async def cancel_tasks(task_ids: set[str]) -> list[str]:
    tasks = [(task, task.get_name()) for task in asyncio.all_tasks()]
    event_ids: list[str] = []
    for task, name in tasks:
        if "<gradio-sep>" not in name:
            continue          # ← 关键过滤条件
        task_id, event_id = name.split("<gradio-sep>")
        if task_id in task_ids:
            matching_tasks.append(task)
            event_ids.append(event_id)
            task.cancel()
```

### 批处理任务名不匹配的根本原因

当 `batch=True` 时：

1. `set_task_name()` 的 `if not batch:` 条件为 `False`
2. `task.set_name()` **不会被调用**
3. task 保持 asyncio 默认的任务名（通常类似 `"Task-123"`）
4. `cancel_tasks()` 检查 `"<gradio-sep>" not in name` → `True`
5. `continue` 跳过，任务不会被匹配和取消

### 为什么批处理不设置任务名的深层原因

查看调用点 [queueing.py L548-L553](file:///d:/fz/0601/solo-dogfeeding/code/260-gradio/gradio/queueing.py#L548-L553)：

```python
set_task_name(
    process_event_task,
    events[0].session_hash,   # ← 注意：只用了 events[0]
    events[0].fn._id,      #    只用了第一个事件的信息
    events[0]._id,           #    只用了第一个事件的 ID
    batch,
)
```

批处理时，一个 task 处理的是**一批 events 列表中的多个事件**（`process_events(events, batch, ...)`）。如果设置任务名，只能用 `events[0]` 的信息，但这个名字无法代表整批事件。

- 若用 `events[0]` 的 `session_hash_fn_index` 作为 task_id，那么：
  - 其他会话的同一批事件会被错误匹配
  - 取消时只能匹配到第一个事件的信息
  - 无法区分同批的其他事件

这是一个设计权衡：批处理任务共享同一个协程，无法用单一事件的信息来命名，所以干脆**不设置**名称。

### 批处理取消失效的影响

| 场景 | 是否能取消 | 原因 |
|------|-----------|------|
| 普通非批处理任务 | ✅ 可取消 | 有 `<gradio-sep>` 分隔符，可匹配 |
| 批处理任务 | ❌ 不可取消 | 无自定义名称，`cancel_tasks()` 跳过 |

**影响范围**：
- 批处理中的事件，当用户点击取消按钮（`cancels` 参数）时，`/cancel` 路由仍然会调用 `cancel_tasks()`，但**找不到匹配的任务**
- 此时 `/cancel` 的 L1（队列移除）仍然生效——如果事件还在等待队列中，会被移除
- 如果事件已经在执行中（已进入 `process_events()` 的批处理），则**无法取消**
- `event.alive` 标志位依然不会被设置（和普通取消一样，`/cancel` 不调用 `clean_events()`）
- 事件会继续执行到完成，**不会被 `alive` 标志截断**（因为 `alive` 仍为 True）
- 最终 `process_events()` 的 `finally` 块会正常释放资源

> **设计取舍**：这是已知的限制。批处理的取消是 Gradio 目前没有完美的解决方案——因为多个事件共享一个协程，取消其中一个必然影响整批。不设置任务名至少避免了错误地取消其他会话的任务。

---

## 总结：理解取消路径的核心框架

1. **失效 vs 信号是两个独立维度**：`clean_events()` 使事件失效（L1+L2），`cancel_tasks()` 发送取消信号（L3），两者可以组合但不是必须的。

2. **用户主动取消 = 信号 + 部分失效**：`/cancel` 路由组合了 `cancel_tasks()`（信号）和 `remove_from_queue()`（L1），但遗漏了 `alive = False`（L2），对执行中事件的中断完全依赖 CancelledError。

3. **客户端断开 = 完全失效但无信号**：`clean_events()` 覆盖了 L1+L2，但不发送 CancelledError。执行中的任务继续运行直到自然结束，只是输出被 `alive` 守卫截断。

4. **迭代器释放是所有路径的最终保障**：`process_events()` 的 `finally` 块确保无论何种取消方式，迭代器都会被清理。`/cancel` 的立即清理只是优化了生成器函数的响应速度。

5. **同步函数是所有路径的盲区**：无论哪条路径，都无法立即中断正在 CPU 上执行的同步函数。唯一的区别是路径一通过 CancelledError 延迟中断，路径四通过 alive 截断输出但任务继续运行。
