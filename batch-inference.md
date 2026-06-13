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

### 3.2 route_utils 中的适配层

在 [route_utils.call_process_api()](file:///d:/fz/0601/solo-dogfeeding/code/256-gradio/gradio/route_utils.py#L362-L417) 中有一个 `batch_in_single_out` 模式：

```python
batch_in_single_out = not body.batched and fn.batch
if batch_in_single_out:
    inputs = [inputs]     # 单请求也包装成批
...
if batch_in_single_out:
    output["data"] = output["data"][0]   # 结果再拆出来
```

这个模式处理的是**非队列直接调用但函数是 batch 模式**的场景（比如直接 API 调用），保证函数签名一致。

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

### 4.2 生成式 / 流式场景

对于 `is_generating=True` 的生成式函数，批处理模式下的处理更复杂：

- 每次生成迭代，所有批内事件都会收到相同的 `ProcessGeneratingMessage`
- 流式事件通过 `wait_for_batch()` 等待所有事件的下一次输入就绪
- 生成结束后，最终结果同样按上述方式拆分

注意：**batch 模式不支持生成器函数**（process_api 第 2225-2228 行有校验直接报错）。这里的生成式指的是 `stream` 连接模式下的迭代调用。

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
