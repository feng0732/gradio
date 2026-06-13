# Gradio 自定义 JS 注入与事件生命周期分析

## 1. 概述

Gradio 提供了多层次的 JavaScript 注入能力，从全局页面脚本到单个事件的前端处理。这些 JS 注入与应用生命周期、组件事件之间的关系可以通过代码追踪清晰地展现。

---

## 2. JS 注入的四个层级

| 层级 | 注入方式 | 作用域 | 代码位置 |
|------|---------|--------|---------|
| L1 | `launch(head=...)` / `head_paths` | HTML `<head>` 内的任意标签 | [blocks.py#L2653](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2653-L2654) |
| L2 | `launch(js=...)` / `head_paths` 中的内联脚本 | 页面加载期，全局作用域 | [blocks.py#L2652](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2652) |
| L3 | 事件监听器的 `js="..."` 参数 | 事件触发时，前端预处理 | [block_function.py#L40](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/block_function.py#L40) |
| L4 | 事件监听器的 `js=True` (Python→JS 自动转译) | 整个事件纯前端执行 | [blocks.py#L2438-L2460](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2438-L2460) |

---

## 3. 脚本传递机制：Python → 前端

### 3.1 配置序列化流程

```
launch() 调用
  │
  ▼
Blocks.launch() ── 设置 self.js / self.head / self.css
  │
  ▼
Blocks.get_config_file() ── 将 js/head 写入 config JSON
  │  [blocks.py#L2393-L2394]
  │
  ▼
routes.py /config 路由 ── 返回 ORJSONResponse(config)
  │  [routes.py#L954-L975]
  │
  ▼
前端 Client.connect() 拉取 config
  │
  ▼
Index.svelte onMount() ── 执行注入逻辑
```

### 3.2 全局 JS/Head 的配置传递

在 [blocks.py#L2381-L2436](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2381-L2436) `get_config_file()` 中：

```python
config: BlocksConfigDict = {
    "js": self.js,          # L2: 全局 JS 字符串
    "head": self.head,      # L1: 自定义 head HTML
    "css": self.css,
    # ... 其他字段
}
```

### 3.3 事件级 JS 的配置传递

在 [block_function.py#L138-L175](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/block_function.py#L138-L175) `BlockFunction.get_config()` 中：

```python
return {
    "js": self.js,                        # L3: 事件级前端 js 代码
    "backend_fn": self.fn is not None,    # 是否有后端 Python 函数
    "js_implementation":                  # L4: Python→JS 转译结果
        getattr(self.fn, "__js_implementation__", None),
    "inputs": [...],
    "outputs": [...],
    # ...
}
```

---

## 4. 前端执行时机与生命周期

### 4.1 完整生命周期时序图

```
Browser 加载页面
  │
  ├─ 1. SPA main.ts 启动，挂载 <gradio-app> WebComponent
  │
  ├─ 2. Index.svelte onMount() 开始
  │   │
  │   ├─ 2a. Client.connect(api_url) → 拉取 config
  │   │
  │   ├─ 2b. mount_custom_css(config.css)        ← CSS 注入
  │   │     [Index.svelte#L359]
  │   │
  │   ├─ 2c. add_custom_html_head(config.head)   ← L1: Head 注入
  │   │     [Index.svelte#L360]
  │   │     内部实现：DOMParser 解析 → 逐个克隆节点到 document.head
  │   │     [Index.svelte#L172-L200]
  │   │
  │   ├─ 2d. 创建 <script> 标签执行 config.js   ← L2: 全局 JS 注入
  │   │     [Index.svelte#L372-L380]
  │   │     ```javascript
  │   │     const script = document.createElement("script");
  │   │     script.textContent = config.js;
  │   │     document.head.appendChild(script);
  │   │     ```
  │   │
  │   └─ 2e. dispatch("loaded") 事件
  │
  ├─ 3. load_demo() → 渲染 Blocks.svelte 组件
  │   │
  │   ├─ 3a. 创建 DependencyManager
  │   │     内部 new Dependency(dep_config) → 编译前端函数
  │   │     [dependency.ts#L52-L86]
  │   │
  │   ├─ 3b. 创建 AppTree → 构建组件树
  │   │
  │   └─ 3c. Svelte onMount() 内
  │         │
  │         └─ app_tree.ready.then(() => {
  │             ready = true;
  │             dep_manager.dispatch_load_events();  ← load 事件触发
  │         })
  │         [Blocks.svelte#L442-L445]
  │
  └─ 4. 用户交互触发组件事件
        │
        └─ gradio_event_dispatcher() → dep_manager.dispatch()
             [Blocks.svelte#L106-L158]
```

### 4.2 L1: `head` 注入执行细节

[Index.svelte#L172-L200](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L172-L200)

```javascript
async function add_custom_html_head(head_string: string | null): Promise<void> {
    if (head_string) {
        const parser = new DOMParser();
        const parsed_head_html = Array.from(
            parser.parseFromString(head_string, "text/html").head.children
        );
        for (let head_element of parsed_head_html) {
            let newElement = document.createElement(head_element.tagName);
            if (newElement.tagName === "SCRIPT") {
                (newElement as HTMLScriptElement).async = false; // 保序执行
            }
            // 复制 attributes 和 children
            document.head.appendChild(newElement);
        }
    }
}
```

**关键点**：SCRIPT 标签设置 `async=false`，确保按文档顺序执行。`head` 注入发生在全局 `js` 注入之前。

### 4.3 L2: 全局 `js` 注入执行细节

[Index.svelte#L372-L380](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L372-L380)

```javascript
if (config.js) {
    try {
        const script = document.createElement("script");
        script.textContent = config.js;
        document.head.appendChild(script);
    } catch (e) {
        console.error("Error executing custom JS:", e);
    }
}
```

**关键点**：
- 此时组件 DOM 尚未渲染（Blocks.svelte 还没 mount），但 `document`、`window` 可用
- 如果需要操作组件 DOM，应使用 DOMContentLoaded 监听或 `load` 事件
- 执行顺序：`head` → `js` → `load_demo()`

### 4.4 load 事件 (Blocks.load)

触发时机：[Blocks.svelte#L442-L445](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte#L442-L445)

```javascript
app_tree.ready.then(() => {
    ready = true;
    dep_manager.dispatch_load_events();  // 所有组件 DOM 已挂载完毕
});
```

dispatch_load_events 实现：[dependency.ts#L851-L864](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts#L851-L864)

```javascript
dispatch_load_events() {
    this.dependencies_by_fn.forEach((dep) => {
        dep.targets.forEach(([target_id, event_name]) => {
            if (event_name === "load") {
                this.dispatch({
                    type: "fn",
                    fn_index: dep.id,
                    event_data: null,
                    target_id: target_id
                });
            }
        });
    });
}
```

**自动 load 事件附加**：[blocks.py#L989-L1011](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L989-L1011) 中 `attach_load_events()` 会自动为有初始值需要后端计算的组件附加 `load` 事件。

---

## 5. Python 事件与前端 JS 协同机制

### 5.1 Dependency 构造：前端函数编译

[dependency.ts#L52-L86](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts#L52-L86)

```javascript
constructor(dep_config: IDependency) {
    this.functions = {
        // L3: 事件级 js 参数 - 预处理函数
        frontend: dep_config.js
            ? process_frontend_fn(
                    dep_config.js,
                    dep_config.backend_fn,
                    dep_config.inputs.length,
                    dep_config.outputs.length
                )
            : undefined,

        backend: dep_config.backend_fn,  // 是否有 Python 后端

        // L4: Python→JS 转译 - 纯前端执行
        backend_js: dep_config.js_implementation
            ? new AsyncFunction(
                    `let result = await (${dep_config.js_implementation})(...arguments);
                    return (!Array.isArray(result)) ? [result] : result;`
                )
            : undefined
    };
}
```

### 5.2 `process_frontend_fn`：L3 前端预处理包装

[dependency.ts#L935-L952](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts#L935-L952)

```javascript
export function process_frontend_fn(
    source: string,
    backend_fn: boolean,
    input_length: number,
    output_length: number
): (...args: unknown[]) => Promise<unknown[]> {
    const wrap = backend_fn ? input_length === 1 : output_length === 1;
    try {
        return new AsyncFunction(
            "__fn_args",
            `  let result = await (${source})(...__fn_args);
  if (typeof result === "undefined") return [];
  return (${wrap} && !Array.isArray(result)) ? [result] : result;`
        );
    } catch (e) {
        throw e;
    }
}
```

**执行逻辑说明**：
- `source` 被当作函数体引用，接收 inputs 作为参数
- 如果有后端函数（`backend_fn=true`），返回值作为后端输入；否则直接作为输出
- `wrap` 参数用于自动将单个值包装为数组（与 Python 端签名对齐）

### 5.3 Dependency.run()：三级执行流水线

[dependency.ts#L88-L129](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts#L88-L129)

```javascript
async run(client, data_payload, event_data, target_id) {
    let _data_payload = data_payload;

    // 优先级 1: js_implementation (js=True 转译) - 完全替代后端
    if (this.functions.backend_js) {
        const data = await this.functions.backend_js(..._data_payload);
        return { type: "data", data };
    }

    // 优先级 2: js 参数 - 前端预处理 → 传递给后端
    if (this.functions.frontend) {
        _data_payload = await this.functions.frontend(data_payload);
    }

    // 优先级 3: 后端 Python 函数
    if (this.functions.backend) {
        return {
            type: "submit",
            data: client.submit(this.id, _data_payload, event_data, ...)
        };
    } else if (this.functions.frontend) {
        return { type: "data", data: _data_payload };  // 纯前端返回
    }
    return { type: "void", data: null };
}
```

**三种协同模式对比**：

| 模式 | Python fn | `js=...` | `js=True` | 执行路径 |
|------|-----------|----------|-----------|---------|
| 纯后端 | ✅ | ❌ | ❌ | client.submit → Python |
| 前后端串联 | ✅ | ✅ 字符串 | ❌ | frontend() → client.submit → Python |
| 纯前端 (L3) | ❌ | ✅ 字符串 | ❌ | frontend() 直接返回 |
| 纯前端 (L4 转译) | ✅ | ❌ | ✅ | transpile → backend_js() 纯前端 |

### 5.4 js=True: Python→JS 自动转译流程

[blocks.py#L2438-L2460](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2438-L2460)

```python
def transpile_to_js(self, quiet: bool = False):
    fns_to_transpile = [
        fn.fn for fn in self.fns.values() if fn.fn and fn.js is True
    ]
    for fn in fns_to_transpile:
        if getattr(fn, "__js_implementation__", None) is None:
            fn.__js_implementation__ = transpile(fn, validate=True)
            # 使用 groovy.transpile 将 Python AST → JS 代码
```

转译结果通过 `BlockFunction.get_config()` 的 `js_implementation` 字段传到前端。

**示例 - todo_list_js/run.py**：[run.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/demo/todo_list_js/run.py)

```python
t.change(lambda : gr.Button(interactive=True, variant="primary"),
         None, b, js=True)  # Python 函数自动转译为 JS
```

### 5.5 DependencyManager：事件调度中枢

[dependency.ts#L194+](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts#L194)

核心调度方法 `dispatch()`：

```javascript
async dispatch(event_meta: DispatchFunction | DispatchEvent): Promise<void> {
    // 1. 查找匹配的 dependencies
    if (event_meta.type === "fn") {
        deps = [this.dependencies_by_fn.get(event_meta.fn_index)];
    } else {
        deps = this.dependencies_by_event.get(
            `${event_meta.event_name}-${event_meta.target_id}`
        );
    }

    for (const dep of deps) {
        this.cancel(dep.cancels);  // 处理 cancels

        const dispatch_status = should_dispatch(
            dep.trigger_modes,   // once | multiple | always_last
            this.submissions.has(dep.id)
        );

        if (dispatch_status === "skip") continue;
        if (dispatch_status === "defer") { this.queue.add(dep.id); continue; }

        // 2. 收集 inputs → 3. 调用 dep.run() → 4. 处理 outputs
        // 5. 触发 trigger_after (then/success/failure 链)
    }
}
```

**协同控制参数**（来自 [block_function.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/block_function.py)）：

| 参数 | 含义 | 生效位置 |
|------|------|---------|
| `trigger_after` | 链式事件 ID (.then/.success/.failure) | dep_manager 完成主 dep 后触发 |
| `trigger_mode` | `"once"` / `"multiple"` / `"always_last"` | dispatch 时决定是否排队或跳过 |
| `cancels` | 触发时需要取消的其他事件 ID | dispatch 首步执行 this.cancel() |
| `collects_event_data` | 是否收集前端 Event 对象到 Python | 序列化时通过 event_data 传递 |

---

## 6. 完整协同示例追踪

以一个典型的按钮点击事件为例，包含前端 JS 预处理 + 后端 Python 处理：

```python
with gr.Blocks() as demo:
    inp = gr.Textbox()
    out = gr.Textbox()
    btn = gr.Button("Run")

    btn.click(
        fn=lambda x: x.upper(),    # Python 后端
        inputs=inp,
        outputs=out,
        js="(x) => x.trim()"       # 前端预处理
    )

demo.launch(js="console.log('global init')")
```

**执行追踪**：

1. **配置阶段**：`BlockFunction.get_config()` 输出
   ```json
   {
     "id": 0,
     "targets": [[3, "click"]],
     "inputs": [1], "outputs": [2],
     "js": "(x) => x.trim()",
     "backend_fn": true
   }
   ```

2. **前端初始化**：
   - `Index.svelte` onMount → 注入 `console.log('global init')`
   - Blocks.svelte onMount → DependencyManager 编译 `process_frontend_fn("(x)=>x.trim()", true, 1, 1)`

3. **用户点击按钮**：
   - 组件 Button emit `click` → `gradio_event_dispatcher(3, "click", null)`
   - `dep_manager.dispatch({type:"event", event_name:"click", target_id:3})`

4. **Dependency.run() 执行**：
   - Step1: frontend() → `"  HELLO  " → "HELLO"`
   - Step2: backend → client.submit(0, ["HELLO"]) → WebSocket/SSE 请求
   - Step3: Python 路由 `/call/0` 执行 `lambda x: x.upper()` → `"HELLO"`

5. **结果回传**：
   - SSE `data` 事件 → dep_manager.update_state(2, {value: "HELLO"})
   - AppTree 更新 → Svelte reactive → Textbox 组件渲染新值

---

## 7. 关键文件索引

| 功能模块 | 文件 | 关键代码位置 |
|---------|------|------------|
| 全局 JS/Head 注入 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py) | `launch()` L2602+ / `get_config_file()` L2381+ |
| 事件函数配置 | [block_function.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/block_function.py) | `get_config()` L138+ |
| 配置 HTTP API | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/routes.py) | `/config` 路由 L954+ |
| SPA 入口注入 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | onMount L306+ / 注入 JS L372+ |
| Head 注入实现 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | `add_custom_html_head()` L172+ |
| 依赖管理与运行 | [dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts) | `Dependency` L25+ / `DependencyManager` L194+ |
| 前端函数编译 | [dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts) | `process_frontend_fn()` L935+ |
| load 事件触发 | [Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte) | onMount L442+ |
| 事件派发器 | [Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte) | `gradio_event_dispatcher()` L106+ |
| Python→JS 转译 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py) | `transpile_to_js()` L2438+ |
