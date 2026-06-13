# Gradio Examples 缓存机制代码理解

本文档追踪 Gradio 中 examples（示例）的加载、缓存预运行、缓存命中和懒加载的完整代码路径。

---

## 1. 核心类与文件

| 文件 | 关键类/函数 | 作用 |
|------|------------|------|
| [helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py) | `Examples` 类、`create_examples()` | 示例管理核心类，处理缓存逻辑 |
| [interface.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/interface.py) | `Interface.render_examples()` | Interface 层调用 Examples 构造器 |
| [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/blocks.py) | `Blocks.extra_startup_events`、`process_api()` | 启动时触发预缓存、调用用户函数 |
| [flagging.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/flagging.py) | `CSVLogger` | 将缓存结果写入 CSV 文件 |
| [components/dataset.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/components/dataset.py) | `Dataset` | 前端展示示例的 UI 组件 |
| [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/routes.py) | `/startup-events` 路由 | 触发启动事件（包括预缓存） |
| [utils.py](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/utils.py) | `get_cache_folder()` | 获取缓存目录 |

---

## 2. 完整流程图

```
用户代码
  │
  ├─ Interface(cache_examples=True, cache_mode="eager"|"lazy")
  │     │
  │     └─ Interface.render_examples()  [interface.py:910]
  │           │
  │           └─ Examples(...) 构造器  [helpers.py:103]
  │                 │
  │                 ├─ 解析参数：确定 cache_examples 是 True/"lazy"/False
  │                 ├─ 处理输入输出组件
  │                 ├─ 创建 Dataset 组件 (前端展示用)
  │                 ├─ 确定缓存目录 cached_folder
  │                 └─ 初始化 CSVLogger (cache_logger)
  │
  └─ demo.launch()
        │
        ├─ FastAPI lifespan (mount_scenarios)  [routes.py:2584]
        │   ├─ run_startup_events()           [blocks.py:3372]
        │   └─ run_extra_startup_events()     [blocks.py:3380]
        │         │
        │         └─ Examples._start_caching()  [helpers.py:497]
        │               │
        │               └─ (仅 eager 模式) Examples.cache()  [helpers.py:512]
        │
        └─ 浏览器请求 /startup-events 路由      [routes.py:1774]
            └─ (如 lifespan 未触发则再次触发) run_startup_events + run_extra_startup_events
```

---

## 3. 参数解析与构造阶段

### 3.1 cache_examples 参数判定 ([helpers.py:155-170](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L155-L170))

```python
self.cache_examples = False
if cache_examples is None:
    # 从环境变量 GRADIO_CACHE_EXAMPLES 读取
    if os.getenv("GRADIO_CACHE_EXAMPLES", "").lower() in ["true", "lazy"] \
       and fn is not None and outputs is not None:
        self.cache_examples = True
elif cache_examples in [True, False]:
    self.cache_examples = cache_examples
```

**注意：** `cache_examples` 参数只接受 `True`/`False`，`"lazy"` 是通过 `cache_mode` 参数控制的。

### 3.2 cache_mode 参数判定 ([helpers.py:172-185](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L172-L185))

```python
# 先检查环境变量 GRADIO_CACHE_MODE
if self.cache_examples and cache_mode == "lazy":
    self.cache_examples = "lazy"  # 从 True 变成字符串 "lazy"
```

最终 `self.cache_examples` 的可能取值：
- `False` - 不缓存
- `True` - 启用缓存，**eager 模式**（启动时全部预运行）
- `"lazy"` - 启用缓存，**lazy 模式**（首次点击时才运行）

### 3.3 缓存目录 ([helpers.py:289-299](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L289-L299))

```python
self.cache_logger = CSVLogger(simplify_file_data=False, verbose=False, dataset_file_name="log.csv")
self.cached_folder = utils.get_cache_folder() / str(self.dataset._id)
# GRADIO_RESET_EXAMPLES_CACHE=True 时删除缓存
if os.environ.get("GRADIO_RESET_EXAMPLES_CACHE") == "True" and self.cached_folder.exists():
    shutil.rmtree(self.cached_folder)
self.cached_file = Path(self.cached_folder) / "log.csv"          # 存输出数据
self.cached_indices_file = Path(self.cached_folder) / "indices.csv"  # 存已缓存的索引
```

默认缓存目录：`.gradio/cached_examples/{dataset_id}/`，可通过环境变量 `GRADIO_EXAMPLES_CACHE` 修改。

---

