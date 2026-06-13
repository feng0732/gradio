# Gradio Themes 主题系统代码解析

## 一、整体架构概览

Gradio 的主题系统采用**四层架构 + 变量引用链**的设计模式，从 Python 后端的主题对象构造，到 CSS 变量生成，再到前端 Svelte 应用的消费，形成一条完整的管线。

```
┌─────────────────────────────────────────────────────────────────┐
│  Python 后端（构造层）                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  Color/Size  │  │  Font 类族   │  │   ThemeClass (基类)     │ │
│  │  (原子令牌)  │  │ (字体加载器) │  │  序列化/反序列化/CSS生成 │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬────────────┘ │
│         │                 │                      │              │
│         └────────┬────────┴──────────────────────┘              │
│                  ▼                                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Base(ThemeClass) — 核心主题基类                           │   │
│  │  __init__(): 原子令牌展开 → 调用 set()                     │   │
│  │  set(): 200+ 变量的"覆盖链"赋值 (引用机制: *var)           │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │ 继承 + 覆盖                             │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Soft / Default / Glass / Monochrome / ... (具体主题)     │   │
│  │  改构造参数 + super().set(选择性覆盖变量)                  │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │  ._get_theme_css()                     │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CSS 字符串生成: :root + :root.dark 两个变量作用域          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │ /theme.css 路由 + stylesheets 数组
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  前端 Svelte 应用（消费层）                                        │
│  1. mount_css() 加载 /theme.css (含缓存哈希)                      │
│  2. 加载 stylesheets (Google Fonts 等外链样式)                    │
│  3. apply_theme(): 给 body 添加/移除 .dark class 切换明暗模式       │
│  4. prefix_css(): Shadow DOM 下的 CSS 作用域隔离                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、核心文件结构

| 文件 | 职责 |
|------|------|
| [base.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py) | `ThemeClass` + `Base` — 主题系统核心引擎 |
| [colors.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/utils/colors.py) | `Color` 类 + 20+ 套调色板（Tailwind 色阶） |
| [sizes.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/utils/sizes.py) | `Size` 类 + 多套 spacing/radius/text 尺寸预设 |
| [fonts.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/utils/fonts.py) | `Font` / `GoogleFont` / `LocalFont` 字体体系 |
| [soft.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/soft.py) 等 | 具体主题子类，差异化配置 |
| [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/blocks.py) `L2569-L2574` | `_set_html_css_theme_variables()` 触发 CSS 生成 + 哈希 |
| [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/routes.py) `L1783-L1786` | `/theme.css` 静态路由 |
| [css.ts](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/css.ts) | `mount_css()` 动态加载样式 + `prefix_css()` 作用域隔离 |
| [\+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte) `L122-L168` | 明暗模式切换逻辑（handle_theme_mode / apply_theme） |

---

## 三、主题构造流程（四层赋值模型）

主题的构造本质上是一个**"自底向上"的四层赋值链**，每层都可以覆盖上层的值。

### 第 1 层：原子令牌层（构造参数 → 基础变量）

由 [Base.__init__()](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py#L367-L522) 完成，接收 7 个构造参数：

```python
def __init__(self, *,
    primary_hue   = colors.blue,    # Color 对象
    secondary_hue = colors.blue,    # Color 对象
    neutral_hue   = colors.zinc,    # Color 对象
    text_size     = sizes.text_md,  # Size 对象
    spacing_size  = sizes.spacing_md, # Size 对象
    radius_size   = sizes.radius_md, # Size 对象
    font          = (...IBMPlexSans, "system-ui", ...),
    font_mono     = (...IBMPlexMono, "Consolas", ...),
):
```

**步骤 1.1 — 快捷字符串展开**：`expand_shortcut()` 允许用户传入 `"blue"` 字符串，自动映射到 `colors.blue` 对象。

**步骤 1.2 — 色阶展开**：每个 `Color` 对象包含 11 个色阶（c50 → c950），逐一展开到实例属性：

```python
self.primary_50   = primary_hue.c50    # 实际值如 "#eff6ff"
self.primary_100  = primary_hue.c100
...
self.primary_950  = primary_hue.c950
# 同样展开 secondary_* / neutral_* 共 33 个色阶变量
```

**步骤 1.3 — 尺寸展开**：每个 `Size` 对象包含 7 个档位（xxs → xxl）：

```python
self.spacing_xxs = spacing_size.xxs  # 如 "1px"
...
self.spacing_xxl = spacing_size.xxl  # 如 "16px"
# 同样展开 radius_* / text_* 共 21 个尺寸变量
```

**步骤 1.4 — 字体处理**：遍历 `_font` + `_font_mono`，为每个字体生成 `stylesheet()`：
- `GoogleFont`：若本地有字体文件则降级为 LocalFont，否则生成 Google Fonts CDN URL → 加入 `_stylesheets[]`
- `LocalFont`：生成 `@font-face { ... }` CSS 字符串 → 加入 `_font_css[]`

**步骤 1.5 — 调用 `self.set()`**：进入第 2 层。

---

### 第 2 层：派生变量层（set() 方法的覆盖链）

[Base.set()](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py#L524-L2073) 有 **~200 个 keyword-only 参数**，每个参数的赋值都遵循统一模式：

```python
self.body_background_fill = (
    body_background_fill                        # ① 用户显式传入
    or getattr(
        self, "body_background_fill",           # ② 实例上已有值（子类在调用前设置）
        "*background_fill_primary"              # ③ 默认值（可含 *引用）
    )
)
```

**这是最核心的"变量覆盖"逻辑**，优先级从高到低：

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1（最高） | `set(X=值)` 显式参数 | 子类或用户直接传入的参数 |
| 2 | `getattr(self, "X")` | 实例上已有的同名属性（可能来自子类覆盖） |
| 3（最低） | 默认字面量 / `*引用` | Base 类定义的兜底值 |

> **注意 `or` 的语义**：只要优先级 1 的值是 truthy，就覆盖优先级 2、3。所以要"清空"某个变量不能传空串，只能手动操作属性。

---

### 第 3 层：具体主题子类层

以 [Soft](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/soft.py) 为例：

```python
class Soft(Base):
    def __init__(self, *,
        primary_hue = colors.indigo,   # 改构造参数 → 影响第 1 层
        neutral_hue = colors.gray,
        ...
    ):
        super().__init__(primary_hue=primary_hue, ...)  # 第 1、2 层先执行完
        self.name = "soft"
        super().set(                        # 用优先级 1 覆盖第 2 层的默认值
            background_fill_primary="*neutral_50",
            block_label_background_fill="*primary_100",
            button_primary_shadow="*shadow_drop_lg",
            ...  # 选改约 40 个变量，其余沿用 Base 默认
        )
