# 批处理推理请求合并机制

本文档梳理 Gradio 中批处理（batch）推理的完整请求合并与结果拆分流程，涉及 batch 参数、队列调度、数据合并与结果分发四个核心环节。

## 一、batch 参数定义与配置入口

### 1.1 BlockFunction 中的核心字段

批处理的元信息存储在 [BlockFunction](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/block_function.py#L22-L175) 类中，两个关键字段：

- `batch: bool = False` — 是否启用批处理模式。启用后，函数应接收每个参数的输入列表，并返回输出列表的元组。
- `max_batch_size: int = 4` — 单次批处理的最大样本数，队列合并时不会超过这个数量。

### 1.2 用户配置入口

用户通过事件监听方法（如 `.click()`、`.submit()`）传入 batch 参数，最终在 [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/blocks.py#L665-L839) 的 `set_event_trigger` 方法中构建 BlockFunction：

```python
def set_event_trigger(
    self,
    ...
    batch: bool = False,
    max_batch_size: int = 4,
    ...
):
```

### 1.3 配置下发前端

BlockFunction 的 `get_config()` 方法会将 `batch` 和 `max_batch_size` 暴露给前端，前端据此进行 API 调用。

---

## 二、队列调度：请求如何被合并

### 2.1 事件入队

请求到达后，在 [Queue.push()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L279-L468) 中创建 `Event` 对象并入队。关键注意点：

- 如果 `fn.batch` 为 True，入队时 `body.event_id` 被设为 `None`（第 389 行），因为批处理模式下多个事件共享一次函数调用，单个 event_id 没有意义。

```python
body.event_id = event._id if not fn.batch else None
```

### 2.2 队列调度循环

`start_processing()` 是队列的主循环，不断从队列中取事件并执行。核心取事件逻辑在 [get_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L496-L519) 方法中：

```python
def get_events(self) -> tuple[list[Event], bool, str] | None:
    # 随机遍历各个 concurrency_id 对应的队列
    for concurrency_id in concurrency_ids:
        event_queue = self.event_queue_per_concurrency_id[concurrency_id]
        if 队列非空且并发未达上限:
            first_event = event_queue.queue[0]
            block_fn = first_event.fn
            events = [first_event]
            batch = block_fn.batch

            if batch:
                # 合并：从队列中取出同函数的事件，最多 max_batch_size 个
                events += [
                    event for event in event_queue.queue[1:]
                    if event.fn == first_event.fn
                ][: block_fn.max_batch_size - 1]

            for event in events:
                event_queue.queue.remove(event)

            return events, batch, concurrency_id
```

**合并规则总结：**
- 只有同一 `concurrency_id` 队列、且同一 `fn`（同一个 BlockFunction）的事件才会被合并
- 合并数量不超过 `max_batch_size`
- 队列中顺序在前的优先被合并
- 非批处理模式下，一次只取 1 个事件

---

## 三、数据合并：多个请求输入如何打包

### 3.1 process_events 中的合并

取出一批事件后，在 [process_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L787-L1081) 中进行数据合并（第 819-827 行）：

```python
if batch:
    body.data = list(
        zip(
            *[event.data.data for event in events if event.data],
            strict=False,
        )
    )
    body.request = events[0].request
    body.batched = True
```

**合并过程图示（假设 3 个请求，每个请求 2 个输入参数）：**

```
事件1: data = [inp1_a, inp2_a]
事件2: data = [inp1_b, inp2_b]
事件3: data = [inp1_c, inp2_c]

zip(*[event.data.data for event in events]) 后:
body.data = [
    [inp1_a, inp1_b, inp1_c],   # 第1个参数的批
    [inp2_a, inp2_b, inp2_c],   # 第2个参数的批
]
```

即：**按参数维度合并**，每个参数变成一个列表，列表长度等于批大小。

### 3.2 两条调用路径：队列合批 vs 直接调用

`call_process_api()` 有两条完全不同的调用路径，**只有队列路径才会发生跨请求合并**：

#### 路径 A：队列路径（跨请求合批）
```
用户请求 A ──┐
用户请求 B ──┼─> Queue.push() ──> 入队 ──> get_events() 合并 ──> process_events()
用户请求 C ──┘
```
- 入口：`/queue/join` 接口
- 在 [Queue.get_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L496-L519) 中从队列取出多个同 `fn` 的 Event 合并
- 在 [process_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L819-L827) 中设置 `body.batched = True`
- 真正的**跨请求合并**发生在这里，多个用户的请求被打包成一批

#### 路径 B：直接调用（单次输入封装）
```
用户请求 A ──> /call/{api_name} ──> call_process_api()
```
- 入口：`/call/{api_name}` 接口（直接 API 调用）
- 没有入队，没有 `get_events()` 合并过程
- `body.batched` 保持默认值 `False`（定义在 [PredictBody](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/data_classes.py#L98-L100)）

#### batch_in_single_out 适配层

在 [route_utils.call_process_api()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/route_utils.py#L362-L417) 中，`batch_in_single_out` 专门处理路径 B：

```python
# 触发条件：函数是 batch 模式，但请求不是来自队列
batch_in_single_out = not body.batched and fn.batch

if batch_in_single_out:
    inputs = [inputs]     # 单次输入包装成批次（加一层列表）
...
if batch_in_single_out:
    output["data"] = output["data"][0]   # 批结果拆回单个
```

**关键区别：**
| 维度 | 队列路径 | 直接调用路径 |
|------|---------|-------------|
| 跨请求合并 | ✅ 多个用户请求合并成一批 | ❌ 只有单个请求，只是格式适配 |
| `body.batched` | `True` | `False` |
| 合并位置 | `get_events()` + `process_events()` | `call_process_api()` 内部适配 |
| 批大小 | 1 ~ `max_batch_size` | 恒为 1 |

这个适配层的意义是：**无论是否走队列，batch 模式的函数签名保持一致**，用户不需要写两套逻辑。

### 3.3 process_api 中的预处理合并

在 [blocks.py 的 process_api()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/blocks.py#L2174-L2352) 中，batch 模式有独立的处理分支（第 2220-2261 行）：

```python
if batch:
    # 1. 校验：所有输入长度一致，且不超过 max_batch_size
    batch_sizes = [len(inp) for inp in inputs]
    # 2. 逐样本预处理，再转回批的维度
    inputs = [
        await self.preprocess_data(block_fn, list(i), state, explicit_call)
        for i in zip(*inputs, strict=False)
    ]
    # 3. 调用函数（传入的是批维度的输入）
    result = await self.call_function(block_fn, list(zip(*inputs, strict=False)), ...)
    preds = result["prediction"]
    # 4. 逐样本后处理，再转回批的维度
    data = [
        await self.postprocess_data(block_fn, list(o), state)
        for o in zip(*preds, strict=False)
    ]
    data = list(zip(*data, strict=False))
```

**关键：** 预处理和后处理都是**逐样本**进行的，因为每个组件的 preprocess/postprocess 只处理单个值。处理完后再通过 `zip(*)` 转回批维度。

---

## 四、结果拆分：批量输出如何分发给各请求

### 4.1 非生成式场景（一次性返回）

在 `process_events()` 的第 1008-1031 行，处理非流式、非生成的结果：

```python
elif response:
    output = copy.deepcopy(response)
    for e, event in enumerate(awake_events):
        if batch and "data" in output:
            # 拆分：按批的第 e 个元素取出
            output["data"] = list(zip(*response.get("data"), strict=False))[e]
        ...
        self.send_message(event, ProcessCompletedMessage(output=output, ...))
```

**拆分过程图示（假设 3 个请求，每个请求 2 个输出）：**

```
批结果: response["data"] = [
    [out1_a, out1_b, out1_c],   # 第1个输出的批
    [out2_a, out2_b, out2_c],   # 第2个输出的批
]

zip(*response.get("data")) 后:
[
    [out1_a, out2_a],  # 事件a的结果
    [out1_b, out2_b],  # 事件b的结果
    [out1_c, out2_c],  # 事件c的结果
]

然后取第 e 个，就是单个请求的结果。
```

### 4.2 stream 连接模式的批处理

对于 `connection = "stream"` 的事件（如实时音频流、视频流），批处理模式有特殊的同步逻辑：

```python
if awake_events[0].streaming:  # streaming 属性判断 connection == "stream"
    awake_events, closed_events = await Queue.wait_for_batch(
        awake_events,
        [cast(float, fn.time_limit or 30) - first_iteration] * len(awake_events),
    )
```

**stream 连接模式的批处理流程：**
- 每次迭代先调用 `wait_for_batch()`，等待批内所有事件都收到下一次输入
- 超时的事件会被标记为 `closed_events`，单独发送完成消息
- 每次生成迭代，所有存活的批内事件都会收到相同的 `ProcessGeneratingMessage`
- 流结束后，最终结果同样按 `zip(*outputs)[e]` 方式拆分

**重要：stream 连接 ≠ 生成器函数**，详见第七章边界条件。

### 4.3 每个事件独立发送消息

拆分后，每个事件通过 `send_message()` 独立发送到自己的 session 队列，前端通过 SSE 或 WebSocket 接收各自的结果。

---

## 五、完整流程示意图

```
用户请求 A      用户请求 B      用户请求 C
    │              │              │
    ▼              ▼              ▼
  Event A        Event B        Event C
    │              │              │
    └──────────────┼──────────────┘
                   ▼
         ┌───────────────────┐
         │  队列 (per concurrency_id)
         └───────────────────┘
                   │
      get_events() 取出同 fn 的一批
                   │
                   ▼
         zip(*inputs) 按参数合并
                   │
                   ▼
         call_process_api → process_api
                   │
      逐样本 preprocess → 函数调用 → 逐样本 postprocess
                   │
                   ▼
         zip(*outputs) 按样本拆分
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
  事件 A         事件 B         事件 C
    │              │              │
send_message   send_message   send_message
    │              │              │
    ▼              ▼              ▼
前端 A          前端 B          前端 C
```

---

## 六、关键代码位置速查

| 环节 | 文件 | 关键方法 / 行号 |
|------|------|----------------|
| 参数定义 | [block_function.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/block_function.py#L22-L76) | `BlockFunction.__init__` |
| 事件入队 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L279-L468) | `Queue.push()` |
| 队列取批 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L496-L519) | `Queue.get_events()` |
| 输入合并 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L819-L827) | `process_events()` L819-L827 |
| 结果拆分 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L1008-L1031) | `process_events()` L1008-L1031 |
| API 批处理 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/blocks.py#L2220-L2261) | `process_api()` batch 分支 |
| 单转批适配 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/route_utils.py#L377-L416) | `call_process_api()` |

---

## 七、关键边界条件详解

### 7.1 边界条件一：只有队列路径才会跨请求合并

这是最容易混淆的一点。`batch=True` 只是声明函数接收批量输入，但**是否真的合并多个用户的请求，完全取决于调用路径**。

#### 队列路径：真正的跨请求合并

当请求走 `/queue/join` 时，流程是：
1. 每个请求在 [Queue.push()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L279-L468) 中创建独立的 `Event` 对象并入队
2. 队列调度循环在 [get_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L496-L519) 中从队首开始，把同 `fn` 的 Event 打包，最多 `max_batch_size` 个
3. 在 [process_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L819-L827) 中设置 `body.batched = True`，并用 `zip(*inputs)` 合并数据

**关键判断标志：** `body.batched` 被显式设置为 `True`。

#### 直接调用路径：只是单次输入的格式包装

当请求走 `/call/{api_name}` 直接调用时：
1. 没有入队，没有 `get_events()` 合并
2. `body.batched` 保持默认值 `False`（定义在 [PredictBody](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/data_classes.py#L98-L100)）
3. 在 `call_process_api()` 中触发 `batch_in_single_out` 逻辑：
   - 输入：`[inp1, inp2]` → `[[inp1], [inp2]]` （加一层列表，假装是批大小为 1 的批次）
   - 输出：`[[out1], [out2]]` → `[out1, out2]` （去掉外层列表）

**代码佐证：**
```python
# route_utils.py L377-L379
batch_in_single_out = not body.batched and fn.batch
if batch_in_single_out:
    inputs = [inputs]     # 包装成批

# route_utils.py L415-L416
if batch_in_single_out:
    output["data"] = output["data"][0]   # 拆回单个
```

**实际效果对比：**

| 场景 | 调用路径 | 函数收到的输入 | 实际合并的请求数 |
|------|---------|---------------|----------------|
| 3 个用户同时请求，走队列 | 队列路径 | `[[a1, b1, c1], [a2, b2, c2]]` | 3 个跨请求 |
| 1 个用户直接调用 API | 直接调用 | `[[a1], [a2]]` | 1 个（无合并） |
| 1 个用户走队列，队列中无其他请求 | 队列路径 | `[[a1], [a2]]` | 1 个（批大小为 1） |

### 7.2 边界条件二：stream 连接 ≠ 生成器函数

这是另一个常见混淆点。两者是完全不同层面的概念，与 batch 模式的兼容性也不同。

#### stream 连接：通信协议层面的概念

`connection = "stream"` 是 [BlockFunction](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/block_function.py#L56) 的一个字段，描述**前后端之间的通信模式**：

```python
# BlockFunction.__init__ L56
connection: Literal["stream", "sse"] = "sse",
```

- **含义**：前后端保持长连接，前端持续发送数据块（如麦克风音频流），后端持续返回结果
- **判断**：`Event.streaming` 属性返回 `self.fn.connection == "stream"`
- **与 batch 兼容性**：✅ **完全兼容**
  - 在 [process_events()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L926-L935) 中有专门处理
  - 通过 `wait_for_batch()` 同步批内所有事件的输入就绪状态
  - 典型场景：多路麦克风音频流的实时语音识别批处理

#### 生成器函数：Python 语言层面的概念

`types_generator` 是 [BlockFunction](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/block_function.py#L96-L98) 通过内省判断的函数类型：

```python
# BlockFunction.__init__ L96-L98
self.types_generator = inspect.isgeneratorfunction(
    self.fn
) or inspect.isasyncgenfunction(self.fn)
```

- **含义**：函数内部使用 `yield` 逐次返回结果（如 LLM 逐字生成文本）
- **判断**：`inspect.isgeneratorfunction(fn)` 或 `inspect.isasyncgenfunction(fn)`
- **与 batch 兼容性**：❌ **完全不兼容**
  - 在 [process_api()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/blocks.py#L2225-L2228) 中直接报错：
  ```python
  if inspect.isasyncgenfunction(block_fn.fn) or inspect.isgeneratorfunction(block_fn.fn):
      raise ValueError("Gradio does not support generators in batch mode.")
  ```

#### 核心区别对比表

| 维度 | stream 连接 | 生成器函数 |
|------|------------|-----------|
| 概念层面 | 前后端通信协议 | Python 函数类型 |
| 代码判断 | `fn.connection == "stream"` | `inspect.isgeneratorfunction(fn)` |
| 多次返回方式 | 前后端长连接，多次调用函数 | 单次调用函数，内部 `yield` 多次 |
| 与 batch 兼容 | ✅ 兼容 | ❌ 不兼容（直接报错） |
| 函数签名 | `def fn(x: list) -> list` | `def fn(x): yield ...` |
| 典型场景 | 实时音频流批处理 | LLM 文本生成 |

#### 为什么生成器函数不能批处理？

技术上的核心矛盾：
- 批处理要求函数一次接收 N 个输入，一次返回 N 个输出（或 N 组输出）
- 生成器函数是一次调用，逐步 yield 结果，无法对齐多个请求的 yield 节奏
- 如果批内有 3 个请求，生成器函数的 yield 顺序无法对应到 3 个独立的输出流

而 stream 连接模式下，每次迭代都是**完整的函数调用**（接收一批输入，返回一批输出），只是前后端会反复调用这个函数，因此可以批处理。

---

## 八、常见误区澄清

1. ❌ **误区**：设置 `batch=True` 就会自动合并多个请求
   ✅ **事实**：只有走队列路径才会合并，直接调用只是格式适配

2. ❌ **误区**：`stream` 模式不能批处理
   ✅ **事实**：`connection="stream"` 完全支持批处理，不能批处理的是生成器函数

3. ❌ **误区**：`body.batched` 表示函数是 batch 模式
   ✅ **事实**：`body.batched` 表示**这个请求是否来自队列合并路径**，函数本身是否 batch 模式看 `fn.batch`

---

## 九、请求入口与处理流程详解

本章自顶向下梳理所有 HTTP 请求入口，明确排队接口和直连推理接口如何进入处理流程，以及单次请求包装逻辑的生效层级。

### 9.1 请求入口总览

Gradio 在 [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/routes.py) 中注册了多组推理相关的 HTTP 路由，按处理路径可以分为两大类：

**A. 直连推理接口（不走队列）**
- `POST /run/{api_name}`
- `POST /api/{api_name}`

**B. 排队接口（走队列）**
- `POST /queue/join`
- `POST /call/{api_name}`
- `POST /call/v2/{api_name}`
- `GET /queue/data`（取排队结果）
- `GET /call/{api_name}/{event_id}`（取排队结果）
- `GET /call/v2/{api_name}/{event_id}`（取排队结果）

注意：**`/call/` 系列接口虽然路径名像"直接调用"，但实际上它们内部也是走队列的**，这是最容易混淆的一点。

### 9.2 直连推理接口：`/run/` 和 `/api/`

这两个别名路由指向同一个 handler [predict()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/routes.py#L1273-L1316)，是唯一真正跳过队列、直接推理的入口。

#### 完整调用栈

```
POST /run/{api_name} 或 POST /api/{api_name}
    │
    ▼
routes.predict() ── L1273
    │
    ├── 1. 校验 api_open 开关  ── L1284
    │       if not api_open and fn.queue: 报错
    │
    ├── 2. 构造 PredictBodyInternal  ── L1279
    │       body = PredictBodyInternal(**body.model_dump(), request=request)
    │       注意：此处没有设置 batched，batched 保持默认 False
    │
    ├── 3. 编译 gr_request  ── L1289
    │       route_utils.compile_gr_request(body, fn, ...)
    │
    ├── 4. 直接调用 call_process_api()  ── L1301
    │       route_utils.call_process_api(app, body, gr_request, fn, ...)
    │       └── 内部可能触发 batch_in_single_out 包装
    │
    ▼
    返回 ORJSONResponse(output)
```

#### api_open 开关的作用

在 [routes.py L1284](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/routes.py#L1284) 有一个关键的门控：

```python
if not app.get_blocks().api_open and fn.queue:
    raise HTTPException(
        detail="This API endpoint does not accept direct HTTP POST requests. "
               "Please join the queue to use this API.",
        status_code=status.HTTP_404_NOT_FOUND,
    )
```

`api_open` 的默认值在 [blocks.py L1140](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/blocks.py#L1140)：

```python
self.api_open = utils.get_space() is None
```

- 非 Hugging Face Space 环境：`api_open = True`（允许直连）
- HF Space 环境：`api_open = False`（禁止直连，必须走队列）
- 可通过 `demo.queue(api_open=True/False)` 手动覆盖

**结论**：直连接口是有开关的，默认在 Space 中被禁用。

### 9.3 排队接口：`/queue/join`

这是最正统的队列入口，前端 SSE/WebSocket 交互使用的就是它。Handler 在 [queue_join()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/routes.py#L1357-L1368)。

#### 完整调用栈

```
POST /queue/join
    │
    ▼
routes.queue_join() ── L1357
    │
    ├── 校验 session_hash 不为空  ── L1363
    │
    ▼
routes.queue_join_helper() ── L1370
    │
    ├── 1. 设置 server_app  ── L1377
    │       blocks._queue.set_server_app(app)
    │
    ├── 2. 转换为 PredictBodyInternal  ── L1385
    │       body = PredictBodyInternal(**body.model_dump(), request=request)
    │       batched 仍为 False（尚未合并）
    │
    ├── 3. 入队  ── L1386
    │       success, event_id, state = await blocks._queue.push(body, ...)
    │       │
    │       └── queueing.py Queue.push()
    │           ├── 创建 Event 对象
    │           ├── 若 fn.batch=True：body.event_id = None  ── L389
    │           ├── 尝试缓存命中（命中则直接返回，不排队）
    │           └── event_queue.queue.append(event)  ── L458
    │
    ▼
    返回 {"event_id": event_id}
```

入队后，请求的处理就交给后台的队列调度循环 [start_processing()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L521-L563) 异步执行：

```
后台队列调度循环（start_processing）
    │
    ├── Queue.get_events() 取出一批事件  ── L534
    │   └── 若 batch=True，合并同 fn 的事件
    │
    ├── Queue.process_events() 执行  ── L545
    │   ├── 若 batch=True：
    │   │     body.data = zip(*[event.data.data for event in events])  ── L820
    │   │     body.batched = True  ── L827  ← 关键！此处才设为 True
    │   └── route_utils.call_process_api(...)
    │
    └── 结果通过 send_message() 发送到 session 的消息队列
```

客户端通过 `GET /queue/data?session_hash=xxx` 以 SSE 长连接方式消费结果。

### 9.4 排队接口：`/call/` 系列

`/call/{api_name}` 和 `/call/v2/{api_name}` 虽然路径看起来像"直接调用"，但**内部实际上也是走队列的**。它们在 [routes.py L1342-L1355](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/routes.py#L1342-L1355) 定义。

#### /call/{api_name}（v1）

```python
@router.post("/call/{api_name}")
async def simple_predict_post(api_name, body: SimplePredictBody, request, username):
    full_body = PredictBody(**body.model_dump(), simple_format=True)
    fn = route_utils.get_fn(blocks=app.get_blocks(), api_name=api_name, body=full_body)
    full_body.fn_index = fn._id
    return await queue_join_helper(full_body, request, username)  # ← 直接调 queue_join_helper
```

和 `/queue/join` 的区别：
- 入参是 [SimplePredictBody](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/data_classes.py#L56-L59)（只有 `data` 和 `session_hash`）
- 内部帮你查 `fn_index`
- 不需要前端传 `session_hash`（服务端会生成）
- 最终同样走 `queue_join_helper()` → `Queue.push()` 入队

#### /call/v2/{api_name}（v2）

```python
@router.post("/call/v2/{api_name}")
async def _(api_name, body: dict, request, username):
    parameters_info = app.api_info["named_endpoints"]["/" + api_name]["parameters"]
    processed_args = client_utils.construct_args(parameters_info, (), body)
    simple_body = SimplePredictBody(data=processed_args)
    full_body = PredictBody(**simple_body.model_dump(), simple_format=True)
    fn = route_utils.get_fn(...)
    full_body.fn_index = fn._id
    return await queue_join_helper(full_body, request, username)  # ← 也是走队列
```

和 v1 的区别：
- 入参是 `dict`（命名参数风格，如 `{"text": "hello", "image": "..."}`）
- 通过 `client_utils.construct_args()` 把命名参数转成位置参数列表
- 之后流程和 v1 完全一致

**结论**：`/call/` 系列 = 简化版 `/queue/join`，都是排队路径。

### 9.5 两类入口对比表

| 维度 | 直连接口 `/run/` `/api/` | 排队接口 `/queue/join` `/call/` |
|------|-------------------------|-------------------------------|
| 路由 handler | `predict()` L1273 | `queue_join()` L1357, `simple_predict_post()` L1344 |
| 是否调用 `queue_join_helper` | ❌ 不调用 | ✅ 调用 |
| 是否调用 `Queue.push()` | ❌ 不入队 | ✅ 入队 |
| 是否经过 `get_events()` 合并 | ❌ 不经过 | ✅ 经过 |
| `body.batched` 设置时机 | 保持默认 `False` | `process_events()` L827 设为 `True` |
| 返回方式 | 同步返回 JSON | 返回 event_id，异步取结果 |
| 受 api_open 控制 | ✅ 受控制 | ❌ 不受控 |
| 支持跨请求合批 | ❌ 不支持 | ✅ 支持 |

### 9.6 单次请求包装逻辑的生效层级

`batch_in_single_out`（单次请求包装成批次）是本章最关心的逻辑，现在精确定位它的生效位置。

#### 生效层级：`route_utils.call_process_api()` 内部

代码在 [route_utils.py L377-L416](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/route_utils.py#L377-L416)：

```python
async def call_process_api(app, body, gr_request, fn, root_path):
    ...
    inputs = body.data

    # ┌──────────────────────────────────────────────────────┐
    # │ 包装逻辑：在 process_api 调用之前，进入之后          │
    # └──────────────────────────────────────────────────────┘
    batch_in_single_out = not body.batched and fn.batch  # 触发条件

    if batch_in_single_out:
        inputs = [inputs]     # L379：包装 ── [inp1, inp2] → [[inp1], [inp2]]

    ...
    output = await app.get_blocks().process_api(   # L386：进入核心处理
        block_fn=fn,
        inputs=inputs,   # 这里传入的已经是包装后的
        ...
    )
    ...

    # ┌──────────────────────────────────────────────────────┐
    # │ 拆包逻辑：在 process_api 返回之后                     │
    # └──────────────────────────────────────────────────────┘
    if batch_in_single_out:
        output["data"] = output["data"][0]  # L416：拆包 ── [[out1], [out2]] → [out1, out2]

    return output
```

#### 触发条件详解

```python
batch_in_single_out = not body.batched and fn.batch
```

两个条件同时满足才会触发：
1. `fn.batch = True`：函数本身声明为 batch 模式
2. `body.batched = False`：请求**不是**来自队列合并路径

关键代码在 [queueing.py L819-L827](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/queueing.py#L819-L827)：

```python
if batch:
    body.data = list(zip(*[event.data.data for event in events if event.data], strict=False))
    body.request = events[0].request
    body.batched = True   # 只要 fn.batch=True，就会执行这里
```

**`body.batched` 的值只和 `fn.batch` 有关，和批大小无关**。即使队列中只有 1 个请求，只要函数是 batch 模式，`body.batched` 也会被设为 `True`。

各场景下的触发情况：

| 场景 | `fn.batch` | `body.batched` | `batch_in_single_out` | 是否包装 |
|------|-----------|---------------|----------------------|---------|
| 直连 `/run/` 调用 batch 函数 | True | **False**（默认值，不经过 process_events） | **True** | ✅ 在 `call_process_api` 包装 |
| 队列 `/queue/join` 调用 batch 函数（无论批大小） | True | **True**（process_events L827 设置） | False | ❌ 已在 process_events 中合并，无需再包装 |
| 直连调用普通函数 | False | False | False | ❌ 不需要 |
| 队列调用普通函数 | False | False | False | ❌ 不需要 |

#### 在完整调用链中的位置

以直连接口为例：

```
POST /run/{api_name}
    │
    └── routes.predict()  [routes.py L1273]
            │
            │  body.batched = False （默认值，从未被修改）
            │
            └── route_utils.call_process_api()  [route_utils.py L362]
                    │
                    │  ┌─ batch_in_single_out = not False and True = True
                    │  │
                    │  ├─► inputs = [inputs]   ← 【在此处包装】
                    │  │
                    ├──┼──► blocks.process_api()  [blocks.py L2174]
                    │  │       正常 batch 分支处理
                    │  │
                    │  └─► output["data"] = output["data"][0]   ← 【在此处拆包】
                    │
                    ▼
              返回结果
```

以队列接口为例（批大小为 1，函数为 batch 模式）：

```
POST /queue/join
    │
    └── Queue.push() → Event 入队
            │
            └── 后台调度 get_events() → 取到 1 个 Event
                    │
                    └── process_events()  [queueing.py L787]
                            │
                            │  if batch: （fn.batch=True）
                            │      body.data = zip(*[event.data.data])
                            │      body.batched = True  ← 已在此处合并
                            │
                            └── call_process_api()
                                    │
                                    │  batch_in_single_out = not True and True = False
                                    │  → 不触发包装，直接透传
                                    │
                                    └── process_api()
```

#### 层级结论

`batch_in_single_out` 包装逻辑的精确位置和适用范围：

- **所在文件**：[route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/route_utils.py)
- **所在函数**：`call_process_api()` L377-L416
- **层级**：路由层和核心处理层之间的**适配层**
- **唯一触发场景**：**直连推理接口**（`/run/` 或 `/api/`）调用 batch 模式的函数
- **不触发场景**：所有走队列的请求，即使批大小为 1，因为 `process_events()` 已经把 `body.batched` 设为 `True`

这个设计的目的是：**让 batch 模式的函数对调用方透明**。无论请求来自直连还是队列，函数只需要实现一套批处理签名，适配层自动处理单请求的格式转换。

### 9.7 请求入口到包装逻辑的完整分层图

```
┌─────────────────────────────────────────────────────────────────┐
│                     HTTP 请求入口层                               │
│  ┌──────────────┐    ┌──────────────────────────────────────┐   │
│  │ /run/ /api/  │    │ /queue/join  /call/  /call/v2/      │   │
│  │  (直连推理)   │    │           (排队推理)                 │   │
│  └──────┬───────┘    └──────────────┬───────────────────────┘   │
│         │                           │                           │
│         ▼                           ▼                           │
│  routes.predict()          routes.queue_join_helper()           │
│         │                    └──► Queue.push() ──► Event 入队   │
│         │                           │                           │
└─────────┼───────────────────────────┼───────────────────────────┘
          │                           │
          │              ┌────────────┴────────────┐
          │              │   后台队列调度循环       │
          │              │  get_events() 取批      │
          │              │  process_events() 执行   │
          │              │  → 合并 inputs: zip(*)   │
          │              │  → body.batched = True   │
          │              └────────────┬────────────┘
          │                           │
          ▼                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              route_utils.call_process_api()  适配层              │
│                                                                   │
│   batch_in_single_out = not body.batched and fn.batch            │
│   (仅直连 batch 函数时为 True)                                    │
│                                                                   │
│   ┌─ if batch_in_single_out:                                     │
│   │     inputs = [inputs]          ← 包装为批                    │
│   │                                                               │
│   ├──► blocks.process_api()         (核心处理层)                 │
│   │                                                               │
│   └─ if batch_in_single_out:                                     │
│         output["data"] = output["data"][0]   ← 拆回单个          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