## 4. Examples.create() - 事件绑定阶段 ([helpers.py:352-471](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L352-L471))

### 4.1 注册启动事件

```python
blocks_config = get_blocks_context()
if blocks_config:
    if self.root_block:
        # 关键：把 _start_caching 加入启动事件列表
        self.root_block.extra_startup_events.append(self._start_caching)
```

### 4.2 两种模式：缓存 vs 不缓存

**情况 A：启用缓存 (`self.cache_examples` 为 True 或 "lazy")**

```python
def load_example_input(example_tuple):
    # example_tuple = (index, example_value) - Dataset type="tuple"
    _, example_value = example_tuple
    processed_example = self._get_processed_example(example_value)
    return utils.resolve_singleton(processed_example)

def load_example_output(example_tuple):
    example_id, _ = example_tuple
    # 关键：点击时加载缓存（或触发懒缓存）
    cached_outputs = self.load_from_cache(example_id)
    return utils.resolve_singleton(cached_outputs)

# 事件链：先填输入 → 再填输出
self.cache_event = self.dataset.click(load_example_input, ...) \
    .then(load_example_output, ...)
```

**情况 B：不启用缓存 (`self.cache_examples=False`)**

```python
def load_example(example_tuple):
    # 只填充输入组件，不运行函数
    # 如果 run_on_click=True，则 .then(self.fn, ...) 再跑函数
```

### 4.3 preload 预加载机制 ([helpers.py:392-425](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L392-L425))

条件：`preload is not False` 且 **非 lazy 模式** 且输入组件没有手动设置 `value`

```python
# 页面加载时自动调用 load_example_input + load_example_output
# 将第 preload 个示例的输入和输出都填好
self.root_block.load(load_example_input, inputs=[State((preload, examples[preload]))], outputs=self.inputs)
self.root_block.load(load_example_output, inputs=[State((preload, examples[preload]))], outputs=self.outputs)
```

---

## 5. Eager 模式 - 预运行缓存逻辑

### 5.1 触发入口：_start_caching() ([helpers.py:497-510](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L497-L510))

```python
async def _start_caching(self):
    if self.cache_examples:
        # ... 校验所有输入组件是否都有 example 值
        if self.cache_examples is True:   # 只有 eager 模式才在这里调用
            await self.cache()
```

### 5.2 核心：Examples.cache() ([helpers.py:512-580](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L512-L580))

**步骤 1：判断是否已有全量缓存**
```python
if Path(self.cached_file).exists() and example_id is None:
    # 已有 log.csv 且是全量缓存（非单个），直接跳过
    print("Using cache from '...' directory ...")
    return
```

**步骤 2：处理生成器函数**
```python
fn, generated_values = resolve_generator(self.fn)
# resolve_generator: 如果是 generator，包装成普通函数并捕获所有 yield
```

**步骤 3：创建「假事件」用于调用 processing pipeline**
```python
# 在 Blocks 的 default_config 中创建一个临时的 load 事件
_, fn_index = self.root_block.default_config.set_event_trigger(
    [EventListenerMethod(Context.root_block, "load")],
    fn=fn,
    inputs=self.inputs,
    outputs=self.outputs,
    preprocess=self.preprocess and not self._api_mode,
    postprocess=self.postprocess and not self._api_mode,
    batch=self.batch,
)
```
这里复用了 Blocks 完整的 preprocess → call_function → postprocess 管线。

**步骤 4：遍历每个示例，逐一执行并缓存**
```python
for i, example in enumerate(self.non_none_examples):
    if example_id is not None and i != example_id:
        continue  # lazy 模式只处理单个

    processed_input = self._get_processed_example(example)
    # 补回 None 值的输入组件
    for index, keep in enumerate(self.input_has_examples):
        if not keep:
            processed_input.insert(index, None)
    if self.batch:
        processed_input = [[value] for value in processed_input]

    # 核心：调用 Blocks 的 process_api 走完整处理流程
    prediction = await self.root_block.process_api(
        block_fn=self.root_block.default_config.fns[fn_index],
        inputs=processed_input,
        request=None,
        in_event_listener=self.cache_examples != "lazy",  # lazy=False
    )
    output = prediction["data"]

    # 处理生成器输出（如 streaming）
    if generated_values:
        output = await merge_generated_values_into_output(...)
    if self.batch:
        output = [value[0] for value in output]

    # 写入 CSV
    self.cache_logger.flag(output)
    # 记录已缓存的索引
    with open(self.cached_indices_file, "a") as f:
        f.write(f"{example_id or i}\n")
```