```

**子类的两种覆盖手段**：
1. **改构造参数**（影响第 1 层）：换色调（primary_hue）、换字号密度（text_size）等——影响面大
2. **改 set 参数**（影响第 2 层）：改具体派生变量——影响面精准

---

### 第 4 层：用户自定义层

用户拿到主题对象后，可以继续链式 `.set()`：

```python
theme = gr.themes.Soft().set(
    button_primary_background_fill="#ff0000",
    body_background_fill_dark="#111111",
)
with gr.Blocks(theme=theme) as demo:
    ...
```

---

## 四、变量引用机制（`*variable` 语法）

这是主题系统中**最绕的设计**，也是变量覆盖逻辑的关键。

### 4.1 引用的定义与语义

在赋值时，值字符串中出现 `*变量名` 表示"引用另一个主题变量的值"：

```python
# base.py L1090-L1092
self.color_accent = color_accent or getattr(
    self, "color_accent", "*primary_500"   # ← 默认值引用了 primary_500
)
```

含义：`color_accent` 的默认值等于 `primary_500` 的值。当用户只改了 `primary_hue` 时，`color_accent` 会自动跟随变化。

引用可以嵌套、可以出现在复杂表达式中：

```python
# L1246-L1248
self.block_label_padding = ... or "*spacing_sm *spacing_lg"
#       解析为: var(--spacing-sm) var(--spacing-lg)

