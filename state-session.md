# Gradio State 组件的会话隔离机制

## 一、概述

`gr.State` 是 Gradio 中一个特殊的隐藏组件，用于在同一用户的页面会话中跨多次交互持久化数据。它既不同于所有用户共享的全局变量，也不同于浏览器端 `localStorage` 持久化的 `BrowserState`。其核心特征是：**数据在同一浏览器会话内持久存在，但不同用户之间完全隔离，页面刷新即重置**。

理解 State 的会话隔离需要追踪三个关键环节：
1. **状态注入** — 服务端如何将 State 值传给用户函数
2. **会话隔离** — 不同用户的 State 如何互不干扰
3. **持久生命周期** — State 从创建到销毁的完整过程

---

## 二、State 组件的定义与特殊性

State 组件定义在 [state.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/components/state.py#L21-L102)，它继承自 `Component`，但有几个关键的覆盖行为：

### 2.1 `stateful = True`

```python
@property
def stateful(self) -> bool:
    return True
```

这是 State 区别于所有其他组件的核心标志。普通组件（如 Textbox、Image）的 `stateful` 默认返回 `False`（见 [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L174-L175)），而 State 组件覆写为 `True`。这个属性在整个数据流中被反复检查，决定了预处理、后处理和 API 暴露的不同行为。

### 2.2 `skip_api = True`

```python
@property
def skip_api(self):
    return True
```

State 组件不出现在 API 文档中，也不会在 API 端点的参数/返回值中暴露。这是因为 State 的值由服务端自动管理，客户端无需（也不应）手动传递。

### 2.3 预处理与后处理均为透传

```python
def preprocess(self, payload: Any) -> Any:
    return payload

def postprocess(self, value: Any) -> Any:
    return value
```

State 不做任何数据转换，值原样进出。这与 Textbox 等组件会做字符串解析/格式化不同——State 的值可以是任意 Python 对象（只要可 deepcopy）。

### 2.4 初始值必须可 deepcopy

```python
try:
    value = deepcopy(value)
except TypeError as err:
    raise TypeError(
        f"The initial value of `gr.State` must be able to be deepcopied. ..."
    ) from err
```

deepcopy 的要求是为了会话隔离：每个新会话必须获得一份独立的初始值副本，而不是共享同一个对象引用。如果无法 deepcopy（如包含文件句柄、数据库连接等），Gradio 会直接报错。

### 2.5 TTL 与删除回调

```python
self.time_to_live = math.inf if time_to_live is None else time_to_live
self.delete_callback = delete_callback or default_delete_callback
```

- `time_to_live`：State 的存活时间（秒），默认无限期
- `delete_callback`：State 被删除时的回调函数，可用于资源清理

---

## 三、状态注入机制

状态注入是指服务端如何将 State 的当前值"注入"到用户函数的参数中，以及如何将函数返回值"写回"State。核心逻辑在 [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L1820-L1896) 的 `preprocess_data` 方法中。

### 3.1 输入端：从 SessionState 读取

当事件触发时，`preprocess_data` 遍历 `block_fn.inputs` 中的所有组件：

```python
for i, block in enumerate(block_fn.inputs):
    if block.stateful:
        processed_input.append(state[block._id])   # 直接从会话状态读取
    else:
        # 普通组件：从前端传来的 data 中取值，做预处理
        ...
```

**关键区别**：
- **普通组件**：值来自前端请求的 `inputs[i]`，经过文件缓存、数据模型校验、`preprocess()` 转换
- **State 组件**：值直接从 `state[block._id]` 读取，跳过所有前端数据流，不做任何预处理

这意味着 State 的值完全存储在服务端内存中，前端永远不持有也不传递 State 的实际值。

### 3.2 输出端：写回 SessionState

在后处理阶段 ([blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L1984-L1992))：

```python
if block.stateful:
    prediction_value = predictions[i]
    if utils.is_prop_update(prediction_value):
        if "value" in prediction_value:
            state[block._id] = prediction_value["value"]
    else:
        state[block._id] = prediction_value
    output.append(None)   # State 不向前端返回任何数据
```

- 函数返回的 State 值被写入 `state[block._id]`
- 前端收到的是 `None`（不会在 UI 上显示任何内容）
- 支持 `gr.update(value=...)` 语法更新 State

### 3.3 变更追踪

[get_state_ids_to_track](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L2354-L2369) 方法会在函数调用前后对比 State 值的哈希：

```python
for block in block_fn.outputs:
    if block.stateful and any(
        (block._id, "change") in fn.targets
        for fn in state.blocks_config.fns.values()
    ):
        value = state[block._id]
        state_ids_to_track.append(block._id)
        hashed_values.append(utils.deep_hash(value))
```

如果 State 值发生了变化（哈希不同），会触发该 State 上的 `.change` 事件监听器。前端 [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/js/state/Index.svelte) 监听 `change` 事件，但 State 本身没有 UI 渲染，所以 `change` 事件主要用于链式触发其他组件的更新。

### 3.4 完整数据流示例

以 [hangman demo](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/demo/hangman/run.py) 为例：

```python
used_letters_var = gr.State([])

def guess_letter(letter, used_letters):
    used_letters.append(letter)
    return {
        used_letters_var: used_letters,      # 写回 State
        used_letters_box: ", ".join(used_letters),
        hangman: answer
    }

btn.click(
    guess_letter,
    [input_letter, used_letters_var],        # State 作为输入
    [used_letters_var, used_letters_box, hangman]  # State 作为输出
)
```

1. 用户点击按钮 → 前端发送请求（不含 State 值）
2. `preprocess_data`：`input_letter` 从前端数据获取，`used_letters` 从 `session_state[used_letters_var._id]` 获取
3. 函数执行后返回 `used_letters`
4. `postprocess_data`：`used_letters_var` 的值写入 `session_state`，前端收到 `None`
5. 下一次请求时，`session_state` 中已经是更新后的值

---

## 四、会话隔离机制

会话隔离的核心问题是：**多个用户同时使用同一个 Gradio 应用时，如何确保每个用户的 State 互不干扰？**

### 4.1 session_hash：会话的唯一标识

每个浏览器会话在连接时生成一个唯一的 `session_hash`：

- **前端 JS 客户端** ([client.ts](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/js/src/client.ts#L54))：
  ```typescript
  session_hash: string = Math.random().toString(36).substring(2);
  ```
  每次页面加载时生成一个随机字符串，作为该会话的标识。

- **Python 客户端** ([client.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/python/gradio_client/client.py#L191))：
  ```python
  self.session_hash = str(uuid.uuid4())
  ```

所有后续请求都携带这个 `session_hash`，服务端据此查找对应的会话状态。

### 4.2 StateHolder：全局会话容器

[StateHolder](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L16-L61) 是一个全局容器，维护所有活跃会话的状态：

```python
class StateHolder:
    def __init__(self):
        self.capacity = 10000
        self.session_data: OrderedDict[str, SessionState] = OrderedDict()
        self.time_last_used: dict[str, datetime.datetime] = {}
        self.lock = threading.Lock()
```

- `session_data`：以 `session_hash` 为键，`SessionState` 为值的有序字典
- `capacity`：最大会话数（默认 10000，可通过 `launch(state_session_capacity=N)` 调整）
- LRU 淘汰：当会话数超过容量时，最久未使用的会话被淘汰

`StateHolder` 在 [App](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/routes.py#L237) 初始化时创建：

```python
class App(FastAPI):
    def __init__(self, ...):
        self.state_holder = StateHolder()
```

并在 `configure_app` 时关联到 Blocks：

```python
def configure_app(self, blocks: gradio.Blocks) -> None:
    ...
    self.state_holder.set_blocks(blocks)
```

### 4.3 SessionState：单会话的隔离状态

[SessionState](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L64-L161) 是每个会话的独立状态容器：

```python
class SessionState:
    def __init__(self, blocks: Blocks):
        self.blocks_config = copy(blocks.default_config)
        self.config_values = {
            k: self.blocks_config.config_for_block(k, [], v)
            for k, v in self.blocks_config.blocks.items()
            if k in blocks.blocks
        }
        self.state_data: dict[int, Any] = {}
        self._state_ttl = {}
        self.is_closed = False
```

关键设计：

- **`blocks_config = copy(blocks.default_config)`**：每个会话持有 Blocks 配置的一个浅拷贝。这意味着组件定义（结构、事件绑定）共享，但组件实例可以独立修改。
- **`state_data: dict[int, Any]`**：以组件 `_id` 为键存储 State 值。这是隔离的核心——每个会话有独立的 `state_data` 字典。
- **`_state_ttl`**：记录每个 State 的存活时间和创建时间。

### 4.4 状态读取时的隔离保证

[SessionState.__getitem__](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L83-L90)：

```python
def __getitem__(self, key: int) -> Any:
    block = self.blocks_config.blocks[key]
    if block.stateful:
        if key not in self.state_data:
            self.state_data[key] = deepcopy(getattr(block, "value", None))
        return self.state_data[key]
    else:
        return block
```

**隔离的关键点**：
1. 首次访问某个 State 时，从组件定义的 `value` 做一次 `deepcopy` 存入 `state_data`
2. 后续访问直接返回 `state_data` 中的值
3. 不同会话的 `state_data` 互不共享，修改只影响当前会话

`deepcopy` 保证了每个会话拿到的是独立的对象副本，而不是对同一对象的引用。这就是为什么 State 初始值必须可 deepcopy——否则隔离就无法保证。

### 4.5 请求处理中的会话查找

当请求到达时，服务端通过 `session_hash` 查找对应的 `SessionState`：

在 [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/route_utils.py#L320-L341) 的 `restore_session_state`：

```python
def restore_session_state(app: App, body: PredictBodyInternal):
    session_hash = getattr(body, "session_hash", None)
    if session_hash is not None:
        session_state = app.state_holder[session_hash]
        ...
    else:
        session_state = SessionState(app.get_blocks())
        ...
    return session_state, iterator
```

[StateHolder.__getitem__](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L28-L33) 会在会话不存在时自动创建：

```python
def __getitem__(self, session_id: str) -> SessionState:
    if session_id not in self.session_data:
        self.session_data[session_id] = SessionState(self.blocks)
    self.update(session_id)
    self.time_last_used[session_id] = datetime.datetime.now()
    return self.session_data[session_id]
```

### 4.6 客户端的自动 State 管理

Python 客户端对 State 做了透明处理，用户无需手动管理：

1. **API 信息中隐藏 State**：`skip_api = True` 导致 State 不出现在 API 参数/返回值中
2. **自动插入空 State**：[insert_empty_state](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/python/gradio_client/client.py#L1224-L1229) 在调用参数中为 State 位置插入 `None`
3. **自动剥离 State 输出**：[remove_skipped_components](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/python/gradio_client/client.py#L1263-L1269) 从返回值中过滤掉 `skip=True` 的组件
4. **重置会话**：[reset_session](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/python/gradio_client/client.py#L749-L751) 生成新的 `session_hash`，等同于刷新页面

---

## 五、持久生命周期管理

State 的生命周期管理有两条独立的清理路径：**TTL 过期清理**和**会话容量淘汰**。它们操作粒度不同、触发条件不同、清理逻辑也不同，容易混淆。此外，TTL 的登记时机有一个容易忽略的细节：只读不写的 State 不会被 TTL 管理。

### 5.1 两条清理路径对比

| | TTL 过期清理 | 会话容量淘汰（LRU） |
|---|---|---|
| **操作粒度** | 单个 State key（`state_data` 中的某一项） | 整个 `SessionState`（含该会话全部 State） |
| **触发条件** | State 写入后经过 `time_to_live` 秒（会话关闭后切换为 1 小时） | 活跃会话数超过 `state_session_capacity` |
| **触发位置** | [StateHolder.delete_state](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L51-L61) | [StateHolder.update](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L38-L43) |
| **是否调 delete_callback** | ✅ 调用 `component.delete_callback(value)` | ❌ 不调用，整个 SessionState 直接丢弃 |
| **运行频率** | 每秒扫描一次（`_delete_state` 后台任务） | 每次请求时即时检查 |
| **涉及 `_state_ttl`** | ✅ 依赖 `_state_ttl` 判断是否过期 | ❌ 无关 |

### 5.2 TTL 登记时机：只有写回时才登记

`_state_ttl` 的登记**只发生在 `SessionState.__setitem__` 中**：

```python
# SessionState.__setitem__（state_holder.py L92-L101）
def __setitem__(self, key: int, value: Any):
    from gradio.components import State
    block = self.blocks_config.blocks.get(key)
    if isinstance(block, State):
        self._state_ttl[key] = (          # ← 登记存活时间和当前时刻
            block.time_to_live,
            datetime.datetime.now(),
        )
        self.state_data[key] = value
    else:
        self.blocks_config.blocks[key] = value
    ...
```

而 `__getitem__`（读取 State 值时）只做惰性初始化，**不登记 TTL**：

```python
# SessionState.__getitem__（state_holder.py L83-L90）
def __getitem__(self, key: int) -> Any:
    block = self.blocks_config.blocks[key]
    if block.stateful:
        if key not in self.state_data:
            self.state_data[key] = deepcopy(getattr(block, "value", None))  # 仅写入 state_data
        return self.state_data[key]                                         # 不碰 _state_ttl
    else:
        return block
```

这意味着：

**如果 State 只作为函数输入被读取，但从未作为函数输出被写回，那么 `_state_ttl` 中不会有该 State 的记录。**

具体场景：假设一个 State 只出现在 `inputs` 中而从未出现在 `outputs` 中：

```python
counter = gr.State(0)

def read_only_fn(count):
    return f"当前计数: {count}"    # 没有写回 counter

btn.click(read_only_fn, [counter], [textbox])   # counter 只在 inputs，不在 outputs
```

此时：
1. 第一次请求时，`preprocess_data` 调用 `state[counter._id]`，触发 `__getitem__`，`deepcopy(0)` 写入 `state_data`，但 `_state_ttl` 中没有登记
2. 函数执行后，`counter` 不在 `outputs` 中，`postprocess_data` 不会对它调用 `state[counter._id] = ...`
3. 结果：`state_data` 中有值，但 `_state_ttl` 中没有记录

### 5.3 未登记 TTL 的 State 在清理时的命运

TTL 过期清理的核心是 [state_components 属性](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L138-L161)：

```python
@property
def state_components(self) -> Iterator[tuple[State, Any, bool]]:
    for _id in self.state_data:
        ...
        block = self.blocks_config.blocks[_id]
        if isinstance(block, State) and _id in self._state_ttl:   # ← 关键：跳过未登记的
            time_to_live, created_at = self._state_ttl[_id]
            if self.is_closed:
                time_to_live = self.STATE_TTL_WHEN_CLOSED
            value = self.state_data[_id]
            yield (
                block, value,
                (datetime.datetime.now() - created_at).total_seconds() > time_to_live,
            )
```

**`_id in self._state_ttl` 这个条件**意味着：

- 未登记 TTL 的 State **不会被 yield 出来**
- `delete_all_expired_state` 遍历 `state_components`，不会看到这些 State
- 因此这些 State **永远不会被 TTL 清理路径删除**
- 它们只会在 **LRU 淘汰**时随整个 `SessionState` 一起被丢弃（但不会调 `delete_callback`）

这个设计是合理的：只读不写的 State 通常持有初始值（如配置常量），没有资源需要释放，不需要 TTL 管理。而 `delete_callback` 也是为需要显式清理的资源（如临时文件、数据库连接）设计的，这些资源只有在 State 被更新后才存在。

### 5.4 TTL 过期清理的完整流程

```
_delete_state（每秒触发）
  └─ StateHolder.delete_all_expired_state()
       └─ 遍历所有 session_id
            └─ StateHolder.delete_state(session_id, expired_only=True)
                 └─ 遍历 session_state.state_components（仅含已登记 TTL 的 State）
                      └─ 对每个 (component, value, expired):
                           ├─ 如果 expired_only=True 且 expired=False → 跳过
                           ├─ 如果 expired=True → 调用 component.delete_callback(value)
                           └─ 从 session_state.state_data 中删除该 key
```

注意：TTL 清理只删 `state_data` 中的个别 key，**不删除 `SessionState` 本身**。即使所有 State 都过期被清理了，空的 `SessionState` 仍然留在 `StateHolder.session_data` 中。

### 5.5 LRU 淘汰的完整流程

LRU 淘汰发生在每次请求访问 `StateHolder` 时：

```python
# StateHolder.__getitem__（state_holder.py L28-L33）
def __getitem__(self, session_id: str) -> SessionState:
    if session_id not in self.session_data:
        self.session_data[session_id] = SessionState(self.blocks)
    self.update(session_id)                                    # ← 触发 LRU 检查
    self.time_last_used[session_id] = datetime.datetime.now()
    return self.session_data[session_id]

# StateHolder.update（state_holder.py L38-L43）
def update(self, session_id: str):
    with self.lock:
        if session_id in self.session_data:
            self.session_data.move_to_end(session_id)          # 当前会话移到末尾
        if len(self.session_data) > self.capacity:
            self.session_data.popitem(last=False)              # 淘汰最旧的会话
```

关键点：
- `OrderedDict` 保证插入/访问顺序，`move_to_end` 把当前会话移到最新位置
- `popitem(last=False)` 弹出最久未访问的会话（FIFO 头部）
- **被淘汰的 `SessionState` 直接从 `session_data` 中移除，不调任何 `delete_callback`**
- 这意味着如果 State 设置了 `delete_callback` 用于资源清理，LRU 淘汰时该回调不会被调用——这是一个潜在的内存/资源泄漏点

### 5.6 心跳断开与会话关闭

[heartbeat 端点](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/routes.py#L1195-L1266) 的 SSE 连接断开时，会话被标记为关闭：

```python
# routes.py L1250-L1253
if session_hash in app.state_holder.session_data:
    app.state_holder.session_data[session_hash].is_closed = True
```

`is_closed = True` 不直接删除任何 State，但它改变了 TTL 过期判定中的 `time_to_live`：

```python
# state_components 属性中
if self.is_closed:
    time_to_live = self.STATE_TTL_WHEN_CLOSED   # 3600 秒（1 小时）
```

**会话关闭前后 TTL 行为的变化**：

| | 会话活跃（`is_closed=False`） | 会话关闭（`is_closed=True`） |
|---|---|---|
| `time_to_live=None`（默认） | `math.inf` → 永不过期 | 3600 秒 → 1 小时后过期 |
| `time_to_live=60` | 60 秒后过期 | 3600 秒后过期（覆盖用户设置） |
| 未登记 `_state_ttl` | 不参与 TTL 清理 | 仍然不参与 TTL 清理 |

关闭后覆盖为固定 1 小时的目的是给用户"重新打开标签页"的缓冲时间——如果用户在 1 小时内恢复连接（相同 `session_hash`），State 还在。

### 5.7 完整生命周期全景图

```
 ┌──────────────────────────────────────────────────────────────┐
 │                      用户打开页面                              │
 │  前端生成 session_hash，建立心跳 SSE 连接                       │
 └────────────────────────┬─────────────────────────────────────┘
                          │
 ┌────────────────────────▼─────────────────────────────────────┐
 │              首次请求到达                                      │
 │  StateHolder.__getitem__ → 创建 SessionState                  │
 │  StateHolder.update → LRU 检查（可能淘汰最旧会话）              │
 └────────────────────────┬─────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │                               │
 ┌────────▼─────────┐          ┌──────────▼──────────────────────┐
 │  State 作为输入    │          │  State 作为输出                   │
 │  __getitem__      │          │  __setitem__                     │
 │  → deepcopy 初始值│          │  → 登记 _state_ttl               │
 │    写入 state_data│          │  → 写入 state_data               │
 │  ⚠️ 不登记 TTL    │          │  ✅ 登记 TTL                     │
 └───────────────────┘          └─────────────────────────────────┘
                          │
           ┌──────────────┴──────────────────┐
           │                                 │
 ┌─────────▼──────────────┐    ┌─────────────▼─────────────────────┐
 │   TTL 过期清理路径       │    │   LRU 淘汰路径                     │
 │   （每秒扫描）           │    │   （每次请求即时触发）               │
 │                        │    │                                   │
 │   只处理 _state_ttl    │    │   不看 _state_ttl                  │
 │   中已登记的 State      │    │   直接弹掉整个 SessionState         │
 │                        │    │                                   │
 │   删除粒度：            │    │   删除粒度：                        │
 │   单个 state_data key  │    │   整个 SessionState 对象            │
 │                        │    │                                   │
 │   ✅ 调 delete_callback│    │   ❌ 不调 delete_callback          │
 │                        │    │                                   │
 │   会话仍存在于          │    │   会话从 session_data 中消失        │
 │   session_data 中      │    │                                   │
 └────────────────────────┘    └───────────────────────────────────┘
                          │
 ┌────────────────────────▼─────────────────────────────────────┐
 │              心跳 SSE 断开（关闭标签页/刷新）                    │
 │  is_closed = True                                             │
 │  所有已登记 TTL 的 State 的 time_to_live 被覆盖为 3600 秒      │
 │  未登记 TTL 的 State 仍不受 TTL 管理                           │
 └──────────────────────────────────────────────────────────────┘
```

### 5.8 一个完整的 TTL 生命周期示例

```python
my_state = gr.State(value=0, time_to_live=300, delete_callback=lambda v: print(f"清理: {v}"))
```

1. **用户 A 首次请求**：`preprocess_data` 调用 `state[my_state._id]` → `__getitem__` → `state_data[id] = 0`，**未登记 TTL**
2. **函数返回新值**：`postprocess_data` 调用 `state[my_state._id] = 42` → `__setitem__` → `_state_ttl[id] = (300, now)`，**登记 TTL**
3. **5 分钟内无写回**：`_delete_state` 每秒扫描，`state_components` 判定 `(now - created_at).seconds = 301 > 300`，`expired = True`
4. **清理**：`delete_state` 调用 `delete_callback(42)`，打印"清理: 42"，从 `state_data` 中删除该 key
5. **用户 A 再次请求**：`__getitem__` 发现 `state_data` 中已无该 key → `deepcopy(0)`（回到初始值），但 `_state_ttl` 仍保留旧记录

如果用户关闭页面：`is_closed = True`，TTL 覆盖为 3600 秒，1 小时后清理。

如果服务器会话数超过 10000：LRU 淘汰直接移除整个 `SessionState`，`delete_callback` 不会被调用。

---

## 六、与全局状态和浏览器状态的对比

| 特性 | 全局状态 | gr.State (会话状态) | BrowserState |
|------|---------|-------------------|-------------|
| 存储位置 | Python 进程内存 | Python 进程内存（按会话隔离） | 浏览器 localStorage |
| 跨用户共享 | ✅ 共享 | ❌ 隔离 | ❌ 隔离 |
| 页面刷新后保留 | ✅ | ❌ | ✅ |
| 数据类型 | 任意 Python 对象 | 任意可 deepcopy 的 Python 对象 | JSON 可序列化数据 |
| 传递方式 | 直接引用 | 服务端注入，不经前端 | 前端传递 |
| 内存开销 | 一份 | 每会话一份 | 无服务端开销 |

---

## 七、无法 deepcopy 的对象的替代方案

当对象无法 deepcopy 时（如数据库连接、模型实例），Gradio 官方建议使用 `gr.Request.session_hash` 手动实现会话隔离：

```python
user_objects = {}

def my_fn(request: gr.Request):
    session_hash = request.session_hash
    if session_hash not in user_objects:
        user_objects[session_hash] = ExpensiveObject()
    obj = user_objects[session_hash]
    ...
```

这种方案本质上是绕过 State 组件，在全局字典中手动按 `session_hash` 隔离，但需要自行管理生命周期和内存清理。

---

## 八、关键源码索引

| 功能 | 文件 | 关键行 |
|------|------|--------|
| State 组件定义 | [state.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/components/state.py#L21-L102) | L21-102 |
| stateful 属性 | [state.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/components/state.py#L60-L62) | L60-62 |
| skip_api 属性 | [state.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/components/state.py#L91-L93) | L91-93 |
| StateHolder 容器 | [state_holder.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L16-L61) | L16-61 |
| SessionState 隔离状态 | [state_holder.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/state_holder.py#L64-L161) | L64-161 |
| 状态注入（preprocess） | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L1820-L1896) | L1820-1896 |
| 状态写回（postprocess） | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L1943-L2085) | L1943-2085 |
| 变更追踪 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/blocks.py#L2354-L2369) | L2354-2369 |
| 心跳保活 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/routes.py#L1195-L1266) | L1195-1266 |
| 会话状态恢复 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/route_utils.py#L320-L341) | L320-341 |
| 过期状态清理 | [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/gradio/route_utils.py#L1027-L1038) | L1027-1038 |
| 前端 session_hash | [client.ts](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/js/src/client.ts#L54) | L54 |
| Python 客户端 reset | [client.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/python/gradio_client/client.py#L749-L751) | L749-751 |
| Python 客户端 State 插入 | [client.py](file:///d:/fz/0601/solo-dogfeeding/code/253-gradio/client/python/gradio_client/client.py#L1224-L1229) | L1224-1229 |