**步骤 5：清理临时事件**
```python
self.root_block.default_config.fns.pop(fn_index)  # 删除假事件
```

### 5.3 CSVLogger.flag() - 写缓存文件 ([flagging.py:293-345](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/flagging.py#L293-L345))

- **第一次调用**：创建 log.csv，写入表头（各输出组件 label + timestamp）
- **每次调用**：对每个输出组件调用 `component.flag(sample)` 序列化 → 写入一行 CSV
- 对文件类组件：文件被复制到 `{flagging_dir}/{component_label}/` 目录下
- 缓存专用 logger 使用 `simplify_file_data=False` 保留完整 FileData 结构

---

## 6. 懒加载（Lazy）模式与缓存命中

### 6.1 懒加载触发时机

用户点击某个示例 → 事件链：
1. `load_example_input` - 填输入
2. `load_example_output` - 调 `load_from_cache(example_id)`

### 6.2 load_from_cache() - 命中/懒加载核心 ([helpers.py:582-619](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L582-L619))

```python
def load_from_cache(self, example_id: int) -> list[Any]:
    # 步骤 1：查 indices.csv 是否已经缓存过这个 example
    cached_index = self._get_cached_index_if_cached(example_id)

    if cached_index is None:
        # ===== 懒加载分支 =====
        # 同步调用 cache(example_id) 只缓存这一个示例
        client_utils.synchronize_async(self.cache, example_id)
        # 此时它刚被追加到 indices.csv 最后一行
        with open(self.cached_indices_file) as f:
            cached_index = len(f.readlines()) - 1

    # 步骤 2：从 log.csv 读取缓存数据
    with open(self.cached_file, encoding="utf-8") as cache:
        examples = list(csv.reader(cache))
    example = examples[cached_index + 1]  # +1 跳过表头

    # 步骤 3：反序列化每个输出组件
    output = []
    for component, value in zip(self.outputs, example):
        try:
            value_as_dict = ast.literal_eval(value)
            # 如果是 gr.update(...) 格式的 dict，直接用
            if isinstance(value_as_dict, list) and isinstance(component, components.File):
                value_to_use = value_as_dict  # 多文件列表
            if not utils.is_prop_update(value_as_dict):
                raise TypeError
            output.append(value_as_dict)
        except (ValueError, TypeError, SyntaxError):
            # 否则用组件的 read_from_flag 还原
            output.append(component.read_from_flag(value_to_use))
    return output
```

### 6.3 _get_cached_index_if_cached() - 索引查询 ([helpers.py:488-495](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L488-L495))

```python
def _get_cached_index_if_cached(self, example_index) -> int | None:
    # indices.csv 每行存一个 example_id，按写入顺序排列
    if Path(self.cached_indices_file).exists():
        with open(self.cached_indices_file) as f:
            cached_indices = [int(line.strip()) for line in f]
        if example_index in cached_indices:
            # 返回在 indices.csv 中的位置 = log.csv 中的行号-1
            return cached_indices.index(example_index)
    return None
```

**为什么需要 indices.csv？** - 因为 lazy 模式下用户点击顺序与 examples 列表顺序不一定一致，需要一个映射表。

---

## 7. 输入处理：_get_processed_example()

将原始 example 值（如本地文件路径字符串）转换成前端可直接渲染的格式：

```python
def _get_processed_example(self, example):
    with utils.set_directory(self.working_directory):
        sub = []
        for component, sample in zip(self.inputs_with_examples, example):
            # 组件 postprocess: 原始值 → 前端数据模型
            prediction_value = component.postprocess(sample)
            # 转成 dict (GradioModel → model_dump)
            if isinstance(prediction_value, (GradioRootModel, GradioModel)):
                prediction_value = prediction_value.model_dump()
            # 把文件移到缓存目录并生成 URL
            prediction_value = processing_utils.move_files_to_cache(
                prediction_value, component, postprocess=True,
            )
            sub.append(prediction_value)
        return sub
```

这个结果存在 `self.non_none_processed_examples` 字典中复用，避免重复计算。

---

## 8. 启动事件的双触发机制

**两种触发路径都会调用 run_extra_startup_events()：**