# L1249-L1253
self.block_label_radius = ... or "calc(*radius_sm - 1px) 0 calc(*radius_sm - 1px) 0"
#       解析为: calc(var(--radius-sm) - 1px) 0 calc(var(--radius-sm) - 1px) 0
```

### 4.2 引用的两种解析路径

#### 路径 A：CSS 运行时解析（_get_theme_css）

[`_get_theme_css()`](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py#L33-L100) 输出 CSS 时，`*变量名` 被正则替换为 `var(--变量名)`：

```python
pattern = r"(\*)([\w_]+)(\b)"
def repl_func(match):
    word = match.group(2)
    word = word.replace("_", "-")
    return f"var(--{word})"

val = re.sub(pattern, repl_func, val)  # "*primary_500" → "var(--primary-500)"
```

**结果**：浏览器通过 CSS 变量的动态特性，在运行时解析引用。这样切换 `:root.dark` 就能联动所有依赖变量。

#### 路径 B：Python 静态解析（_get_computed_value）

[`_get_computed_value()`](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py#L102-L124) 用于服务端需要拿到**最终字面量**的场景（如 SSR 时内联 body 的背景色，见 blocks.py L2406-L2407）：

```python
def _get_computed_value(self, property: str, depth=0) -> str:
    if depth > 100:  # 循环引用防护
        warn("circular reference")
        return ""
    is_dark = property.endswith("_dark")
    set_value = getattr(self, property, ...)  # dark 缺省回退到 light

    pattern = r"(\*)([\w_]+)(\b)"
    def repl_func(match, depth):
        word = match.group(2)
        dark_suffix = "_dark" if is_dark else ""
        return self._get_computed_value(word + dark_suffix, depth + 1)  # 递归

    return re.sub(pattern, lambda m: repl_func(m, depth), set_value)
```

**关键行为**：如果当前属性是 `X_dark`，引用的目标会自动变成 `Y_dark`。这就是为什么在 light 模式下写 `color_accent = "*primary_500"` 时，dark 模式下会自动引用 `primary_500_dark`——**引用解析时会自动传递暗色调后缀**。

### 4.3 引用的约束检查（_get_theme_css 中的 repl_func）

代码在 [base.py L51-L68](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py#L51-L68) 做了三项校验：

| 校验 | 错误示例 | 说明 |
|------|---------|------|
| 禁止在值中使用 `*X_dark` | `body_text_color = "*neutral_100_dark"` | dark 引用是自动附加的，不需手动写后缀 |
| 禁止 `X_dark` 引用 `X` 本身 | `body_text_color_dark = "*body_text_color"` | 会造成双向绑定；如二者相同，应设 `None` |
| 禁止非 dark 变量设 `None` | `body_text_color = None` | 只有 `*_dark` 属性可以是 `None`（表示与亮色相同） |

---

### 4.4 Dark 模式变量的 None 语义

`None` 是 `*_dark` 变量的合法特殊值，含义是"暗模式和亮模式相同，不需要单独声明"：

```python
# base.py L1198-L1200
self.block_border_width_dark = ... or getattr(
    self, "block_border_width_dark", None  # ← None 表示与 light 同值
)
```

在 `_get_theme_css()` 的输出阶段（L80-L82），如果某个亮变量在 `dark_css` 中是 `None` 或缺失，会自动用亮值回填：

```python
for attr, val in css.items():
    if attr not in dark_css:
        dark_css[attr] = val          # light 值兜底
