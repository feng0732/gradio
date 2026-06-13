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

## 4. 前端执行时机与生命周期

### 4.1 确定性顺序 vs 条件性顺序

[Index.svelte#L306-L427](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L306-L427) 的 onMount 是一个 `async` 函数，内部通过 `await` 划分出**同步段**。

- **确定性**：同一同步段内的语句，或由 `await` 先后分隔的同步段之间，先后关系由代码结构硬性保证
- **条件性**：Svelte 响应式语句（`$:`）的触发时机、动态 `import()` 的解析时机取决于运行时调度

下文严格区分二者，并给出每条结论的代码论据。

### 4.2 确定性执行顺序（由代码结构保证）

以下时序来源于 Index.svelte onMount 函数体，每一步之间的先后由 `await` 或同步顺序**硬性保证**：

```
┌─────────────────────────────────────────────────────────────────┐
│ 确定性时序（onMount async 函数体内，按代码行顺序）              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① config = app.get_url_config()                               │
│     [L342] ← config 赋值，触发 $: 响应式依赖变化               │
│                                                                 │
│  ② if (app.config?.i18n_translations) {                        │
│         await setupi18n(...)                                    │
│     }                                                           │
│     [L345-L348] ← 条件性 await（见 4.3）                       │
│                                                                 │
│  ③ await mount_custom_css(config.css)                           │
│     [L359] ← CSS 注入，await 阻塞直到主题 CSS 加载完成          │
│                                                                 │
│  ④ await add_custom_html_head(config.head)                      │
│     [L360] ← L1: Head 注入，await 阻塞直到所有 head 资源加载   │
│     内联 <script> 在 appendChild 时同步执行                     │
│     外部 <script src> 通过 async=false 保序加载                 │
│     [Index.svelte#L172-L226]                                    │
│                                                                 │
│  ⑤ css_ready = true                                             │
│     [L361] ← 同步赋值，使模板条件 config && Blocks && css_ready │
│     的 css_ready 条件满足                                       │
│                                                                 │
│  ⑥ dispatch("loaded")                                           │
│     [L364] ← 同步调用，Svelte 组件事件，通知父级 <gradio-app>  │
│                                                                 │
│  ⑦ if (config.js) {                                             │
│         document.createElement("script")                        │
│         script.textContent = config.js                          │
│         document.head.appendChild(script)                       │
│     }                                                           │
│     [L372-L380] ← L2: 全局 JS，同步阻塞执行                    │
│                                                                 │
│  ⑧ onMount 函数体结束，控制权交还事件循环                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

确定性结论：
  ④ Head 注入  严格先于  ⑥ dispatch("loaded")  严格先于  ⑦ 全局 JS
  ⑤ css_ready=true  先于  ⑥ dispatch("loaded")  先于  ⑦ 全局 JS
  ⑦ 全局 JS  先于  ⑧ onMount 结束（即先于任何微任务刷新）
```

#### 为什么 ⑦ 全局 JS **一定**先于 Blocks 组件渲染？

关键论据来自三段代码的协作：

**论据 1：`load_demo()` 是 fire-and-forget**

[Index.svelte#L449-L452](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L449-L452)

```javascript
function load_demo(): void {
    if (config.auth_required) get_login();
    else get_blocks();  // ← 不 await！异步 import，立即返回
}
```

`get_blocks()` 内部的 `await import(...)` 是动态导入，其 Promise 解析（resolve）**始终发生在微任务队列**，不可能在当前同步段中完成。

**论据 2：L361 到 onMount 结束之间无 `await`**

从 `css_ready = true`（L361）到 onMount 结束（L427），代码全部同步执行，包括 `dispatch("loaded")`（L364）和 `config.js` 注入（L372）。JS 引擎不会在此同步段期间处理微任务队列——这是 ECMAScript 规范对 `async function` 执行语义的保证：**在两个 `await` 之间的代码是原子的**。

**论据 3：模板渲染需要 Svelte 刷新**

即使 `Blocks` 变量在之前的 `await` yield 期间被赋值（动态 import 解析），`<Blocks>` 组件的渲染也需要 Svelte 的响应式刷新，而刷新本身由 `schedule_update()` 调度，使用 `Promise.resolve().then(flush)` 排入微任务队列。在 L361-L427 同步段内，微任务队列不会被处理。

因此：**`config.js` 注入（⑦）必定在 Blocks 组件首次渲染之前完成**。这是确定性结论。

#### 为什么 ⑦ 全局 JS **不一定**先于 `load_demo()` 调用？

`load_demo()` 由 Svelte 响应式语句触发：

```javascript
$: config && (eager || $intersecting[_id]) && load_demo();
//  [Index.svelte#L436]
```

`config` 在 ① L342 被赋值后，Svelte 标记该响应式依赖为脏，排入刷新队列。刷新在下一个微任务执行。在 onMount 中的 `await` yield 点（② 或 ③），微任务队列有机会被处理，此时 `load_demo()` 被调用。

因此：**`load_demo()` 的调用通常早于 ⑦ 全局 JS**，但它启动的 `import()` 的 resolve 一定晚于 ⑦。

### 4.3 条件性时序（依赖运行时状态）

以下环节的触发时机受条件控制，不能硬性断言与确定性步骤的相对顺序：

#### 条件 1：`load_demo()` 何时被调用

```javascript
$: config && (eager || $intersecting[_id]) && load_demo();
//  [Index.svelte#L436]
```

| 条件 | `load_demo()` 触发时机 | 说明 |
|------|----------------------|------|
| `eager=true` | `config` 赋值后第一个 `await` yield 时 | 通常在 ② `await setupi18n()` 或 ③ `await mount_custom_css()` 处 Svelte 刷新 |
| `eager=false`，元素在视口内 | 同上（可能稍晚） | IntersectionObserver 在 [L491](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L491) 注册后可能尚未检测到 |
| `eager=false`，元素不在视口内 | 元素滚入视口时，可能远晚于 ⑦ | 嵌入场景下宿主页面可能未将 gradio-app 滚入视口 |

`eager` 的值由 [main.ts#L115](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/main.ts#L115) 决定：

```javascript
eager: this.eager === "true" ? true : false,
// 默认为 false（无 eager attribute 时）
```

**Svelte `$:` 响应式语句的调度机制**：
- `config` 在 L342 被赋值 → Svelte 标记该响应式依赖为脏
- Svelte 不立即重新执行 `$:` 语句，而是将其排入刷新队列
- 刷新在**下一个微任务**执行（`Promise.resolve().then(flush)`）
- 在 `async onMount` 中，每次 `await` 恢复前，微任务队列有机会被处理
- 因此 `load_demo()` 最早在 `config` 赋值后的**第一个 await 恢复点**被调用

**结论**：`load_demo()` 的调用时机取决于 `eager` 和 IntersectionObserver 状态。通常早于 ⑦ 全局 JS，但不可硬性保证。

#### 条件 2：`Blocks` 变量何时被赋值

```javascript
async function get_blocks(): Promise<void> {
    Blocks = (await import("@gradio/core/blocks")).default;
    //  [Index.svelte#L442-L444]
}
```

`await import(...)` 解析时机取决于模块加载状态：

| 场景 | `Blocks` 赋值时机 | 相对于 ⑦ `config.js` 的位置 |
|------|------------------|-------------------------|
| 模块未缓存，需要网络请求 | 网络请求完成后（数百 ms） | 远晚于 ⑦ |
| 模块已缓存（Vite 预打包） | 微任务队列解析 | 仍晚于 ⑦（见下方论证） |
| 开发模式 HMR 热替换 | 取决于 Vite dev server | 不确定，但通常晚于 ⑦ |

**为什么即使模块已缓存，`Blocks` 也晚于 `config.js` 赋值？**

动态 `import()` 返回一个 Promise。即使模块已在 Vite 模块图中缓存，Promise 的 `.then()` 回调仍然在**微任务队列**中执行，而不是同步执行。而 `config.js` 的 `document.head.appendChild(script)` 是同步操作（内联 `textContent` 的 `<script>` 在 `appendChild` 时立即执行），发生在 L361-L427 的同步段内。

由 ECMAScript 规范保证：**正在执行的同步代码不会被微任务中断**。因此 `import()` 的 resolve 不可能在 L361-L427 同步段执行期间被处理。

```
load_demo() 调用（在某次 await yield 期间）
  │
  ├─ get_blocks() 开始 → import(...) 发起
  │
  │  ... onMount 继续执行 ...
  │
  ├─ ⑦ config.js 同步执行 ← 此时 Blocks 尚未赋值
  │
  ├─ ⑧ onMount 结束
  │
  └─ 微任务队列：Svelte flush → Blocks 赋值（import resolve）→ 下一次 Svelte flush
     → <Blocks> 组件渲染
```

**结论**：`Blocks` 赋值**确定性晚于** ⑦ `config.js`。这是由 JavaScript 事件循环模型保证的。

#### 条件 3：`<Blocks>` 组件何时渲染

模板条件 ([Index.svelte#L595](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L595))：

```svelte
{:else if config && Blocks && css_ready}
    <Blocks ... bind:ready bind:render_complete ... />
```

三个条件全部满足时渲染：

| 条件 | 赋值时机 | 确定性 |
|------|---------|--------|
| `config` | ① L342 | 远早于其他两个条件 |
| `css_ready` | ⑤ L361（onMount 同步段内） | 早于 onMount 结束 |
| `Blocks` | 动态 import resolve 后 | 最晚，取决于网络/缓存 |

三者中 `Blocks` 是**最后满足的条件**，因此 **`<Blocks>` 渲染时刻 ≈ `Blocks` 赋值时刻 + Svelte 刷新**。

**结合条件 2 的结论**：`<Blocks>` 渲染**确定性晚于** ⑦ `config.js`。

#### 条件 4：`app_tree.ready` 何时 resolve

[init.svelte.ts#L232-L235](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L232-L235)

```javascript
if (this.components_to_register.size === 0 && !this.resolved) {
    this.resolved = true;
    this.ready_resolve();
}
```

触发条件：
- AppTree 构造时先将所有 `visible=true` 的组件 ID 加入 `components_to_register` Set
  [init.svelte.ts#L123-L125](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L123-L125)
- 不可见组件（`visible=false`、折叠的 accordion 子项、非活动 tab 子项）会被 `_untrack()` 排除
  [init.svelte.ts#L818-L871](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L818-L871)
- 每个可见组件在其 Gradio 基类 constructor 中调用 `register_component()` 时从 Set 中删除
  [utils.svelte.ts#L430-L435](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/utils/src/utils.svelte.ts#L430-L435)

**条件性**：非活动 tab / 折叠 accordion 中的组件不会触发 ready，直到它们被展开时才会注册。如果应用大量使用 tab 布局，`app_tree.ready` 可能延迟很久才 resolve。

### 4.4 完整时序图（含确定性/条件性标注）

```
Browser 加载页面
  │
  ├─ 1. SPA main.ts 启动，挂载 <gradio-app> WebComponent
  │     [main.ts]
  │
  ├─ 2. Index.svelte onMount() 开始
  │     [Index.svelte#L306]
  │     │
  │     ├─ 2a. await Client.connect(api_url)
  │     │     [L328] ← await，阻塞直到 /config 响应
  │     │
  │     ├─ 2b. config = app.get_url_config()
  │     │     [L342] ← 【确定性】config 赋值
  │     │     → Svelte 标记 $: 响应式语句为脏
  │     │
  │     ├─ 2c. if (i18n_translations) await setupi18n(...)
  │     │     [L345-L348] ← 【条件性 await】
  │     │     → 若执行了 await，则 Svelte 在此刷新
  │     │     → 若 eager=true，load_demo() 在此处被调用
  │     │
  │     ├─ 2d. await mount_custom_css(config.css)
  │     │     [L359] ← 【确定性 await】CSS 注入
  │     │     → 若 load_demo() 尚未调用且 eager=true，
  │     │       则 Svelte 在此刷新时调用 load_demo()
  │     │
  │     ├─ 2e. await add_custom_html_head(config.head)
  │     │     [L360] ← 【确定性】L1: Head 注入
  │     │     → 内联 <script> 同步执行，外部 <script src> 保序加载
  │     │
  │     │  ────────── 以下为同步段，无 await，不可中断 ──────────
  │     │
  │     ├─ 2f. css_ready = true
  │     │     [L361] ← 【确定性】
  │     │
  │     ├─ 2g. dispatch("loaded")
  │     │     [L364] ← 【确定性】页面级信号
  │     │
  │     ├─ 2h. if (config.js) { script 注入 }
  │     │     [L372-L380] ← 【确定性】L2: 全局 JS 同步执行
  │     │
  │     │  ────────── 同步段结束 ──────────
  │     │
  │     └─ 2i. onMount 返回
  │           [L427]
  │
  ├─ 3. 微任务队列处理
  │     │
  │     ├─ 3a. Svelte 响应式刷新
  │     │     → 若 Blocks 已赋值：模板渲染 <Blocks> 组件
  │     │     → 若 Blocks 未赋值：等待
  │     │
  │     └─ 3b. 动态 import 解析（若模块已加载）
  │           → Blocks = (await import(...)).default
  │           → 触发下一次 Svelte 刷新 → <Blocks> 渲染
  │
  └─ 4. Blocks.svelte 组件挂载 【确定性晚于 2h】
        │
        ├─ 4a. 构造 AppTree
        │     [init.svelte.ts#L96-L144]
        │     ├─ this.ready = new Promise(resolve => ready_resolve = resolve)
        │     ├─ components_to_register = Set(所有 visible=true 组件的 ID)
        │     └─ 递归 traverse 布局树
        │
        ├─ 4b. 构造 DependencyManager
        │     [dependency.ts#L194+]
        │     └─ new Dependency(dep_config) → 编译前端函数
        │
        ├─ 4c. Svelte 渲染 <MountComponents>
        │     [Blocks.svelte#L479]
        │     └─ 每个组件的 Gradio 基类 constructor：
        │        this.register_component(id, this.set_data, this.get_data)
        │        [utils.svelte.ts#L430-L435]
        │        └─ app_tree.register_component(id, ...)
        │           ├─ components_to_register.delete(id)
        │           └─ if (size === 0) ready_resolve()
        │              [init.svelte.ts#L232-L235]
        │
        └─ 4d. Blocks.svelte onMount
              [Blocks.svelte#L422-L456]
              └─ app_tree.ready.then(() => {
                  ready = true;
                  dep_manager.dispatch_load_events();
              })
              [Blocks.svelte#L442-L445]
```

### 4.5 关键时间点对比表

| 事件 | 代码位置 | 组件 DOM | 能否操作组件 | 顺序确定性 | 适用场景 |
|------|---------|---------|-------------|-----------|---------|
| **head 注入** | [L360](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L360) | ❌ | ❌ | ✅ 确定先于 dispatch("loaded") | 第三方库、CSS、全局配置 |
| **`dispatch("loaded")`** | [L364](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L364) | ❌ | ❌ | ✅ 确定先于 config.js | 父级容器页面状态信号 |
| **全局 `js` 执行** | [L372-L380](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L372-L380) | ❌ | ❌ | ✅ 确定先于 Blocks 渲染 | 全局函数、window 变量 |
| **`load_demo()` 调用** | [L436](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L436) | ❌ | ❌ | ⚠️ 条件性（见 4.3 条件 1） | 触发 Blocks 动态加载 |
| **`Blocks` 赋值** | [L443](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L443) | ❌ | ❌ | ✅ 确定晚于 config.js（见 4.3 条件 2） | 解除模板渲染条件 |
| **`<Blocks>` 渲染** | [L595](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte#L595) | ✅ 正在渲染 | ⚠️ 部分 | ✅ 确定晚于 config.js | 组件 DOM 创建 |
| **`app_tree.ready`** | [init.svelte.ts#L234](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L234) | ✅ | ✅ | ⚠️ 条件性（tab/accordion） | 组件注册全部完成 |
| **`dispatch_load_events()`** | [Blocks.svelte#L444](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte#L444) | ✅ | ✅ | ✅ 确定晚于 app_tree.ready | 组件 load 事件触发 |

### 4.6 全局脚本 vs load_demo vs 组件挂载：确定性总结

```
                     确定性关系                    条件性关系
                ──────────────────────      ──────────────────────
                head 注入 < dispatch("loaded")    load_demo() 调用时机
                dispatch("loaded") < config.js    （取决于 eager/IObserver）
                config.js < Blocks 赋值
                config.js < <Blocks> 渲染
                <Blocks> 渲染 < app_tree.ready     app_tree.ready 延迟
                app_tree.ready < load 事件         （取决于 tab/accordion）
```

**用时间轴表示（eager=true 常见场景）**：

```
T0  config = ...          ← 【确定性】L342
T1  await setupi18n()     ← 【条件性 await】
T2  load_demo() 被调用    ← 【条件性】Svelte 刷新时
T3  get_blocks() 开始     ← import() 发起，异步
T4  await mount_custom_css()  ← 【确定性】
T5  await add_custom_html_head()  ← 【确定性】
T6  css_ready = true      ← 【确定性】
T7  dispatch("loaded")    ← 【确定性】
T8  config.js 执行        ← 【确定性】同步阻塞
    ── onMount 同步段结束 ──
T9  微任务：Svelte flush  ← Blocks 可能在此赋值
T10 微任务：import resolve ← Blocks 赋值（若 T9 未完成）
T11 <Blocks> 组件渲染     ← 【确定性】晚于 T8
T12 组件 register_component() 逐个完成
T13 app_tree.ready resolve     ← 【条件性】
T14 dispatch_load_events()     ← 【确定性】晚于 T13
```

### 4.7 L1: `head` 注入执行细节

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

关键点：
- `await` 会等待所有外部资源（如 `<script src>`、`<link rel="stylesheet">`）加载完成
- 内联 `<script>` 会在 `appendChild` 时**同步阻塞**执行
- `async=false` 保证多个 SCRIPT 按文档顺序执行
- 执行时机早于 `dispatch("loaded")`，早于全局 `js`（确定性）

### 4.8 `dispatch("loaded")`：页面级信号（⚠️ 不是 DOM ready！）

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

重要澄清：
- `dispatch("loaded")` 是 Svelte 组件事件，只通知父级 `<gradio-app>` 容器
- 触发时：✅ CSS 就绪 / ✅ Head 资源加载完成 / ❌ 组件 DOM **未** 渲染 / ❌ 全局 js **未** 执行
- **与 `load` 事件无关**，是页面级容器状态信号
- 触发后才执行全局 `js` 注入（确定性，见 4.2 ⑥→⑦）

### 4.9 L2: 全局 `js` 注入执行细节

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

关键点：
- **执行顺序**：`head` → `dispatch("loaded")` → **全局 `js`**（确定性）
- 此时：✅ `window`/`document` 可用 / ❌ Blocks.svelte 组件可能仍在导入 / ❌ 组件 DOM **未** 渲染
- 如果需要操作组件 DOM，**必须**用 `load` 事件，不能用全局 `js` 直接查询 DOM
- 作用于全局作用域，可以定义 `window.myFn = function() {...}` 供后续事件级 JS 调用

### 4.10 `app_tree.ready`：组件就绪的内部 Promise

[init.svelte.ts#L232-L235](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts#L232-L235)

```javascript
if (this.components_to_register.size === 0 && !this.resolved) {
    this.resolved = true;
    this.ready_resolve();   // ← 所有组件注册完成
}
```

注意：非活动 tab / 折叠 accordion 中的组件不会触发 ready，直到它们被展开时才会注册。

### 4.11 `load` 事件：组件级初始化事件

触发时机：[Blocks.svelte#L442-L445](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte#L442-L445)

```javascript
app_tree.ready.then(() => {
    ready = true;
    dep_manager.dispatch_load_events();  // 所有可见组件 DOM 已挂载
});
```

两种 load 事件来源：

1. **显式声明**：`demo.load(fn=..., inputs=..., outputs=...)` 或 `component.load(fn)`
2. **自动附加**：[blocks.py#L989-L1011](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py#L989-L1011) `attach_load_events()` 为有初始值需要后端计算的组件（如 `gr.Video(value=...)`）自动附加 `load` 事件

`load` 事件的 JS 能力：
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
        frontend: dep_config.js
            ? process_frontend_fn(
                    dep_config.js,
                    dep_config.backend_fn,
                    dep_config.inputs.length,
                    dep_config.outputs.length
                )
            : undefined,

        backend: dep_config.backend_fn,

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
        return { type: "data", data: _data_payload };
    }
    return { type: "void", data: null };
}
```

四种协同模式对比：

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
```

### 5.5 DependencyManager：事件调度中枢

[dependency.ts#L194+](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts#L194)

协同控制参数：

| 参数 | 含义 | 生效位置 |
|------|------|---------|
| `trigger_after` | 链式事件 ID (.then/.success/.failure) | dep_manager 完成主 dep 后触发 |
| `trigger_mode` | `"once"` / `"multiple"` / `"always_last"` | dispatch 时决定是否排队或跳过 |
| `cancels` | 触发时需要取消的其他事件 ID | dispatch 首步执行 this.cancel() |
| `collects_event_data` | 是否收集前端 Event 对象到 Python | 序列化时通过 event_data 传递 |

---

## 6. 完整协同示例追踪

```python
with gr.Blocks() as demo:
    inp = gr.Textbox()
    out = gr.Textbox()
    btn = gr.Button("Run")

    btn.click(
        fn=lambda x: x.upper(),
        inputs=inp,
        outputs=out,
        js="(x) => x.trim()"
    )

demo.launch(
    head="<script>window.LIB_VERSION='1.0'</script>",
    js="console.log('global init', window.LIB_VERSION);"
)
```

执行追踪（含确定性/条件性标注）：

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
   - T1: `config = app.get_url_config()` 【确定性】
   - T2: `await setupi18n()` 或跳过 【条件性】→ Svelte 刷新，`load_demo()` 可能在此调用
   - T3: `await mount_custom_css(config.css)` 【确定性】→ 若 T2 未触发 load_demo，此处触发
   - T4: `await add_custom_html_head(config.head)` 【确定性】→ 执行 `<script>window.LIB_VERSION='1.0'</script>`
   - ── 以下同步段，不可中断 ──
   - T5: `css_ready = true` 【确定性】
   - T6: `dispatch("loaded")` 【确定性】通知父容器
   - T7: `config.js` 注入 → 执行 `console.log('global init', '1.0')` → 输出 `global init 1.0` 【确定性】
   - ── 同步段结束 ──
   - T8: onMount 返回，微任务队列开始处理
   - T9: `Blocks` 动态 import 解析 → `Blocks` 变量赋值 【确定性晚于 T7】
   - T10: Svelte 刷新 → `<Blocks>` 组件渲染 【确定性晚于 T7】
   - T11: 各组件 constructor 调用 `register_component()`，Set 逐个清空
   - T12: 最后一个组件注册 → `app_tree.ready_resolve()` 【条件性：tab/accordion】
   - T13: `dep_manager.dispatch_load_events()` → 触发所有 load 事件 【确定性晚于 T12】

3. **用户点击按钮**：
   - `gradio_event_dispatcher(3, "click", null)` → `dep_manager.dispatch()`

4. **Dependency.run() 执行**：
   - Step1: frontend() → `"  HELLO  " → "HELLO"`
   - Step2: backend → client.submit(0, ["HELLO"])
   - Step3: Python `/call/0` → `lambda x: x.upper()` → `"HELLO"`

---

## 7. 常见问题与最佳实践

### Q1: 全局 `js` 中为什么 `document.getElementById("component-id")` 返回 null？

**原因**：全局 `js` 在 T7 执行，Blocks.svelte 组件在 T10 才开始渲染。DOM 尚未创建。

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
| 触发者 | Index.svelte onMount L364 | Blocks.svelte onMount L444 |
| 触发时机 | Head 注入完成后，**仍在 onMount 同步段内** | 所有组件 `register_component()` 完成后 |
| 组件 DOM | ❌ 未渲染（Blocks 模块可能还在加载） | ✅ 已挂载 |
| 作用对象 | 父级 `<gradio-app>` 容器 | 所有带 `load` 事件的组件 |
| 能否触发 Python 代码 | ❌ 不能 | ✅ 能 |
| 与全局 js 的顺序 | ✅ 确定先于 `config.js` | ✅ 确定晚于 `config.js` |

### Q3: `head` 中的内联 `<script>` 和 `launch(js=...)` 有什么区别？

| 特性 | `head="<script>...</script>"` | `launch(js="...")` |
|------|------------------------------|-------------------|
| 执行时机 | `dispatch("loaded")` 之前 | `dispatch("loaded")` 之后 |
| 顺序保证 | 与其他 head 资源按序执行 | 保证在 head 之后执行 |
| 适用场景 | 第三方库引入、全局 CSS、早于所有代码的配置 | 初始化全局函数、挂载 window 变量 |

### Q4: 非活动 tab 中的组件会触发 load 事件吗？

**不会**。非活动 tab / 折叠 accordion 中的组件在布局树遍历阶段会被 `_untrack()` 从 `components_to_register` Set 中移除，因此不会计入 `app_tree.ready` 的 resolve 条件。当 tab 切换到活动状态时，组件才会被渲染并调用 `register_component()`，此时也会触发其 `load` 事件。

### Q5: `load_demo()` 一定在全局 `js` 之前触发吗？

**通常如此，但不保证**。`load_demo()` 由 Svelte 响应式语句在 `await` yield 点触发，通常早于 `config.js`（L372）。但如果 `eager=false` 且 IntersectionObserver 尚未检测到元素，`load_demo()` 可能延迟到元素滚入视口时才触发，此时可能远晚于 `config.js`。

然而，无论 `load_demo()` 何时触发，它启动的 `import()` 的 resolve **确定性晚于** `config.js`，因此 Blocks 组件渲染**确定性晚于** `config.js`。

### Q6: 自定义 JS 中如何调用另一个事件的 Python 函数？

在全局 `js` 或事件级 `js` 中，可以通过组件实例的 API：
```javascript
const button = document.querySelector("[data-testid='button']");
button.click();
```

或者通过 `dep_manager` 直接调度（需在 Svelte 上下文内）。

---

## 8. 关键文件索引

| 功能模块 | 文件 | 关键代码位置 |
|---------|------|------------|
| 全局 JS/Head 注入配置 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py) | `launch()` L2602+ / `get_config_file()` L2381+ |
| 事件函数配置 | [block_function.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/block_function.py) | `get_config()` L138+ |
| 配置 HTTP API | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/routes.py) | `/config` 路由 L954+ |
| SPA 入口注入时序 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | onMount L306+ / dispatch("loaded") L364 / 注入 JS L372+ |
| Head 注入实现 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | `add_custom_html_head()` L172+ |
| 响应式 load_demo 触发 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | `$: load_demo()` L436 |
| 动态 import Blocks | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | `get_blocks()` L442-L444 |
| Blocks 渲染条件 | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/Index.svelte) | `{#if config && Blocks && css_ready}` L595 |
| eager 属性传递 | [main.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/spa/src/main.ts) | L115 / L183 |
| 依赖管理与运行 | [dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts) | `Dependency` L25+ / `DependencyManager` L194+ |
| 前端函数编译 | [dependency.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/dependency.ts) | `process_frontend_fn()` L935+ |
| load 事件触发 | [Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/Blocks.svelte) | onMount L442+ |
| AppTree 组件就绪 | [init.svelte.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/core/src/init.svelte.ts) | `register_component()` L209+ / ready_resolve L234 |
| 组件基类注册 | [utils.svelte.ts](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/js/utils/src/utils.svelte.ts) | Gradio 基类 constructor L430+ |
| Python→JS 转译 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/257-gradio/gradio/blocks.py) | `transpile_to_js()` L2438+ |