| 触发时机 | 位置 | 说明 |
|---------|------|------|
| FastAPI lifespan | [routes.py:2584-2591](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/routes.py#L2584-L2591) | 使用 `gr.mount_gradio_app()` 挂载时 |
| `/startup-events` 路由 | [routes.py:1774-1781](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/routes.py#L1774-L1781) | 前端 JS 启动时请求，用 `startup_events_triggered` 标志防重 |

```python
@router.get("/startup-events")
async def startup_events():
    if not app.startup_events_triggered:   # 幂等保护
        app.get_blocks().run_startup_events()
        await app.get_blocks().run_extra_startup_events()
        app.startup_events_triggered = True
        return True
    return False
```

**注意：** eager 模式下预缓存是串行的，每个示例依次调用 `process_api`，示例多时启动较慢。

---

## 9. 与 gr.cache 装饰器的区别

| 特性 | Examples 缓存 | `@gr.cache` 装饰器 |
|------|-------------|-------------------|
| **存储位置** | 磁盘 CSV + 文件目录 | 内存 OrderedDict (LRU) |
| **适用对象** | Examples 组件的输入输出 | 任意函数调用 |
| **生命周期** | 跨进程持久化（磁盘文件） | 进程内，重启丢失 |
| **命中方式** | 按 example 索引查 indices.csv | 按参数内容 hash 查字典 |
| **写入时机** | eager=启动时 / lazy=首次点击 | 首次调用函数时 |
| **序列化** | component.flag() / read_from_flag() | 直接 deepcopy 内存对象 |
| **生成器支持** | 需 merge_generated_values_into_output 合并 | 原生捕获所有 yield 列表 |

---

## 10. 关键文件目录结构

```
.gradio/cached_examples/
└── {dataset_id}/              # 每个 Examples 组件一个子目录，ID 是 Dataset._id
    ├── log.csv                # 缓存的输出值 (CSVLogger)
    │   ├── 第1行：表头（输出组件名 + timestamp）
    │   └── 后续行：每个示例一行，component.flag() 序列化结果
    ├── indices.csv            # 已缓存示例的索引列表（按写入顺序，lazy 模式用）
    │   └── 每行一个数字：example_index
    └── {component_label}/     # 文件类组件的文件存储目录
        └── ...
```

重置缓存：删除整个目录，或设环境变量 `GRADIO_RESET_EXAMPLES_CACHE=True`。

---

## 11. 特殊情况处理

### State 组件禁用缓存 ([interface.py:270-275](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/interface.py#L270-L275))
```python
if cache_examples:
    warnings.warn("Cache examples cannot be used with state inputs and outputs.")
self.cache_examples = False
```

### 生成器函数的输出合并 ([helpers.py:622-649](file:///d:/fz/0601/solo-dogfeeding/code/245-gradio/gradio/helpers.py#L622-L649))
- `resolve_generator()` 把生成器包装成普通函数，捕获所有 yield 到 `generated_values` 列表
- `merge_generated_values_into_output()` 对每个 StreamingOutput 组件：
  - 对每个 chunk 调 postprocess → stream_output 拿二进制
  - 最后调 combine_stream 合并成最终文件（如完整音频/视频）

### batch 模式
- 缓存时：输入包一层 list，输出再解包 `[value[0] for value in output]`
- 因为 `process_api` 的 batch 模式要求每个输入是 N 长度列表。

---

## 12. 事件链汇总

### Eager 模式启动流程
```
launch()
  → lifespan /startup-events
    → run_extra_startup_events
      → _start_caching()
        → cache()           ← 一次性跑所有 examples，存 log.csv + indices.csv
```

### 用户点击示例（Eager 已预缓存）
```
dataset.click
  → load_example_input      ← 填输入（从内存 non_none_processed_examples 取）
  → .then(load_example_output)
    → load_from_cache(id)
      → _get_cached_index_if_cached → 找到 (命中)
      → 读 log.csv + component.read_from_flag()
      → 返回输出值
```

### 用户点击示例（Lazy 模式，首次点击第 i 个）
```
dataset.click
  → load_example_input      ← 填输入
  → .then(load_example_output)
    → load_from_cache(i)
      → _get_cached_index_if_cached → None (未命中)
      → synchronize_async(cache, i)   ← 在这里同步阻塞调用 cache(i)
        → 对单个 i 执行 process_api
        → 追加写入 log.csv 和 indices.csv
      → 读 log.csv 最后一行
      → 返回输出值
后续再点同一个 → 直接命中
```

### 非缓存模式
```
dataset.click
  → load_example (只填输入)
  → (如果 run_on_click=True) .then(fn)  ← 实时跑函数，不存结果
```