```

---

## 五、CSS 输出流程

### 5.1 _get_theme_css() 输出结构

```python
# L33-L100
def _get_theme_css(self):
    css = {}          # 亮模式变量字典
    dark_css = {}     # 暗模式变量字典

    for attr, val in self.__dict__.items():
        if attr.startswith("_"):  continue   # 跳过内部属性 _stylesheets/_font_css 等
        # ... 正则替换 *var → var(--xxx) ...
        # ... attr 下划线 → 连字符 ...
        if attr.endswith("-dark"):
            dark_css[attr[:-5]] = val        # 去除 _dark 后缀存入 dark_css
        else:
            css[attr] = val

    # dark_css 缺失的变量 → 用 css 同值兜底
    for attr, val in css.items():
        if attr not in dark_css:
            dark_css[attr] = val

    # 拼装最终字符串
    font_css    = "\n".join(self._font_css)    # LocalFont 的 @font-face
    css_code    = ":root {\n  --xxx: val;\n}"
    dark_css_code = "\n:root.dark, :root .dark {\n  --xxx: val;\n}"
    theme_css   = f"{font_css}\n{css_code}\n{dark_css_code}"
    if self.custom_css:
        theme_css += f"\n\n{self.custom_css}"  # 用户自定义 CSS 追加
    return theme_css
```

**输出结构示意**：

```css
/* _font_css: LocalFont 的内联 @font-face */
@font-face {
    font-family: 'IBM Plex Sans';
    src: url('static/fonts/IBMPlexSans/IBMPlexSans-Regular.woff2') format('woff2');
    font-weight: 400;
}

/* :root 作用域 → 所有变量的 light 模式默认值 */
:root {
  --primary-50: #eff6ff;
  --primary-100: #dbeafe;
  ...
  --color-accent: var(--primary-500);
  --body-background-fill: var(--background-fill-primary);
  ...
}

/* :root.dark 作用域 → 覆写为 dark 模式值（自动补齐 light 中未声明的变量） */
:root.dark, :root .dark {
  --primary-50: #172554;
  ...
  --body-background-fill: var(--neutral-950);
  ...
}

/* custom_css（如果有） */
```

### 5.2 变量命名映射

| Python 属性名 | CSS 变量名 |
|--------------|-----------|
| `primary_50` | `--primary-50` |
| `body_background_fill` | `--body-background-fill` |
| `button_primary_background_fill_dark` | `--button-primary-background-fill`（在 `:root.dark` 块内） |

转换规则：下划线 `_` → 连字符 `-`，`_dark` 后缀剥离并迁入 `:root.dark` 选择器。

---

## 六、服务端路由与前端消费

### 6.1 后端触发：_set_html_css_theme_variables()

在 [blocks.py L2569-L2574](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/blocks.py#L2569-L2574)，`launch()` 调用链中会执行：

```python
def _set_html_css_theme_variables(self):
    self.theme_css     = self.theme._get_theme_css()  # 生成 CSS 字符串
    self.stylesheets   = self.theme._stylesheets      # Google Fonts URL 等外链
    theme_hasher       = hashlib.sha256()
    theme_hasher.update(self.theme_css.encode("utf-8"))
    self.theme_hash    = theme_hasher.hexdigest()     # 内容哈希 → 用于浏览器缓存
```

同时，`blocks.get_config()` 会把 `stylesheets`、`theme_hash`、`body_css` 等注入到前端 config JSON（blocks.py L2402-L2407）。

### 6.2 /theme.css 路由

[routes.py L1783-L1786](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/routes.py#L1783-L1786) 暴露静态路由：

```python
@router.get("/theme.css", response_class=PlainTextResponse)
def theme_css():
    return PlainTextResponse(app.get_blocks().theme_css, media_type="text/css")
```

### 6.3 前端加载：mount_css() + 主题哈希

前端在 [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) 中：

```javascript
// 1) 加载主主题 CSS（带哈希做缓存键）
await mount_css(config.root + "/theme.css?v=" + config.theme_hash, document.head);

