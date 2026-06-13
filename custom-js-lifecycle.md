# Gradio 自定义 JS 注入与事件生命周期分析

## 1. 概述

Gradio 提供了多层次的 JavaScript 注入能力，从全局页面脚本到单个事件的前端处理。这些 JS 注入与应用生命周期、组件事件之间的关系可以通过代码追踪清晰地展现。

本文档重点澄清**四个易混淆概念**的执行顺序和适用场景：
- `launch(head=...)` — L1 级头部注入
- `launch(js=...)` — L2 级全局脚本
- `dispatch("loaded")` — 页面级加载完成信号
- `app_tree.ready` + `dispatch_load_events()` — 组件 load 事件

---

## 2. JS 注入的四个层级

| 层级 | 注入方式 | 作用域 | 代码位置 |
|------|---------|--------|---------|
| L1 | `launch(head=...)` / `head_paths` | HTML `<head>` 内的任意标签 | [blocks.py#L2653](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2653-L2654) |
| L2 | `launch(js=...)` | 页面加载期，全局作用域 | [blocks.py#L2652](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L2652) |
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

## 4. 前端执行时机与生命周期（修正版）

### 4.1 精确执行时序

以下是基于 `Index.svelte` onMount 真实代码的精确时序：

[Index.svelte#L306-L427](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L306-L427)

```
Browser 加载页面
  │
  ├─ 1. SPA main.ts 启动，挂载 <gradio-app> WebComponent
  │
  ├─ 2. Index.svelte onMount() 开始 (L306)
  │   │
  │   ├─ 2a. Client.connect(api_url) → HTTP /config 拉取配置
  │   │   (await 阻塞)
  │   │
  │   ├─ 2b. await mount_custom_css(config.css)
  │   │     [Index.svelte#L359]
  │   │
  │   ├─ 2c. await add_custom_html_head(config.head)   ← L1: Head 注入
  │   │     [Index.svelte#L360]
  │   │     内部实现：DOMParser 解析 → 逐个克隆节点到 document.head
  │   │     [Index.svelte#L172-L226]
  │   │     注意：SCRIPT 标签 async=false，按文档顺序阻塞执行
  │   │
  │   ├─ 2d. dispatch("loaded")                        ← 页面级 loaded 信号
  │   │     [Index.svelte#L364]
  │   │     ⚠️  重要：此时组件 DOM **尚未**渲染！
  │   │     ⚠️  重要：全局 config.js 也 **尚未** 执行！
  │   │
  │   ├─ 2e. 响应式触发 load_demo()
  │   │     [Index.svelte#L436]
  │   │     $: if (config && (eager || $intersecting[_id])) load_demo();
  │   │     └─> 动态 import("@gradio/core/blocks")
  │   │         └─> 等待 CSS 就绪 + i18n 就绪
  │   │             └─> Blocks.svelte 组件开始挂载
  │   │
  │   └─ 2f. 同步执行 config.js                       ← L2: 全局 JS 注入
  │         [Index.svelte#L372-L380]
  │         ```javascript
  │         if (config.js) {
  │             const script = document.createElement("script");
  │             script.textContent = config.js;
  │             document.head.appendChild(script);  // 同步阻塞执行
  │         }
  │         ```
  │         ⚠️  重要：此时 Blocks.svelte 可能仍在导入中，组件 DOM **尚未**渲染
  │
  └─ 3. Blocks.svelte 组件挂载 (异步)
      │
      ├─ 3a. 构造 AppTree (组件树)
      │   [init.svelte.ts#L96-L144]
      │   ├─ 初始化 this.ready = new Promise(...)
      │   ├─ 构建 components_to_register Set（所有 visible=true 的组件）
      │   └─ 递归 traverse 布局树
      │
      ├─ 3b. 构造 DependencyManager
      │   [dependency.ts#L194+]
      │   └─ new Dependency(dep_config) → 编译前端函数
      │      [dependency.ts#L52-L86]
      │
      ├─ 3c. Svelte 渲染 <MountComponents node={app_tree.root} />
      │   [Blocks.svelte#L479]
      │   │
      │   └─ 每个组件的 Gradio 基类 constructor 中调用：
      │      this.register_component(id, this.set_data, this.get_data)
      │      [utils.svelte.ts#L430-L435]
      │      │
      │      └─ app_tree.register_component(id, ...)
      │         ├─ this.components_to_register.delete(id)
      │         └─ if (size === 0) this.ready_resolve()
      │            [init.svelte.ts#L232-L235]
      │
      └─ 3d. Blocks.svelte onMount() 内
            [Blocks.svelte#L422-L456]
            │
            └─ app_tree.ready.then(() => {
                ready = true;
                dep_manager.dispatch_load_events();   ← 组件 load 事件触发
            })
            [Blocks.svelte#L442-L445]
```

### 4.2 关键时间点对比表

| 事件 | 代码位置 | 组件 DOM 是否渲染 | 能否操作组件 | 适用场景 |
|------|---------|-----------------|-------------|---------|
| **head 注入** | [Index.svelte#L360](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L360) | ❌ 未渲染 | ❌ 不能 | 注入第三方库、CSS 样式、全局配置脚本 |
| **`dispatch("loaded")`** | [Index.svelte#L364](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L364) | ❌ 未渲染 | ❌ 不能 | 父级容器监听页面加载完成状态（与组件无关） |
| **全局 `js` 执行** | [Index.svelte#L372-L380](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L372-L380) | ❌ 未渲染 | ❌ 不能 | 定义全局函数、挂载 window 变量、注册全局事件监听器 |
| **`app_tree.ready`** | [init.svelte.ts#L234](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L234) | ✅ 已挂载 | ✅ 能（DOM 已存在） | 组件注册全部完成的内部信号 |
| **`dispatch_load_events()`** | [Blocks.svelte#L444](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte#L444) | ✅ 已挂载 | ✅ 能 | 组件 load 事件触发，可读取/修改组件值 |
| **`render_complete` → CustomEvent("render")** | [Index.svelte#L495-L503](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L495-L503) | ✅ 已挂载 | ✅ 能 | 嵌入场景通知父页面渲染完成 |

### 4.3 L1: `head` 注入执行细节

[Index.svelte#L172-L226](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L172-L226)

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
            // 复制 attributes、textContent、children
            document.head.appendChild(newElement);
        }
    }
}
```

**关键点**：
- `await` 会等待所有外部资源（如 `<script src>`、`<link rel="stylesheet">`）加载完成
- 内联 `<script>` 会在 `appendChild` 时**同步阻塞**执行
- `async=false` 保证多个 SCRIPT 按文档顺序执行
- 执行时机早于 `dispatch("loaded")`，早于全局 `js`

### 4.4 `dispatch("loaded")`：页面级信号（⚠️ 不是 DOM ready！）

[Index.svelte#L364](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L364)

```javascript
await mount_custom_css(config.css);
await add_custom_html_head(config.head);
css_ready = true;
dispatch("loaded");   // ← 此处触发

// 下面的代码在 dispatch("loaded") 之后才执行
pages = config.pages;
current_page = config.current_page;
root = config.root;
// ...
if (config.js) { /* 注入全局 JS */ }
```

**重要澄清**：
- `dispatch("loaded")` 是 Svelte 组件事件，只通知父级 `<gradio-app>` 容器
- 触发时：✅ CSS 就绪 / ✅ Head 资源加载完成 / ❌ 组件 DOM **未** 渲染 / ❌ 全局 js **未** 执行
- **与 `load` 事件无关**，是页面级容器状态信号
- 触发后才执行全局 `js` 注入，然后才开始动态 import Blocks 模块

### 4.5 L2: 全局 `js` 注入执行细节

[Index.svelte#L372-L380](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L372-L380)

```javascript
if (config.js) {
    try {
        const script = document.createElement("script");
        script.textContent = config.js;
        document.head.appendChild(script);  // 同步阻塞执行
    } catch (e) {
        console.error("Error executing custom JS:", e);
    }
}
```

**关键点**：
- **执行顺序**：`head` → `dispatch("loaded")` → **全局 `js`**
- 此时：✅ `window`/`document` 可用 / ❌ Blocks.svelte 组件可能仍在导入 / ❌ 组件 DOM **未** 渲染
- 如果需要操作组件 DOM，**必须**用 `load` 事件，不能用全局 `js` 直接查询 DOM
- 作用于全局作用域，可以定义 `window.myFn = function() {...}` 供后续事件级 JS 调用

### 4.6 `app_tree.ready`：组件就绪的内部 Promise

[init.svelte.ts#L232-L235](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L232-L235)

```javascript
if (this.components_to_register.size === 0 && !this.resolved) {
    this.resolved = true;
    this.ready_resolve();   // ← 所有组件注册完成
}
```

**触发条件**：
- AppTree 构造时先将所有 `visible=true` 的组件 ID 加入 `components_to_register` Set
  [init.svelte.ts#L123-L125](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L123-L125)
- 不可见组件（`visible=false`、折叠的 accordion 子项、非活动 tab 子项）会被 `_untrack()` 从 Set 中排除
  [init.svelte.ts#L818-L871](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L818-L871)
- 每个可见组件在其 Gradio 基类 constructor 中调用 `register_component()` 时从 Set 中删除 ID
  [utils.svelte.ts#L430-L435](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/utils/src/utils.svelte.ts#L430-L435)
- 当 Set size 降为 0 时 resolve `app_tree.ready`

**注意**：非活动 tab / 折叠 accordion 中的组件不会触发 ready，直到它们被展开时才会注册。

### 4.7 `load` 事件：组件级初始化事件

触发时机：[Blocks.svelte#L442-L445](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte#L442-L445)

```javascript
app_tree.ready.then(() => {
    ready = true;
    dep_manager.dispatch_load_events();  // 所有可见组件 DOM 已挂载
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

**两种 load 事件来源**：

1. **显式声明**：`demo.load(fn=..., inputs=..., outputs=...)` 或 `component.load(fn)`
2. **自动附加**：[blocks.py#L989-L1011](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L989-L1011) `attach_load_events()` 为有初始值需要后端计算的组件（如 `gr.Video(value=...)`）自动附加 `load` 事件

**`load` 事件的 JS 能力**：
- 触发时组件 DOM 已完全挂载，可以通过 `document.getElementById()` 查询组件元素
- 可以读取和修改组件值（`gr.State`、`gr.Textbox` 等）
- 支持 L3（`js="..."` 预处理）和 L4（`js=True` 转译）两种前端执行方式
- 如果有 Python 后端，会正常发起 `/call` 请求

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

**四种协同模式对比**：

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

demo.launch(
    head="<script>window.LIB_VERSION='1.0'</script>",
    js="console.log('global init', window.LIB_VERSION);"
)
```

**执行追踪（精确时序）**：

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

2. **前端初始化时序**：
   - T0: `Index.svelte` onMount → `Client.connect()` 拉取 config
   - T1: `await mount_custom_css(config.css)` → CSS 就绪
   - T2: `await add_custom_html_head(config.head)` → 执行 `<script>window.LIB_VERSION='1.0'</script>`
   - T3: `dispatch("loaded")` → 通知父容器（组件 DOM 未渲染）
   - T4: `load_demo()` 触发 → 开始动态 import Blocks 模块（异步）
   - T5: 注入全局 `js` → 执行 `console.log('global init', '1.0')` → 输出 `global init 1.0`
   - T6: Blocks 模块加载完成 → 构造 AppTree（components_to_register = {0,1,2,3}）
   - T7: Blocks.svelte 首次渲染 → 每个组件 constructor 调用 `register_component`，逐个从 Set 中删除
   - T8: 最后一个组件注册完成 → `app_tree.ready_resolve()`
   - T9: `dep_manager.dispatch_load_events()` → 触发所有 load 事件

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

## 7. 常见问题与最佳实践

### Q1: 全局 `js` 中为什么 `document.getElementById("component-id")` 返回 null？

**原因**：全局 `js` 执行时（T5），Blocks.svelte 组件仍在异步导入/渲染中，DOM 尚未创建。

**解决方案**：
```python
# ❌ 错误
demo.launch(js="document.getElementById('my-text').value = 'hello'")

# ✅ 正确 - 使用 load 事件
with gr.Blocks() as demo:
    t = gr.Textbox(elem_id="my-text")
    demo.load(
        fn=None,
        outputs=t,
        js="() => 'hello'"  # 此时组件 DOM 已就绪
    )
```

### Q2: `dispatch("loaded")` 和 `dispatch_load_events()` 有什么区别？

| 特性 | `dispatch("loaded")` | `dispatch_load_events()` |
|------|---------------------|-------------------------|
| 触发者 | Index.svelte | Blocks.svelte onMount |
| 触发时机 | head 注入完成后 | 所有组件注册完成后 |
| 组件 DOM | ❌ 未渲染 | ✅ 已挂载 |
| 作用对象 | 父级 `<gradio-app>` 容器 | 所有带 `load` 事件的组件 |
| 能否触发 Python 代码 | ❌ 不能 | ✅ 能 |
| 能否访问组件值 | ❌ 不能 | ✅ 能 |

### Q3: `head` 中的内联 `<script>` 和 `launch(js=...)` 有什么区别？

| 特性 | `head="<script>...</script>"` | `launch(js="...")` |
|------|------------------------------|-------------------|
| 执行时机 | `dispatch("loaded")` 之前 | `dispatch("loaded")` 之后 |
| 顺序保证 | 与其他 head 资源按序执行 | 保证在 head 之后执行 |
| 适用场景 | 第三方库引入、全局 CSS、早于所有代码的配置 | 初始化全局函数、挂载 window 变量 |

### Q4: 非活动 tab 中的组件会触发 load 事件吗？

**不会**。非活动 tab / 折叠 accordion 中的组件在布局树遍历阶段会被 `_untrack()` 从 `components_to_register` Set 中移除，因此不会计入 `app_tree.ready` 的 resolve 条件。当 tab 切换到活动状态时，组件才会被渲染并调用 `register_component()`，此时也会触发其 `load` 事件。

### Q5: 自定义 JS 中如何调用另一个事件的 Python 函数？

在全局 `js` 或事件级 `js` 中，可以通过组件实例的 API：
```javascript
// 在 load 事件或用户交互事件中
const button = document.querySelector("[data-testid='button']");
button.click();  // 触发按钮点击事件，会执行其绑定的 Python 函数
```

或者通过 `dep_manager` 直接调度（需在 Svelte 上下文内）。

---

## 8. 关键文件索引

| 功能模块 | 文件 | 关键代码位置 |
|---------|------|------------|
| 全局 JS/Head 注入配置 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py) | `launch()` L2602+ / `get_config_file()` L2381+ |
| 事件函数配置 | [block_function.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/block_function.py) | `get_config()` L138+ |
| 配置 HTTP API | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/routes.py) | `/config` 路由 L954+ |
| SPA 入口注入时序 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | onMount L306+ / 注入 JS L372+ / dispatch("loaded") L364 |
| Head 注入实现 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | `add_custom_html_head()` L172+ |
| 依赖管理与运行 | [dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts) | `Dependency` L25+ / `DependencyManager` L194+ |
| 前端函数编译 | [dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts) | `process_frontend_fn()` L935+ |
| load 事件触发 | [Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte) | onMount L442+ |
| AppTree 组件就绪 | [init.svelte.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts) | `register_component()` L209+ / ready_resolve L234 |
| 组件基类注册 | [utils.svelte.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/utils/src/utils.svelte.ts) | Gradio 基类 constructor L430+ |
| 事件派发器 | [Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte) | `gradio_event_dispatcher()` L106+ |
| Python→JS 转译 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py) | `transpile_to_js()` L2438+ |
