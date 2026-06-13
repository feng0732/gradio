# Gradio 自定义组件打包链路全解析

本文档从源码层面梳理 Gradio 自定义组件从创建、构建到前端集成的完整链路，帮助理解组件模板、构建命令和前端加载是如何串联起来的。

## 目录

- [整体架构概览](#整体架构概览)
- [一、组件模板与创建](#一组件模板与创建)
- [二、构建命令与打包流程](#二构建命令与打包流程)
- [三、前端集成与加载](#三前端集成与加载)
- [四、后端路由与静态文件服务](#四后端路由与静态文件服务)
- [五、开发模式](#五开发模式)
- [六、回退逻辑详解](#六回退逻辑详解)
- [七、组件未渲染原因分析](#七组件未渲染原因分析)
- [八、浏览器与服务端集成路径差异](#八浏览器与服务端集成路径差异)

---

## 整体架构概览

Gradio 自定义组件的完整生命周期包含以下阶段：

```
创建 (gradio cc create)
    ↓
开发 (gradio cc dev) ← 热更新循环
    ↓
构建 (gradio cc build)
    ├─ 前端构建 (Vite + @gradio/preview)
    │   └─ 输出到 templates/ 目录
    └─ Python 打包 (python -m build)
        └─ 生成 wheel 包
    ↓
运行时加载
    ├─ 内置组件: 构建时静态注入 (virtual:component-loader)
    └─ 自定义组件: 运行时动态加载 (HTTP + import())
```

---

## 一、组件模板与创建

### 1.1 创建命令入口

命令：`gradio cc create <ComponentName>`

入口文件：[create.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/create.py)

核心函数：`_create()`

### 1.2 模板来源

组件模板有两个来源：

1. **内置组件模板**：基于 Gradio 自带的组件（如 `Fallback`、`Textbox`、`Image` 等）复制代码
2. **空白模板**：使用 `Fallback` 组件作为基础模板

模板映射配置在 [_create_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/_create_utils.py#L72-L245) 的 `OVERRIDES` 字典中，定义了每个组件对应的 Python 文件名、JS 目录名和 demo 代码。

### 1.3 创建流程

创建流程由 `_create()` 函数驱动，主要步骤：

1. **创建后端代码** (`_create_backend()`)
   - 从 `gradio.components` 或 `gradio.layouts` 中找到模板组件类
   - 复制模板组件的 Python 源码到 `backend/<package_name>/` 目录
   - 替换类名为用户指定的名称
   - 生成 `__init__.py`、`pyproject.toml`、`demo/app.py` 等文件

2. **创建前端代码** (`_create_frontend()`)
   - 前端源码来源：优先从本地 `gradio/_frontend_code/<version>/` 目录复制，否则从 HuggingFace Hub 下载
   - 复制对应组件的 Svelte 源码到 `frontend/` 目录
   - 修改 `package.json` 的包名和依赖版本
   - 复制配置文件：[gradio.config.js](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/files/gradio.config.js)、[tsconfig.json](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/files/tsconfig.json)

3. **可选：安装组件** (`_install_command()`)
   - 执行 `pip install -e .` 进行开发模式安装
   - 执行 `npm install` 安装前端依赖

### 1.4 项目结构

创建后的组件目录结构：

```
my-component/
├── backend/
│   └── gradio_mycomponent/
│       ├── __init__.py
│       ├── mycomponent.py       # 组件 Python 源码
│       └── templates/           # 构建后生成的前端产物目录
│           ├── component/
│           │   ├── index.js
│           │   └── style.css
│           └── example/
│               ├── index.js
│               └── style.css
├── frontend/
│   ├── Index.svelte             # 组件主文件
│   ├── Example.svelte           # 示例组件
│   ├── package.json
│   ├── gradio.config.js         # Gradio 构建配置
│   └── tsconfig.json
├── demo/
│   ├── app.py                   # 演示应用
│   └── __init__.py
├── pyproject.toml               # Python 包配置
└── README.md
```

### 1.5 pyproject.toml 模板

模板文件：[pyproject_.toml](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/files/pyproject_.toml)

关键配置项：

```toml
[project]
name = "<<name>>"                    # 包名，如 gradio-mycomponent
keywords = ["gradio-custom-component", ...]  # 标记为自定义组件

[tool.hatch.build]
artifacts = ["/backend/<<name>>/templates", "*.pyi"]  # 构建产物目录

[tool.hatch.build.targets.wheel]
packages = ["/backend/<<name>>"]      # 包源码位置
```

`keywords` 中的 `gradio-custom-component` 是识别自定义组件的重要标记，[examine.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/examine.py#L24-L27) 会检查这个关键字来判断是否为自定义组件。

---

## 二、构建命令与打包流程

### 2.1 构建命令入口

命令：`gradio cc build`

入口文件：[build.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/build.py)

核心函数：`_build()`

### 2.2 构建总流程

构建分为两大阶段：**前端构建** 和 **Python 包构建**。

```
gradio cc build
    ├─ 1. 检查 pyproject.toml
    ├─ 2. 检查组件是否已安装
    ├─ 3. 可选：升级版本号 (--bump-version)
    ├─ 4. 可选：生成文档 (--generate-docs)
    ├─ 5. 前端构建 (--build-frontend)
    │   └─ 调用 node @gradio/preview --mode build
    │       └─ Vite 打包 → templates/component/ 和 templates/example/
    └─ 6. Python 包构建
        └─ python -m build → dist/*.whl
```

### 2.3 前端构建详解

#### 2.3.1 调用链

Python 侧通过 `subprocess` 调用 Node.js：

```python
# build.py 第 145-156 行
node_cmds = [
    node,
    gradio_node_path,           # @gradio/preview 入口
    "--component-directory",
    component_directory,
    "--root",
    gradio_template_path,       # gradio/templates/frontend
    "--mode",
    "build",
    "--python-path",
    python_path,
]
```

`@gradio/preview` 包的入口是 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/index.ts)，根据 `--mode` 参数决定执行构建还是开发模式。

#### 2.3.2 构建核心逻辑

构建函数：[make_build()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/build.ts#L19-L138)

主要步骤：

1. **检查模块元数据** (`examine_module()`)
   - 调用 [examine.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/examine.py) Python 脚本
   - 获取组件的 `template_dir`、`frontend_dir`、`component_class_id`

2. **读取组件配置**
   - 从 `frontend/gradio.config.js` 读取自定义配置
   - 支持自定义 plugins、svelte preprocess、build target 等

3. **Vite 构建**
   - 每个组件构建两个变体：`component`（主组件）和 `example`（示例）
   - 入口文件是 `svelte_runtime_entry.js` + 组件的 `Index.svelte`
   - 使用 `es` 模块格式输出
   - 输出目录：`templates/component/` 和 `templates/example/`

#### 2.3.3 examine.py 的作用

[examine.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/examine.py) 是连接 Python 和 JavaScript 的桥梁脚本。

功能：
1. 读取 `pyproject.toml`，通过 `keywords` 识别是否为自定义组件
2. 导入组件模块，找出所有 `Component` 或 `BlockContext` 子类
3. 输出每个组件的元信息（用 `~|~|~|~` 分隔）：
   ```
   组件类名~|~|~|~模板目录路径~|~|~|~前端目录路径~|~|~|~组件类ID
   ```
4. **build 模式下**：自动更新 `pyproject.toml` 中的 `artifacts` 配置，将 `templates` 目录加入构建产物

#### 2.3.4 构建输出

构建产物输出到组件 Python 包的 `templates/` 目录下：

```
templates/
├── component/
│   ├── index.js                  # 主组件 ES 模块
│   ├── svelte_runtime_entry.js   # Svelte 运行时导出
│   └── style.css                 # 组件样式
└── example/
    ├── index.js
    ├── svelte_runtime_entry.js
    └── style.css
```

`svelte_runtime_entry.js` 的作用：导出 Svelte 的 `mount` 和 `unmount` 函数，用于运行时动态挂载组件。

### 2.4 Python 包构建

前端构建完成后，执行 `python -m build` 构建 Python 包。

关键配置：
- 包源码位于 `/backend/<package_name>/`
- 构建产物包含 `templates/` 目录（通过 `artifacts` 配置）
- 输出 wheel 文件到 `dist/` 目录

---

## 三、前端集成与加载

### 3.1 组件加载器机制

Gradio 使用 **虚拟模块** `virtual:component-loader` 来统一处理内置组件和自定义组件的加载。

#### 3.1.1 虚拟模块的实现

虚拟模块由 Vite 插件提供，有两套实现：

**1. 内置组件加载器（主应用构建时）**

位置：[inject_component_loader()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/build/out/index.js#L271-L287) in `@self/build`

工作方式：
- 构建时扫描 `js/` 目录下所有组件包
- 生成 `component_map` 对象，包含所有内置组件的动态导入
- 组件通过 `component: () => import("@gradio/textbox")` 形式懒加载

**2. 自定义组件开发模式加载器**

位置：[make_gradio_plugin()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/plugins.ts#L96-L140) in `@gradio/preview`

工作方式：
- 开发模式下，将自定义组件信息注入到 `window.__GRADIO__CC__` 和 `window.__GRADIO__CC__RUNTIMES__`
- 组件加载器优先从这些全局变量中查找组件

#### 3.1.2 load_component 函数

核心加载函数：[load_component()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/build/out/component_loader.js#L8-L75)

加载策略（优先级从高到低）：

1. **缓存检查**：`request_map` 中是否已有请求
2. **开发模式自定义组件**：从 `window.__GRADIO__CC__` 查找
3. **内置组件**：从 `component_map` 查找
4. **动态 HTTP 加载**：通过 `/custom_component/{id}/...` 接口加载（生产模式自定义组件）
5. **降级处理**：加载失败时使用 `@gradio/fallback` 组件（仅 example 变体）

#### 3.1.3 动态 HTTP 加载

生产环境下，自定义组件通过 HTTP 动态加载：

```javascript
// component_loader.js 第 91-119 行
function get_component_with_css(api_url, id, variant) {
    const path = `${api_url}/custom_component/${id}/client/${variant}/index.js`;
    
    return [
        // 1. 加载样式 + 组件 JS
        Promise.all([
            load_css(`${api_url}/custom_component/${id}/client/${variant}/style.css`),
            import(path)  // 动态 ES 模块导入
        ]),
        // 2. 加载 Svelte 运行时
        import(`${api_url}/custom_component/${id}/client/${variant}/svelte_runtime_entry.js`)
    ];
}
```

### 3.2 组件挂载机制

自定义组件使用独立的 Svelte 运行时，通过 `mount` API 动态挂载。

挂载组件：[MountCustomComponent.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/core/src/MountCustomComponent.svelte)

```svelte
<script lang="ts">
    let { node, children, ...rest } = $props();
    
    let component = $derived(await node.component);
    let runtime = $derived((await node.runtime) as {
        mount: typeof import("svelte").mount;
        unmount: typeof import("svelte").unmount;
    });
    let el: HTMLElement = $state(null);
    
    $effect(() => {
        if (!el || !runtime || !component) return;
        
        const mounted = _runtime.mount(component.default, {
            target: el,
            props: {
                shared_props: _shared_props,
                props: _props,
                children
            }
        });
        
        return () => {
            _runtime.unmount(mounted);
        };
    });
</script>

<span bind:this={el}></span>
```

**为什么需要独立的 Svelte 运行时？**

自定义组件可能使用与主应用不同版本的 Svelte。为了避免版本冲突，每个自定义组件自带一份 Svelte 运行时（通过 `svelte_runtime_entry.js` 导出），使用 `mount`/`unmount` API 进行生命周期管理。

### 3.3 组件类 ID

每个组件类有一个唯一的 `component_class_id`，用于标识和查找组件。

生成方式：[get_component_class_id()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/blocks.py#L446-L454)

```python
@classmethod
def get_component_class_id(cls) -> str:
    module_path = inspect.getfile(cls)
    module_hash = hashlib.sha256(
        f"{cls.__name__}_{module_path}".encode()
    ).hexdigest()
    return module_hash
```

这个 ID 用于：
- 前端路由中标识自定义组件
- 组件加载时的查找 key
- 缓存 key

---

## 四、后端路由与静态文件服务

### 4.1 自定义组件路由

路由定义：[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/routes.py#L981-L1044)

```python
@router.get("/custom_component/{id}/{environment}/{type}/{file_name}")
def custom_component_path(
    id: str,              # component_class_id
    environment: Literal["client", "server"],  # 客户端/服务端渲染
    type: str,            # component / example / base
    file_name: str,       # index.js / style.css / svelte_runtime_entry.js
    req: fastapi.Request,
):
    # 1. 根据 component_class_id 查找组件类
    components = utils.get_all_components()
    location = next(
        (item for item in components if item.get_component_class_id() == id),
        None,
    )
    
    # 2. 拼接到 templates 目录的路径
    requested_path = utils.safe_join(
        location.TEMPLATE_DIR,
        UserProvidedPath(f"{type}/{file_name}"),
    )
    
    # 3. 返回文件，支持 ETag 缓存
    return FileResponse(path, headers=headers)
```

### 4.2 TEMPLATE_DIR 与 FRONTEND_DIR

每个组件类定义了两个路径属性：

```python
# base.py 第 226-227 行
TEMPLATE_DIR = DeveloperPath("./templates/")    # 构建产物目录
FRONTEND_DIR = "../../frontend/"                # 前端源码目录
```

- `TEMPLATE_DIR`：相对于 Python 模块文件的路径，存放构建后的前端产物
- `FRONTEND_DIR`：前端源码目录，开发模式下使用

---

## 五、开发模式

### 5.1 开发命令入口

命令：`gradio cc dev`

入口文件：[dev.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/dev.py)

### 5.2 开发模式架构

开发模式启动两个服务器：

```
┌─────────────────────────────────────────────────┐
│  @gradio/preview (Node.js)                     │
│  ├─ Vite Dev Server (前端热更新)                │
│  └─ 代理后端 API 请求                            │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│  Gradio Backend (Python)                        │
│  └─ demo/app.py (用户的演示应用)                 │
└─────────────────────────────────────────────────┘
```

### 5.3 开发模式工作流程

1. **启动 Python 后端**
   - 运行 `gradio demo/app.py --watch-dirs <component_dir>`
   - 监听组件目录变化，自动重载

2. **启动 Vite 开发服务器**
   - 由 [create_server()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/dev.ts#L39-L91) 创建
   - 支持 Svelte 热更新

3. **注入自定义组件**
   - 通过 `make_gradio_plugin` 将自定义组件信息注入到 `window.__GRADIO__CC__`
   - 组件加载器优先从全局变量加载，实现热更新

### 5.4 开发模式下的组件配置生成

[generate_imports()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/dev.ts#L137-L...) 函数负责生成开发模式下的组件导入映射：

1. 调用 `examine_module()` 找出所有自定义组件
2. 生成组件的动态导入代码
3. 通过虚拟模块 `virtual:cc-init` 注入到页面中

---

## 六、回退逻辑详解

Gradio 中的回退（Fallback）机制分为两类：**组件加载失败时的 fallback 组件** 和 **作为空白模板的 Fallback 组件**。两者同名但用途不同。

### 6.1 Fallback 组件本身

Fallback 是 Gradio 内置的一个特殊组件，用于：
1. 作为自定义组件的**空白模板**（创建组件时默认基于 Fallback）
2. 作为**组件加载失败时的降级显示**（仅 example 变体）

**Python 端实现**：[fallback.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/components/fallback.py)

```python
class Fallback(Component):
    EVENTS = [Events.change]

    def preprocess(self, payload):
        return payload

    def postprocess(self, value):
        return value

    def example_payload(self):
        return {"foo": "bar"}

    def example_value(self):
        return {"foo": "bar"}

    def api_info(self):
        return {"type": {}, "description": "any valid json"}
```

**前端 Svelte 实现**：[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/fallback/Index.svelte)

```svelte
<Block visible={gradio.shared.visible} ...>
    {#if gradio.shared.loading_status}
        <StatusTracker ... />
    {/if}
    <!-- 使用 JsonView 以 JSON 形式展示原始 value -->
    <JsonView json={gradio.props.value} />
</Block>
```

Fallback 组件的核心特点：
- 对数据不做任何转换，`preprocess`/`postprocess` 直接透传
- 前端使用 `JsonView` 以 JSON 格式渲染任意数据
- 支持 `change` 事件和加载状态显示

### 6.2 组件加载失败时的回退逻辑

核心回退代码位于 [component_loader.js](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/build/out/component_loader.js) 中。

#### 6.2.1 Example 变体回退

当**示例组件**（example variant）加载失败时，自动回退到 `@gradio/fallback/example`：

```javascript
// component_loader.js 第 55-63 行
export async function load_component(id, variant, api_url) {
    // ... 尝试各种加载方式 ...
    // 最终兜底：只有 example 变体支持 fallback
    if (variant === "example") {
        request_map[`${_id}-${variant}`] = import("@gradio/fallback/example");
        return [request_map[`${_id}-${variant}`], Promise.resolve(false)];
    }
    throw new Error(`Could not load component ${id} variant ${variant}`);
}
```

**注意**：主组件（`variant === "component"`）加载失败时**不会**使用 fallback，而是直接抛出错误。这意味着：
- `gr.Examples` 中展示的示例组件加载失败 → 使用 JSON 形式兜底显示
- 用户界面中的实际组件加载失败 → 页面报错，组件不可用

#### 6.2.2 SSR 模式下的强制回退

在**服务端渲染（SSR）**环境中，自定义组件会被强制回退：

```javascript
// component_loader.js 第 89-98 行
function get_component_type(_id, variant, api_url) {
    const environment = is_browser ? "client" : "server";

    if (environment === "server") {
        // Fall back to @gradio/fallback during SSR; the real component
        // will be loaded client-side.
        return [import("@gradio/fallback"), Promise.resolve(false)];
    }
    // ... 浏览器环境正常加载 ...
}
```

SSR 回退策略的设计意图：
1. **避免 Node.js 环境问题**：自定义组件的 JS 可能依赖浏览器 API，在 Node SSR 环境中无法运行
2. **两阶段渲染**：SSR 阶段先渲染 Fallback（JSON 占位），客户端水化（hydration）阶段再加载真实组件
3. **自定义组件 SSR 尚未支持**：路由代码中有注释 `// Uncomment when we support custom component SSR`，表明未来可能支持

### 6.3 回退触发场景总结

| 场景 | 回退组件 | 是否用户可见 | 说明 |
|------|---------|------------|------|
| `gradio cc create` 空白模板 | `gradio.components.Fallback` | 否 | 创建时基于其复制代码 |
| Example 组件加载失败 | `@gradio/fallback/example` | 是 | 示例区以 JSON 显示 |
| SSR 服务端渲染 | `@gradio/fallback` | 短暂可见 | 客户端水化后替换为真实组件 |
| 主组件加载失败 | 无回退 | 是（报错） | 直接抛出错误 |

---

## 七、组件未渲染原因分析

组件在界面上"看不见"可能由多个层面的原因造成，从 Python 后端到前端 Svelte，按层级梳理如下：

### 7.1 后端层面：`render=False`

**触发位置**：[blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/blocks.py#L165-L166)

```python
# Block.__init__ 第 165-166 行
if render:
    self.render()
```

当 `render=False` 时：
- 组件的 `render()` 方法不会被调用
- `self.is_rendered` 保持为 `False`
- 组件不会被添加到当前 `Blocks` 的 `blocks` 字典中
- 前端完全**不会收到**该组件的配置

**组件加入布局的关键代码**：[render()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/blocks.py#L197-L222)

```python
def render(self):
    root_context = get_blocks_context()
    render_context = get_render_context()
    self.rendered_in = LocalContext.renderable.get(None)
    # ...
    if render_context is not None:
        render_context.add(self)       # 加入父容器的 children
        self.parent = render_context
    if root_context is not None:
        root_context.blocks[self._id] = self  # 注册到 Blocks 全局
        self.is_rendered = True
    return self
```

`render=False` 的典型用途：
- 在 `gr.render()` 装饰器中动态创建组件
- 组件仅作为数据载体，不需要显示（如 `gr.State`）
- 稍后通过代码手动调用 `.render()` 加入布局

### 7.2 配置层面：`get_config()` 过滤

在 [get_config()](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/blocks.py#L309) 方法中，`render` 参数会被主动排除：

```python
config.pop("render", None)
```

这意味着前端**永远不会收到** `render` 字段，可见性完全由 `visible` 控制。

### 7.3 前端全局层面：MountComponents 渲染条件

**关键入口**：[MountComponents.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/core/src/MountComponents.svelte#L9-L30)

```svelte
{#if node && component}
    {#if node.props.shared_props.visible && !node.runtime}
        <!-- 内置组件：直接使用 svelte:component -->
        <svelte:component this={component.default} ... />
    {:else if node.props.shared_props.visible && node.runtime}
        <!-- 自定义组件：使用 MountCustomComponent 挂载 -->
        <MountCustomComponent {...rest} {node}>...</MountCustomComponent>
    {/if}
{/if}
```

**两层渲染条件**：
1. **外层**：`node && component` —— 组件对象存在且 JS 模块已成功加载
2. **内层**：`node.props.shared_props.visible` —— 只有 visible 为 truthy 才渲染

这里隐含的未渲染原因：
- **组件加载中**：`component` 是 Promise，尚未 resolve → 外层 `{#if}` 不满足 → 空白
- **组件加载失败**：`component` 为 `null`/undefined → 外层 `{#if}` 不满足 → 空白
- **visible 为 falsy** → 内层 `{#if}` 不满足 → 不进入挂载分支

### 7.4 组件个体层面：`visible` 的三种取值

`visible` 属性有三种取值，行为在 [Block.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/atoms/src/Block.svelte#L91-L98) 中统一处理：

```svelte
{#if visible === true || visible === "hidden"}
    <svelte:element ...
        class:hidden={visible === "hidden"}
        ...
    >
```

| visible 值 | 是否在 DOM 中 | 视觉可见性 | 说明 |
|-----------|-------------|----------|------|
| `true` | ✅ 是 | ✅ 可见 | 正常渲染 |
| `"hidden"` | ✅ 是 | ❌ 不可见 | DOM 存在，添加 `class:hidden`（`display: none`） |
| `false` | ❌ 否 | ❌ 不可见 | 整个组件从 DOM 中移除 |

**设计意图区别**：
- `visible=false`：完全卸载组件，释放资源（如视频播放器、WebSocket 连接）
- `visible="hidden"`：仅视觉隐藏，保留 DOM 状态和事件监听（如 Tab 切换、暂存表单输入）

### 7.5 各组件实现的不一致问题

由于历史原因，不同组件对 `visible` 的处理存在差异，这是排查"不显示"问题时容易忽略的点：

| 处理方式 | 组件示例 | `visible=false` 行为 | `visible="hidden"` 行为 |
|---------|---------|--------------------|----------------------|
| **标准 Block.svelte** | Textbox, Image, Slider 等大多数 | 不在 DOM | CSS 隐藏 |
| **{#if gradio.shared.visible}** | [Sidebar](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/sidebar/Index.svelte#L18) | 不在 DOM | **仍在 DOM 且可见**（字符串 `"hidden"` 是 truthy） |
| **class:hide={!visible}** | [Row](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/row/Index.svelte#L50) | CSS 隐藏（`!false = true`） | **仍可见**（`!"hidden" = false`） |
| **Tab 逻辑** | [TabItem](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/tabitem/shared/TabItem.svelte#L53) | `display:none` | **仍显示**（只检查 `!== false`） |

**典型陷阱**：使用 `visible="hidden"` 在 Row、Sidebar、TabItem 上**不会生效**，因为它们的实现不支持这种模式。

### 7.6 动态渲染层面：`rendered_in` 匹配

在 [_init.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/core/src/_init.ts#L291-L315) 中，动态渲染（`gr.render()` 装饰器）有一段清理逻辑：

```javascript
Object.entries(instance_map).forEach(([id, component]) => {
    let _id = Number(id);
    if (component.rendered_in === render_id) {
        let replacement_component = replacement_components.find(
            (c) => c.key === component.key
        );
        if (component.key != null && replacement_component !== undefined) {
            // 有 key → 更新 props
        } else {
            // 无 key 或找不到匹配 → 从 instance_map 中删除
            if (instance_map) delete instance_map[_id];
            if (_component_map.has(_id)) {
                _component_map.delete(_id);
            }
        }
    }
});
```

动态渲染导致组件消失的常见原因：
1. **未设置 `key`**：每次 render 重新创建时，旧组件会被直接删除
2. **`key` 不匹配**：前后两次 render 的 key 对不上，旧组件被删除
3. **`rendered_in` 对不上**：新渲染的 render_id 与旧组件不匹配

### 7.7 组件未渲染排查清单

按优先级顺序排查：

| 层级 | 排查项 | 验证方法 |
|------|-------|---------|
| Python | `render=False` 未手动调用 `.render()` | 检查 `comp.is_rendered` 属性 |
| Python | 组件在 Blocks 上下文外创建 | 检查是否在 `with gr.Blocks():` 内 |
| 配置 | `visible=False` | 浏览器 F12 搜索组件 ID，看是否在 DOM |
| 配置 | `visible="hidden"` + 不支持该模式的组件 | 查看组件源码对 visible 的处理 |
| 前端 | 组件 JS 加载失败 | 浏览器 Console 看 404/import 错误 |
| 前端 | 动态渲染 key 不匹配 | 检查 render 装饰器设置的 key 参数 |
| SSR | 服务端回退到 Fallback | 检查 SSR 模式，查看 hydration 后是否正常 |

---

## 八、浏览器与服务端集成路径差异

Gradio 在浏览器和服务端（SSR/Node.js 环境）对自定义组件的处理存在显著差异，核心原因是：**Node.js 环境缺少浏览器 DOM API，且自定义组件可能依赖浏览器特性**。

### 8.1 组件加载路径：environment 参数

路由定义中的 `environment` 参数是区分两者的关键：

```python
# routes.py 第 984 行
environment: Literal["client", "server"],
```

#### 8.1.1 浏览器环境（environment="client"）

[component_loader.js](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/build/out/component_loader.js#L92) 通过 `is_browser` 判断：

```javascript
const environment = is_browser ? "client" : "server";
```

浏览器环境的完整加载路径：
1. `load_component()` → `get_component_type()`
2. 检查 `window.__GRADIO__CC__`（开发模式自定义组件）
3. 检查内置 `component_map`
4. **动态 HTTP 加载**：向 `/custom_component/{id}/client/{variant}/index.js` 发起请求
5. `import()` 动态加载 ES Module
6. 返回 `[component_promise, runtime_promise]`

#### 8.1.2 服务端环境（environment="server"）

SSR 模式下走完全不同的路径：

```javascript
// component_loader.js 第 94-98 行
if (environment === "server") {
    // Fall back to @gradio/fallback during SSR;
    // the real component will be loaded client-side.
    return [import("@gradio/fallback"), Promise.resolve(false)];
}
```

关键点：
- **不发起任何 HTTP 请求**，直接返回 `@gradio/fallback`
- `runtime` 返回 `false`（表示不需要独立的 Svelte 运行时）
- 真实组件完全不参与 SSR

对应后端路由中也有相关注释：
```python
# routes.py 第 1025-1026 行
# Uncomment when we support custom component SSR
# if environment == "server":
```

表明自定义组件的 SSR 支持目前处于**未实现状态**。

### 8.2 两阶段渲染流程（SSR + 客户端水化）

当 `ssr_mode=True` 时，页面渲染分为两个阶段：

```
阶段 1：SSR (Node.js 环境)
    ├─ SvelteKit +page.ts 的 load() 执行
    ├─ 调用 Client.connect() 获取 config
    ├─ 自定义组件 → 强制回退为 @gradio/fallback
    ├─ Fallback 以 JSON 形式渲染 value（空壳）
    └─ 生成 HTML 发送给浏览器

阶段 2：客户端水化 (Browser 环境)
    ├─ 浏览器接收 HTML 并渲染（看到 Fallback 的 JSON）
    ├─ Svelte hydration 启动
    ├─ 重新执行 load_component() → environment="client"
    ├─ 发起 /custom_component/{id}/client/... 请求
    ├─ 加载真实组件并替换 Fallback
    └─ 用户看到真实组件界面（可能有闪烁）
```

**用户可见影响**：开启 SSR 后，自定义组件区域会先显示 JSON 内容，闪烁后才变为真实组件。

### 8.3 API URL 构造差异

在 SvelteKit 页面加载器 [+page.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/app/src/routes/%5B...catchall%5D/%2Bpage.ts#L32-L47) 中，API URL 构造方式不同：

```typescript
const api_url =
    browser && !local_dev_mode && root_url
        ? new URL(mount_path || "/", root_url).href   // 浏览器：使用 root_url（绝对路径）
        : server;                                       // SSR：使用 server（可能是 localhost）

const headers = new Headers();
if (!browser) {
    // SSR 环境：附加服务端专用 header
    headers.append("x-gradio-server", root_url);
    if (cookie) {
        headers.append("Cookie", cookie);            // 转发浏览器 Cookie
    }
} else {
    // 浏览器环境：基于当前 origin
    headers.append(
        "x-gradio-server",
        new URL(mount_path, location.origin).href
    );
}
```

| 维度 | 浏览器环境 | 服务端环境 (SSR) |
|------|----------|---------------|
| API URL 基准 | `root_url` / 当前页面 origin | `server`（内部通信地址） |
| Cookie 传递 | 浏览器自动附带 | 手动从 load 参数转发 |
| Header 标识 | `x-gradio-server: 当前页面 origin` | `x-gradio-server: root_url` |

### 8.4 鉴权处理差异

在 [+page.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/app/src/routes/%5B...catchall%5D/%2Bpage.ts#L50-L93) 中，SSR 阶段如果检测到需要鉴权，会**跳过 Client.connect**：

```typescript
// If the server-side check determined auth is required, skip Client.connect
// This prevents the 401 error on the client during hydration
if (auth_required) {
    await setupi18n(undefined, accept_language);
    return {
        config: {
            auth_required: true,         // 标记需要鉴权
            components: [],              // 空组件列表
            dependencies: [],
            layout: {},
            // ... 其余字段占位
        },
        api_url,
        layout: {},
        app: null                        // app 实例为 null
    };
}
```

这意味着：
- **SSR 阶段**：遇到鉴权不返回真实 config，页面不渲染任何组件
- **客户端阶段**：重新发起 connect，获取登录表单或跳转鉴权页面
- **设计目的**：避免 SSR 阶段出现 401 错误，导致 hydration 前后 HTML 不一致

### 8.5 自定义组件：开发模式 vs 生产模式

| 维度 | 开发模式 (gradio cc dev) | 生产模式 (pip install) |
|------|------------------------|---------------------|
| 组件来源 | `window.__GRADIO__CC__` 全局变量 | `/custom_component/{id}/client/...` HTTP 请求 |
| Svelte 运行时来源 | `window.__GRADIO__CC__RUNTIMES__` | `/custom_component/{id}/client/.../svelte_runtime_entry.js` |
| 热更新 | ✅ Vite HMR 直接生效 | ❌ 需重新安装包 |
| SSR 行为 | 同样回退到 Fallback | 同样回退到 Fallback |
| 组件 JS 构建 | ❌ 不需要构建（Vite 按需编译） | ✅ 必须先 gradio cc build |

### 8.6 差异总结

| 层面 | 浏览器 (Client) | 服务端 (SSR) |
|-----|---------------|------------|
| 自定义组件加载 | HTTP 动态加载真实组件 | 强制回退 Fallback |
| 自定义组件 SSR | N/A（运行时） | 尚未支持 |
| environment 参数 | `"client"` | `"server"` |
| runtime 返回值 | 组件独立的 Svelte 运行时 | `false` |
| API URL | root_url / location.origin | server 内部地址 |
| Cookie 传递 | 自动附带 | 手动转发 |
| 组件挂载方式 | MountCustomComponent mount API | 直接 svelte:component |
| 鉴权失败处理 | 显示登录 UI | 返回空 config 避免 hydration 错误 |

---

## 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 创建命令 | [create.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/create.py) |
| 创建工具函数 | [_create_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/_create_utils.py) |
| 构建命令 | [build.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/build.py) |
| 开发命令 | [dev.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/dev.py) |
| 组件检查脚本 | [examine.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/examine.py) |
| 前端构建逻辑 | [build.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/build.ts) |
| 开发服务器 | [dev.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/dev.ts) |
| Vite 插件 | [plugins.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/preview/src/plugins.ts) |
| 组件加载器 (运行时) | [component_loader.js](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/build/out/component_loader.js) |
| 组件加载器 (构建插件) | [index.js](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/build/out/index.js) |
| 组件挂载组件 | [MountCustomComponent.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/core/src/MountCustomComponent.svelte) |
| 组件树渲染入口 | [MountComponents.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/core/src/MountComponents.svelte) |
| 动态渲染/初始化逻辑 | [_init.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/core/src/_init.ts) |
| Block 原子组件 | [Block.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/atoms/src/Block.svelte) |
| Fallback 组件 (Python) | [fallback.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/components/fallback.py) |
| Fallback 组件 (Svelte) | [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/fallback/Index.svelte) |
| 后端路由 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/routes.py#L981-L1044) |
| SSR 页面加载器 | [+page.ts](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/js/app/src/routes/%5B...catchall%5D/%2Bpage.ts) |
| 组件基类 | [base.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/components/base.py) |
| Block 基类 | [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/blocks.py) |
| pyproject.toml 模板 | [pyproject_.toml](file:///d:/fz/0601/solo-dogfeeding/code/250-gradio/gradio/cli/commands/components/files/pyproject_.toml) |