// 2) 加载 stylesheets 数组中的外链（Google Fonts 等）
if (config.stylesheets) {
    await Promise.all(config.stylesheets.map(stylesheet => {
        if (绝对URL) return mount_css(stylesheet, document.head); // <link rel=stylesheet>
        else         fetch → prefix_css → <style> 注入;        // 本地 CSS 内联
    }));
}
```

[mount_css()](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/css.ts#L15-L37) 的核心是创建 `<link rel="stylesheet">` 并监听 `load` 事件，已存在的 URL 不重复加载。

### 6.4 明暗模式切换

[+page.svelte L122-L168](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte#L122-L168) 中的 `apply_theme()`：

```javascript
function apply_theme(target, theme: "dark" | "light") {
    // 嵌入模式/独立模式选择不同的元素挂 .dark
    const dark_class_element = is_embed ? target.parentElement! : document.body;
    dark_class_element.classList.add("theme-loaded");

    if (theme === "dark")   dark_class_element.classList.add("dark");
    else                    dark_class_element.classList.remove("dark");
    // ↑ 此操作触发 CSS 中 :root.dark 选择器生效，所有变量切换为暗模式值
}
```

模式优先级：`显式 theme_mode prop` > `URL ?__theme=dark|light` > `matchMedia("(prefers-color-scheme: dark)")`（system）。

### 6.5 Shadow DOM 隔离：prefix_css()

[prefix_css()](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/css.ts#L39-L119) 在组件嵌入场景下（通过 adoptedStyleSheets 注入样式）给所有选择器加上 `.gradio-container.gradio-container-${version} .contain` 前缀，防止样式外泄到宿主页面。同时会把 `@import` 单独抽离放在最前面（CSS 语法约束），并保留 `@keyframes` / `@font-face` 等 at-rule。

---

## 七、序列化与版本兼容

### 7.1 JSON 导出/导入

| 方法 | 用途 |
|------|------|
| `theme.dump("path.json")` | 写入磁盘 |
| `theme.to_dict()` | → 带 `gradio_version` 的 dict |
| `ThemeClass.load("path.json")` | 从磁盘读回 |
| `ThemeClass.from_dict(dict)` | 核心反序列化函数 |
| `ThemeClass.from_hub("author/theme")` | 从 HuggingFace Hub 下载 |

**from_dict 的关键兼容逻辑**（base.py L175-L179）：

```python
base = Base()
for attr in base.__dict__:
    if not attr.startswith("_") and not hasattr(new_theme, attr):
        setattr(new_theme, attr, getattr(base, attr))
```

**效果**：旧版本主题 JSON 缺少新版 Gradio 新增的变量时，自动用当前 `Base` 的默认值补齐——保证向前兼容。

同时比较 `gradio_version` 的主次版本号，不一致时 warn（L160-L169）。

### 7.2 字体对象的自定义序列化

[FontEncoder / as_font](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/utils/fonts.py#L42-L76) 使用自定义 JSON hook：

```python
# 序列化时：Font → {"__gradio_font__": True, "name": "Inter", "class": "google", "weights": [400,600]}
# 反序列化时：检测到 "__gradio_font__" key → 构造对应 GoogleFont/LocalFont/Font 实例
```

---

## 八、变量命名约定与分层速览

所有变量（约 250+ 个）按语义分组，命名层次：

```
{前缀}_{语义}_{状态}_{后缀}
 │       │      │     │
 │       │      │     └── 可选: _dark (暗模式专用，None=同步亮模式)
 │       │      └── 可选: hover/focus/selected/active 等交互态
 │       └── 必选: 语义描述 (background_fill, border_color, text_size, shadow...)
 └── 必选: 命名空间
