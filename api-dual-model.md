# Gradio 双形态应用构建：简洁接口与组合式界面的共享机制分析

## 1. 核心继承关系——Interface 本质上就是 Blocks

Gradio 的"双形态"并非两个独立体系，而是同一棵继承树上的两层抽象。关键代码在 [interface.py#L42](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L42)：

```python
class Interface(Blocks):
```

同样，[ChatInterface](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/chat_interface.py#L53) 和 [TabbedInterface](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L960) 也直接继承 `Blocks`：

```python
class ChatInterface(Blocks):
class TabbedInterface(Blocks):
```

这意味着 **所有高层接口的能力都来自 Blocks 的基础设施**——组件注册、事件绑定、配置序列化、队列调度、服务启动全部复用同一套底层逻辑。

---

## 2. 共享的组件模型

### 2.1 统一的组件注册机制

无论使用 `Interface` 还是 `Blocks`，组件都通过同一个 `Block.render()` 方法注册到 `BlocksConfig.blocks` 字典中。

注册入口在 [blocks.py#L197-L222](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/blocks.py#L197-L222)：

```python
def render(self):
    root_context = get_blocks_context()
    render_context = get_render_context()
    if render_context is not None:
        render_context.add(self)
        self.parent = render_context
    if root_context is not None:
        root_context.blocks[self._id] = self
        self.is_rendered = True
```

**Interface 的组件注册路径**：在 `Interface.__init__` 中通过 `with self:` 上下文管理器（继承自 `BlockContext.__enter__`），将 `input_components`、`output_components`、按钮、布局等逐一 `.render()` 注册到自身的 `BlocksConfig` 中。见 [interface.py#L479-L541](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L479-L541)。

**Blocks 的组件注册路径**：用户在 `with gr.Blocks() as demo:` 上下文中手动声明组件，组件自动调用 `Block.__init__` → `render()` 完成注册。

两条路径最终汇聚于同一个 `root_context.blocks[self._id] = self` 调用。

### 2.2 组件实例化桥接：`get_component_instance`

Interface 允许用字符串简写（如 `"image"`、`"label"`）代替组件实例，这个转换由 [components/\_\_init\_\_.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/components/__init__.py) 中的 `get_component_instance` 完成：

```python
self.main_input_components = [
    get_component_instance(i, unrender=True) for i in inputs
]
```

`unrender=True` 确保组件在实例化时不立即注册，而是在 Interface 内部的 `with self:` 布局中按需 `.render()`。这是 Interface 能够控制组件排列顺序的关键。

### 2.3 全局 ID 分配

所有组件共享同一个自增 ID 计数器 `Context.id`（[context.py#L18](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/context.py#L18)），无论来自 Interface 还是 Blocks 声明，确保组件 ID 全局唯一。

---

## 3. 共享的事件模型

### 3.1 事件注册的唯一入口：`BlocksConfig.set_event_trigger`

所有事件——无论是 Interface 的提交/清除/标记，还是 Blocks 的 `.click()`/`.change()`——最终都调用 [blocks.py#L639-L874](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/blocks.py#L639-L874) 中的 `BlocksConfig.set_event_trigger`。

该方法创建一个 `BlockFunction` 对象并存入 `BlocksConfig.fns` 字典，同时返回 `(block_fn, fn_id)` 元组。

### 3.2 Interface 的事件绑定方式

Interface 的事件绑定在 [interface.py#L679-L814](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L679-L814) 的 `attach_submit_events` 方法中，根据 `live` 和 `interface_type` 走不同分支：

| 条件 | 事件绑定方式 | 底层调用 |
|------|------------|---------|
| `live=True` + 输出型 | `_submit_btn.click(...)` | `set_event_trigger` |
| `live=True` + 标准型 | `on(events, fn, ...)` | `set_event_trigger` |
| `live=False` + 有 stop_btn | `on(triggers,...).then(fn,...).then(cleanup,...)` | 多次 `set_event_trigger` |
| `live=False` + 无 stop_btn | `on(triggers, fn, ...)` | `set_event_trigger` |

所有分支都通过组件的事件方法（`.click()`、`.stream()`、`.change()`）或 `gr.on()` 触发，与 Blocks 用户手写的代码完全一致。

### 3.3 EventListener：事件定义的统一抽象

[events.py#L538-L783](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/events.py#L538-L783) 中 `EventListener._setup` 是所有组件事件方法的工厂函数。每个 `EventListener` 实例通过 `__new__` 成为字符串（事件名），同时通过 `_setup` 生成 `event_trigger` 闭包，挂载到组件类上。

```python
class EventListener(str):
    def __new__(cls, event_name, *_args, **_kwargs):
        return super().__new__(cls, event_name)

    def __init__(self, event_name, ...):
        self.listener = self._setup(event_name, ...)
```

**ComponentMeta**（[component_meta.py#L199-L234](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/component_meta.py#L199-L234)）在类创建时自动将 `EVENTS` 列表中的每个 `EventListener` 转化为组件实例方法：

```python
class ComponentMeta(ABCMeta):
    def __new__(cls, name, bases, attrs):
        for event in events:
            trigger = (...).copy()
            attrs[event] = trigger.listener   # 例如 attrs["click"] = click_trigger
```

**BlocksMeta**（[blocks_events.py#L9-L22](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/blocks_events.py#L9-L22)）同理，将 `load` 事件注入 `Blocks` 类。

### 3.4 Dependency 链式调用

`Dependency` 对象（[events.py#L86-L157](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/events.py#L86-L157)）提供 `.then()`、`.success()`、`.failure()` 三个链式方法，每个方法内部创建新的 `EventListener` 并调用 `set_event_trigger`。Interface 的 `attach_submit_events` 也使用了 `.then()` 链：

```python
predict_event = on(triggers, ...).then(
    self.fn, self.input_components, self.output_components, ...
)
final_event = predict_event.then(cleanup, ...)
```

---

## 4. 上下文系统：两种形态的运行时桥梁

[context.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/context.py) 定义了全局和线程局部两层上下文：

| 上下文变量 | 作用 |
|-----------|------|
| `Context.root_block` | 当前根 Blocks 实例（全局） |
| `Context.id` | 全局自增组件 ID |
| `LocalContext.blocks` | 线程局部根 Blocks |
| `LocalContext.blocks_config` | 线程局部 BlocksConfig |
| `LocalContext.renderable` | 当前 @gr.render 上下文 |

`get_blocks_context()` 函数（[context.py#L50-L54](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/context.py#L50-L54)）根据是否在 `@gr.render` 中返回不同的 `BlocksConfig`，确保事件注册总是指向正确的配置对象。Interface 继承了这套机制，无需任何额外处理。

---

## 5. 边界不清的具体表现与根本原因

### 5.1 表现一：Interface 内部用了 Blocks 的 `with` 语法

[interface.py#L479-L541](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L479-L541) 中 Interface 的 UI 构建全部在 `with self:` 内完成：

```python
with self:
    self.render_title_description()
    with Row():
        # ...渲染组件
    _submit_event = self.attach_submit_events(_submit_btn, _stop_btn)
```

Interface 的构造函数实际上就是一个预制的 Blocks 定义脚本——用户手写在 `with gr.Blocks()` 里的代码被 Interface 内部自动生成了。

### 5.2 表现二：Interface 和 Blocks 可互嵌

`TabbedInterface`（[interface.py#L960-L1003](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L960-L1003)）接受 `Sequence[Blocks]` 作为参数，意味着 Interface 和 Blocks 实例可以混合放入标签页：

```python
TabbedInterface([interface_instance, blocks_instance], ["Tab A", "Tab B"])
```

在内部，它通过 `interface.render()` 将子 Blocks 的组件和事件合并到父 Blocks 的 `BlocksConfig` 中。

### 5.3 表现三：共享 `launch()` 和服务生命周期

`launch()` 方法定义在 `Blocks` 上，Interface 直接继承。两者启动时走完全相同的 FastAPI 路由注册、队列初始化、隧道创建流程。

### 5.4 根本原因

边界不清的本质是 **Interface 没有独立的状态模型**。它的 `input_components`、`output_components`、`fn` 等属性仅在构造阶段用于生成 Blocks 配置，构造完成后通过 `self.config = self.get_config_file()` 固化为 JSON 配置。运行时行为完全由 `Blocks` + `BlocksConfig` + `BlockFunction` 决定，Interface 自身的属性不再被引用。

---

## 6. 架构总结图

```
                     Block (基类：ID/渲染/配置)
                       │
               BlockContext (容器：children/enter/exit)
                       │
            ┌──────────┴──────────┐
            │                     │
        Blocks               Layout 类
   (BlocksMeta 元类)       (Row/Column/Tabs...)
            │
    ┌───────┼───────────┐
    │       │           │
Interface  ChatInterface  TabbedInterface
    │
    │  构造时通过 with self: 自动生成
    │  Row → Column → Component → Button
    │  → attach_submit_events → set_event_trigger
    │
    └──→ 最终产物：BlocksConfig { blocks, fns }
                              ↓
                     统一序列化为 config JSON
                              ↓
                    前端 Svelte 渲染 + 后端路由

事件流：
  EventListener._setup → event_trigger 闭包
      → 挂载到 Component/Blocks 实例方法 (.click/.change/.load)
      → 调用 root_block.set_event_trigger
      → 创建 BlockFunction → 存入 BlocksConfig.fns
      → 返回 Dependency (.then/.success/.failure 链)
```

---

## 7. 关键源文件索引

| 文件 | 核心职责 |
|------|---------|
| [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/blocks.py) | Block/BlockContext/Blocks/BlocksConfig 定义，set_event_trigger 入口 |
| [interface.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py) | Interface/TabbedInterface，自动布局与事件绑定 |
| [chat_interface.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/chat_interface.py) | ChatInterface，聊天场景的高层封装 |
| [events.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/events.py) | EventListener/Dependency/Events/on()，事件定义与链式调用 |
| [blocks_events.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/blocks_events.py) | BlocksMeta 元类，为 Blocks 注入 load 事件 |
| [component_meta.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/component_meta.py) | ComponentMeta 元类，为组件注入 EVENTS 方法 |
| [context.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/context.py) | Context/LocalContext，全局与线程局部渲染上下文 |
| [block_function.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/block_function.py) | BlockFunction，事件函数的运行时表示 |
| [renderable.py](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/renderable.py) | Renderable，@gr.render 动态渲染机制 |
| [data_classes.py#L160](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/data_classes.py#L160) | InterfaceTypes 枚举（STANDARD/INPUT_ONLY/OUTPUT_ONLY/UNIFIED） |

---

## 8. 子 Blocks 嵌入父容器的完整渲染流程

### 8.1 嵌入场景举例

```python
with gr.Blocks() as parent:
    with gr.Tab("Tab A"):
        child_interface.render()   # Interface 嵌入父 Blocks
    with gr.Tab("Tab B"):
        child_blocks.render()      # Blocks 嵌入父 Blocks
```

或 `TabbedInterface` 内部（[interface.py#L992-L1003](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L992-L1003)）：

```python
with self:
    with Tabs():
        for interface, tab_name in zip(interface_list, tab_names):
            with Tab(label=tab_name):
                interface.render()    # 每个子 Interface/Blocks 被 render
```

### 8.2 渲染前的状态：子 Blocks 已完成自构建

当 `child_interface.render()` 被调用时，子 Blocks 的构造函数已经执行完毕。此时子 Blocks 的 `default_config` 中已经包含了完整的 `blocks`（组件字典）和 `fns`（事件函数字典），布局树 `children` 也已构建好。

关键点：**子 Blocks 的所有组件 ID 和函数 ID 都是全局唯一的**，因为它们在构造时就通过 `Context.id` 自增分配。但它们的 `fns` 字典的 key（`_id`）从 0 开始编号，可能与父容器的 `fns` key 冲突。

### 8.3 `Blocks.render()` 的六步合并过程

[blocks.py#L1457-L1517](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/blocks.py#L1457-L1517) 是核心合并逻辑。当在父 Blocks 的 `with` 上下文中调用 `child.render()` 时，`root_context` 指向父的 `BlocksConfig`，`Context.root_block` 指向父 Blocks 实例。

**第一步：冲突检测**（L1460-L1470）

```python
if self._id in root_context.blocks:
    raise DuplicateBlockError(...)
overlapping_ids = set(root_context.blocks).intersection(self.blocks)
for id in overlapping_ids:
    if not isinstance(self.blocks[id], components.State):
        raise DuplicateBlockError(...)
```

子 Blocks 自身的 `_id`（作为 Block 实例）不能与父 blocks 中已有的 ID 冲突。同时，子 blocks 字典与父 blocks 字典之间只允许 `State` 组件有 ID 重叠（因为 State 可以跨 Blocks 共享）。

**第二步：合并 blocks 字典**（L1472-L1474）

```python
for block in self.blocks.values():
    block.page = Context.root_block.current_page
root_context.blocks.update(self.blocks)
```

将子 Blocks 的所有组件一次性 `update` 到父的 `blocks` 字典中。组件的 `_id` 不变（它们在构造时就是全局唯一的），但 `page` 属性被重写为父容器的当前页面。

**第三步：计算依赖偏移量**（L1475）

```python
dependency_offset = max(root_context.fns.keys(), default=-1) + 1
```

假设父 `fns` 中已有 key 0, 1, 2，则 `dependency_offset = 3`。子 `fns` 中所有函数的 `_id` 将加上这个偏移量，避免与父的 key 冲突。

**第四步：重编号与重映射依赖**（L1481-L1508）

这是最关键的步骤，逐个处理子 Blocks 的 `BlockFunction`：

```python
for dependency in self.fns.values():
    dependency.page = Context.root_block.current_page
    dependency._id += dependency_offset            # (a) 重编号
    for target in dependency.targets:              # (b) 重映射 target
        if target[0] == self._id:
            target = (Context.root_block._id, target[1])
    dependency.api_name = ...                      # (c) API 名称去重
    dependency.cancels = [c + dependency_offset for c in dependency.cancels]  # (d) cancels 偏移
    if dependency.trigger_after is not None:
        dependency.trigger_after += dependency_offset  # (e) trigger_after 偏移
    root_context.fns[dependency._id] = dependency  # (f) 写入父 fns
```

逐条解析：

- **(a) 重编号 `_id`**：子 Blocks 的 `fn._id` 从 0 开始编号（例如 0, 1, 2），加上偏移后变为 3, 4, 5，与父的 0, 1, 2 不冲突。

- **(b) 重映射 target**：`dependency.targets` 是 `list[tuple[int | None, str]]`，每个元组是 `(block_id, event_name)`。当 `target[0] == self._id` 时，说明这个事件是由子 Blocks 自身触发的（最典型的就是 `Blocks.load()` 事件——它的 target 是 Blocks 本身的 `_id`）。嵌入后，这个事件应该由根 Blocks 触发，所以将 `block_id` 替换为 `Context.root_block._id`。

  > ⚠️ 注意一个潜在的代码问题：`for target in dependency.targets` 遍历的是 tuple，`target = (Context.root_block._id, target[1])` 只是重新绑定了局部变量 `target`，并未修改 `dependency.targets` 列表中的元素。这意味着 target 的实际重映射可能没有生效——除非 targets 列表中的 tuple 在 `set_event_trigger` 阶段就是通过 `EventListenerMethod` 构造的且后续由 `get_config()` 直接序列化。实际上，`BlockFunction.get_config()`（[block_function.py#L138-L175](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/block_function.py#L138-L175)）直接序列化 `self.targets`，所以如果这里没有原地修改，load 事件的 target 仍指向子 Blocks 的 `_id`。但从前端的角度看，只要子 Blocks 的 `_id` 在 `root_context.blocks` 中存在（已通过 `update` 合并），前端仍能找到对应的组件来绑定事件。

- **(c) API 名称去重**：如果子的 API 名称与父的重复，自动追加后缀。例如两个子 Interface 都有 `/predict`，会变成 `/predict` 和 `/predict_1`。

- **(d) cancels 偏移**：`cancels` 存储的是要取消的函数 `_id` 列表，需要同样加上偏移量以指向合并后的正确函数。

- **(e) trigger_after 偏移**：链式事件（`.then()`/`.success()`/`.failure()`）通过 `trigger_after` 指向前一个函数的 `_id`，偏移后指向合并后的正确位置。

- **(f) 写入父 fns**：用偏移后的 `_id` 作为 key，将依赖写入父的 `fns` 字典。

**第五步：更新父 fn_id 计数器**（L1509）

```python
root_context.fn_id = max(root_context.fns.keys(), default=-1) + 1
```

**第六步：合并布局树**（L1514-L1516）

```python
render_context = get_render_context()
if render_context is not None:
    render_context.children.extend(self.children)
```

将子 Blocks 的顶层 children（即布局树的根节点）追加到当前渲染上下文的 children 中。这使得子 Blocks 的布局被嵌入到父的布局树中正确的位置（例如某个 Tab 下）。

### 8.4 合并后的数据结构

合并完成后，父 Blocks 的 `BlocksConfig` 变为：

```
blocks: {
    0: parent_Textbox,
    1: parent_Button,
    2: parent_Output,
    # 子 Blocks 的组件（_id 继续自增，因为 Context.id 全局唯一）
    5: child_Textbox,
    6: child_Button,
    7: child_Output,
    ...
}
fns: {
    0: parent_click_fn,       # 父的原始函数
    1: parent_change_fn,      # 父的原始函数
    # 子的函数（_id 偏移后）
    2: child_submit_fn,       # 原 _id=0, 偏移 +2 → 2
    3: child_clear_fn,        # 原 _id=1, 偏移 +2 → 3
    4: child_flag_fn,         # 原 _id=2, 偏移 +2 → 4
}
```

所有组件和事件函数都在一个扁平的字典中，不再有层级关系。**嵌套在运行时被消解为扁平结构**。

---

## 9. Interface 为何只保留一套运行时配置

### 9.1 构造阶段：Interface 属性的临时角色

Interface 的 `__init__` 方法（[interface.py#L84-L542](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/interface.py#L84-L542)）设置了大量自身属性，例如：

| 属性 | 用途 |
|------|------|
| `self.input_components` | 存储输入组件引用，传给 `attach_submit_events` |
| `self.output_components` | 存储输出组件引用，传给 `attach_submit_events` |
| `self.fn` | 用户函数，传给 `on()` / `.click()` |
| `self.interface_type` | 决定布局分支和事件绑定方式 |
| `self.flagging_callback` | 传给 `FlagMethod` 闭包 |
| `self.api_mode` | 控制 `preprocess`/`postprocess` 开关 |

但这些属性的使用 **全部发生在 `with self:` 代码块内部**，即构造阶段。一旦构造完成：

```python
self.config = self.get_config_file()  # L542
```

所有信息已被序列化到 `BlocksConfig` 中。`input_components` 和 `output_components` 的 `_id` 已经记录在 `BlockFunction.inputs` 和 `BlockFunction.outputs` 中；`fn` 已经被 `BlockFunction.fn` 持有；`flagging_callback` 已经被 `FlagMethod` 闭包捕获。

### 9.2 运行阶段：仅 BlocksConfig 和 BlockFunction 驱动

Gradio 运行时的请求处理流程是：

1. 前端发送事件请求，携带 `fn_index`
2. 后端通过 `self.fns[fn_index]` 找到 `BlockFunction`
3. 从 `BlockFunction.inputs` 取出输入组件的 `_id`，从 `blocks` 字典取出组件实例
4. 调用 `BlockFunction.fn`（即用户函数或 `FlagMethod` 闭包）
5. 将结果通过 `BlockFunction.outputs` 中的组件 `_id` 返回前端

全程不需要访问 Interface 的 `input_components`、`output_components`、`fn` 等属性。`BlockFunction` 持有所有运行时必要信息的引用。

### 9.3 FlagMethod 闭包的间接持有

Interface 的 `flagging_callback` 并非完全无人引用。它被 `FlagMethod`（[flagging.py#L392-L426](file:///d:/fz/0601/solo-dogfeeding/code/238-gradio/gradio/flagging.py#L392-L426)）闭包持有：

```python
class FlagMethod:
    def __init__(self, flagging_callback, label, value, ...):
        self.flagging_callback = flagging_callback

    def __call__(self, request, *flag_data):
        self.flagging_callback.flag(list(flag_data), ...)
```

而 `FlagMethod` 实例又作为 `fn` 参数传给了 `BlockFunction`。所以 `flagging_callback` 的生命周期由 `BlockFunction.fn` → `FlagMethod` → `flagging_callback` 这条引用链保证，与 Interface 自身无关。

### 9.4 嵌入后的 Interface 属性彻底失效

当 Interface 被 `.render()` 嵌入父 Blocks 后，情况更进一步：

- **组件引用可能指向过时的列表**：`self.input_components` 仍是构造时的组件对象列表，但这些组件的 `_id` 已经在父的 `blocks` 字典中被注册。Interface 自身持有的 `default_config.blocks` 和 `default_config.fns` 已被合并到父中，Interface 的 `BlocksConfig` 实质上变为空壳。

- **事件函数已被搬走**：`Blocks.render()` 的 `root_context.fns[dependency._id] = dependency` 将子的 `BlockFunction` 搬入父的 `fns` 字典。子的 `default_config.fns` 中虽然仍保留引用，但运行时只有父的 `fns` 字典被使用。

- **唯一例外：`self.config`**：Interface 在 `__init__` 末尾通过 `self.config = self.get_config_file()` 保存了一份 JSON 配置。但如果 Interface 作为子组件嵌入父 Blocks，这份配置不会被使用——父 Blocks 会在自己的 `__exit__` 中重新调用 `self.get_config_file()` 生成包含所有子组件的全局配置。

### 9.5 总结：一套配置的本质

所谓"一套运行时配置"的本质是：**整个 Gradio 应用在运行时只有一个 `BlocksConfig` 实例处于活跃状态**，即最外层根 Blocks 的 `default_config`。所有子 Blocks/Interface 的组件和事件函数在 `.render()` 时被合并到这个唯一的配置中，子的 `BlocksConfig` 从此不再参与运行时逻辑。

```
构造阶段:
  Interface.__init__
    → with self: (设置 Context.root_block = self)
    → 组件.render() → 注册到 self.default_config.blocks
    → 事件绑定 → 注册到 self.default_config.fns
    → self.config = self.get_config_file()   ← 局部配置快照

嵌入阶段 (interface.render() 被调用):
  Blocks.render()
    → root_context = 父 BlocksConfig
    → root_context.blocks.update(self.blocks)          ← 合并组件
    → dependency._id += offset; root_context.fns[...]   ← 合并函数
    → render_context.children.extend(self.children)     ← 合并布局

运行阶段:
  只有 root_block.default_config 中的 blocks 和 fns 被使用
  Interface 自身的 default_config 和属性全部不参与
```