```

命名空间分层：

| 前缀 | 含义 | 典型变量 |
|------|------|---------|
| `primary_/secondary_/neutral_` | 色阶原子 | `primary_50` ~ `primary_950` |
| `spacing_/radius_/text_` | 尺寸原子 | `spacing_xxs` ~ `radius_xxl` |
| `font/font_mono` | 字体原子 | `font` / `font_mono` |
| `body_` | 全局背景/文本 | `body_background_fill`, `body_text_color` |
| `background_fill_/border_color_/color_` | 通用颜色原子 | `background_fill_primary`, `color_accent` |
| `shadow_` | 通用阴影 | `shadow_drop`, `shadow_inset` |
| `block_/panel_/container_/layout_` | 布局原子容器 | `block_background_fill`, `panel_border_width` |
| `block_label_/block_title_/block_info_` | Block 部件 | `block_label_radius`, `block_title_padding` |
| `input_/checkbox_/checkbox_label_/slider_/loader_/error_` | 组件原子 | `input_background_fill`, `error_text_color` |
| `button_{primary/secondary/cancel}_` | 按钮按变体分 | `button_primary_background_fill_hover` |
| `button_{small/medium/large}_` | 按钮按尺寸分 | `button_large_padding` |
| `link_/prose_/code_/table_/chatbot_/stat_` | 专用组件样式 | `link_text_color_visited`, `table_radius` |

**最常用的"覆盖传播链"示例**（引用路径 → 最终落在色阶上）：

```
button_primary_background_fill → *primary_500 → primary_500 = blue.c500 = "#2563eb"
body_background_fill           → *background_fill_primary → (默认 "white")
block_border_color             → *border_color_primary → *neutral_200 → neutral_200 = "#e4e4e7"
```

---

## 九、关键代码片段速查

### 变量引用 → CSS var()（L49-L78）

```python
pattern = r"(\*)([\w_]+)(\b)"
def repl_func(match):
    ...校验...
    word = match.group(2).replace("_", "-")
    return f"var(--{word})"
val = re.sub(pattern, repl_func, val)
```

### 暗模式变量自动回退亮模式（L80-L82）

```python
for attr, val in css.items():
    if attr not in dark_css:
        dark_css[attr] = val
```

### set() 的三层赋值模式（到处都是）

```python
self.X = param_X or getattr(self, "X", "默认值或*引用")
```

### from_dict 旧版本兼容补齐（L175-L179）

```python
base = Base()
for attr in base.__dict__:
    if not attr.startswith("_") and not hasattr(new_theme, attr):
        setattr(new_theme, attr, getattr(base, attr))
```

---

## 十、设计亮点与潜在陷阱

### 亮点
1. **引用驱动的变量连锁**：只改 3 个 hue 对象就能让 ~250 个变量自动适配，`*var` 语法比手动级联简洁得多。
2. **dark 模式自动代理**：`*ref` 在暗模式下自动寻找 `ref_dark`，用户不需要写两套引用。
3. **缓存友好**：theme_css 的内容哈希做 URL 参数，浏览器缓存精准。
4. **向前兼容**：`from_dict()` 用当前 `Base` 做缺失属性的兜底，老主题 JSON 在新版也能用。

### 潜在陷阱
1. **`or` 语义的坑**：`set(block_border_width="0px")` 会被 `or getattr(self, ...)` 忽略（因为 `"0px"` truthy，但空串 `""` 会被忽略）。`None` 不能用于非 dark 变量。
2. **引用循环**：`A → *B` 且 `B → *A`，`_get_computed_value()` 有 `max_depth=100` 保护，但 `_get_theme_css()` 走 CSS 运行时解析，浏览器会报循环引用警告。
3. **dark 变量默认 `None` 的含义**：很多 `*_dark` 在 Base.set() 中默认就是 `None`，表示和 light 相同——在 CSS 输出阶段才会被 light 值回填，读代码时容易困惑"为什么 dark 字典里没有这个键"。
4. **子类调用顺序**：子类必须先 `super().__init__()`（会自动调用 `self.set()` 产生默认值），再 `super().set(...)` 覆盖。如果顺序颠倒会出 bug。
