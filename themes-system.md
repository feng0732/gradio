# Gradio Themes 主题系统代码解析

> **2026-06-13 勘误更新**：经过第二轮代码核查，发现并修正了第十一章中关于独立/嵌入模式样式链路的多处不准确描述，包括：
> - ✅ 修正用户自定义 CSS 的注入路径（独立模式实际由 `<Blocks>` 内部注入）
> - ✅ 修正 `.theme-loaded` 类的使用差异（嵌入模式 `apply_theme` 缺失该类）
> - ✅ 修正防闪屏机制的实际参与情况（仅独立模式有效）
> - ✅ 发现并记录了 3 个代码 Bug（本地 stylesheets 注入失效、`prefix_css` 冗余 `remove()`）
> - ✅ 补充 `css_ready` 变量在两种模式下的不同作用
> 
> 第十一章已全部重写，新增了详细的调用时机、代码位置和 Bug 说明。

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

---

## 十一、独立模式 vs 嵌入模式：主题样式链路深度对比

Gradio 应用有两种运行模式，主题 CSS、字体和暗色模式的注入/生效链路存在本质差异。

### 11.1 两种模式的入口与判断

| 模式 | 入口 | `is_embed` 默认值 | 典型场景 |
|------|------|-------------------|----------|
| **独立模式（App 模式）** | SvelteKit 路由 `/[...catchall]` | `false`（+page.svelte L115） | 用户直接在浏览器打开 Gradio 应用 |
| **嵌入模式（SPA 模式）** | Web Component `<gradio-app>` | `true`（main.ts L68） | 用 `<gradio-app src="...">` 嵌入到其他网页 |

#### 判断逻辑

**嵌入模式判断链**（[main.ts](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/main.ts) L44-L130）：
```javascript
class GradioApp extends HTMLElement {
    is_embed: string;
    constructor() {
        this.is_embed = this.getAttribute("embed") ?? "true";  // 默认 true
    }
    connectedCallback() {
        const opts = {
            props: {
                is_embed: this.is_embed === "false" ? false : true  // 任何非 "false" 都视为 true
            }
        };
        this.app = mount(IndexComponent, opts);  // 传给 Index.svelte
    }
}
```

**独立模式判断**（[+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte) L109-L120）：
```javascript
let {
    is_embed = false,    // 默认 false
    ...
}: Props = $props();
```

---

### 11.2 主题 CSS 注入路径对比

两种模式下主题 CSS 的加载时机、方式、位置都不同，是理解整个链路的关键。

#### 链路总览

```
                              ┌─────────────────────────────────────┐
                              │        Python 后端（相同）           │
                              │  Base → Soft → _get_theme_css()     │
                              │  _stylesheets[], _font_css[]        │
                              │  theme_hash (SHA256)                │
                              └──────────────┬──────────────────────┘
                                             │ config.theme_css / config.stylesheets
                                             │ /theme.css 路由
                    ┌────────────────────────┴────────────────────────┐
                    │                                                 │
        ┌───────────▼───────────┐                         ┌───────────▼───────────┐
        │   独立模式（App）     │                         │   嵌入模式（SPA）     │
        │  SvelteKit SSR        │                         │  Web Component         │
        │  静态注入 head        │                         │  动态 JS 加载          │
        └───────────────────────┘                         └───────────────────────┘
```

#### 独立模式（App）注入路径

**第 1 步：布局层静态导入**（[+layout.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/+layout.svelte) L1-L6）

SvelteKit 构建时会将这些 CSS 打包进应用的样式块中：
```javascript
import "@gradio/theme/reset.css";       // 浏览器样式重置
import "@gradio/theme/global.css";      // 全局基础样式
import "@gradio/theme/pollen.css";      // pollen 设计令牌
import "@gradio/theme/typography.css";  // 排版预设
import "@gradio/theme/gradio-style.scss"; // Gradio 组件样式
```

**第 2 步：页面层静态注入主题 CSS**（[+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte) L422-L430）

通过 `<svelte:head>` 在 SSR 时将 `<link>` 标签注入到 `<head>`：
```svelte
<svelte:head>
    <link rel="stylesheet" href={root + "/theme.css?v=" + config?.theme_hash} />
    {#if config?.stylesheets}
        {#each config.stylesheets as stylesheet}
            {#if stylesheet.startsWith("http:") || stylesheet.startsWith("https:")}
                <link rel="stylesheet" href={stylesheet} />
            {/if}
        {/each}
    {/if}
</svelte:head>
```

**关键特点**：
- `/theme.css` 由 SvelteKit 在 SSR/CSR 时作为普通 `<link>` 注入到 `document.head`
- Google Fonts 等外链 stylesheets 同样走 `<link>` 静态注入
- **⚠️ 相对路径 stylesheets 被完全忽略**（`{#if stylesheet.startsWith("http:") || stylesheet.startsWith("https:")}` 判断只保留绝对 URL）
- 首屏即有样式，无 FOUC（无样式内容闪烁）

**第 3 步：Blocks 组件内联用户自定义 CSS**（[Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/Blocks.svelte) L463-L469）

用户通过 `gr.Blocks(css="...")` 传入的自定义 CSS，是在 `<Blocks>` 组件内部通过 `<svelte:head>` 注入的：
```svelte
<svelte:head>
    {#if control_page_title}
        <title>{title}</title>
    {/if}
    {#if css}
        {@html `<style>${prefix_css(css, version)}</style>`}
    {/if}
</svelte:head>
```

**注意**：`prefix_css(css, version)` 在这里只传了两个参数，`style_element` 参数为 `undefined`。`prefix_css` 内部会临时创建一个 `<style>` 元素用于 `CSSStyleSheet` 解析，解析完后立即 `remove()`，最终只返回处理后的字符串，由 `{@html}` 直接写入 `<style>` 标签。

**第 4 步：SSR 防闪屏内联样式**（[+layout.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/+layout.svelte) L12-L21）

```css
:global(body) {
    background: var(--body-background-fill);
    color: var(--body-text-color);
}

@media (prefers-color-scheme: dark) {
    :global(body:not(.theme-loaded)) {
        background: var(--neutral-950);  /* 深色模式下先显示深色背景 */
    }
}
```

在主题初始化完成（`.theme-loaded` 类加上）之前，用系统媒体查询兜底，避免白屏闪。

#### 嵌入模式（SPA）注入路径

**第 1 步：入口启动预加载**（[main.ts](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/main.ts) L33-L41, L86-L90）

`<gradio-app>` 自定义元素连接到 DOM 时，先预加载字体和入口 CSS：
```javascript
// main.ts L86-L90
if (typeof FONTS !== "string") {
    FONTS.forEach((f) => mount_css(f, document.head));  // 字体预加载
}
await mount_css(ENTRY_CSS, document.head);               // 应用入口 CSS
```

**第 2 步：Index.svelte 动态加载**（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L137-L171）

在 `mount_custom_css()` 中动态加载主题和用户自定义 CSS：
```javascript
let css_text_stylesheet: HTMLStyleElement | null = null;
async function mount_custom_css(css_string: string | null): Promise<void> {
    // 1) 用户自定义 CSS: 手动创建 <style> 标签，prefix_css 后写入 textContent
    if (css_string) {
        if (!css_text_stylesheet) {
            css_text_stylesheet = document.createElement("style");
            document.head.appendChild(css_text_stylesheet);
        }
        css_text_stylesheet.textContent = prefix_css(
            css_string, version, css_text_stylesheet
        );
    }

    // 2) 主题 CSS: 走 mount_css() 动态创建 <link>
    await mount_css(
        config.root + "/theme.css?v=" + config.theme_hash,
        document.head
    );

    // 3) stylesheets: 外链走 mount_css
    if (!config.stylesheets) return;
    await Promise.all(
        config.stylesheets.map((stylesheet) => {
            let absolute_link = stylesheet.startsWith("http:") || stylesheet.startsWith("https:");
            if (absolute_link) {
                return mount_css(stylesheet, document.head);  // 绝对URL → <link>
            }
            // ⚠️ Bug: 相对 URL fetch 后 prefix_css 的返回值被丢弃，没有写入 DOM！
            return fetch(config.root + "/" + stylesheet)
                .then((response) => response.text())
                .then((css_string) => {
                    prefix_css(css_string, version);  // 只处理但不写入！
                });
        })
    );
}
```

**嵌入模式的 Bug**：相对路径的 stylesheets 在 `fetch` 后调用了 `prefix_css(css_string, version)`，但返回值被直接丢弃，**没有被写入到任何 `<style>` 标签中**。因此，嵌入模式下的本地 stylesheets **实际上不会生效**。

**第 3 步：`css_ready` 控制渲染时机**（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L288, L359-L361, L595）

嵌入模式使用 `css_ready` 变量作为 `<Blocks>` 组件渲染的前置条件：
```javascript
let css_ready = false;
// ...
await mount_custom_css(config.css);
await add_custom_html_head(config.head);
css_ready = true;
```

在模板中：
```svelte
{:else if config && Blocks && css_ready}
    <Blocks {app} {...config} ... />
{/if}
```

这确保了在所有样式加载完成之前，不会渲染组件内容，避免 FOUC。

**关键差异总结**：

| 注入项 | 独立模式（App） | 嵌入模式（SPA） |
|--------|-----------------|-----------------|
| 主题 CSS `/theme.css` | `<svelte:head>` 静态 `<link>` | `mount_css()` 动态 `<link>` |
| 重置/全局/排版样式 | 构建时静态 import 打包 | main.ts 启动时 `mount_css(ENTRY_CSS)` 预加载 |
| Google Fonts 外链 | `<svelte:head>` 静态 `<link>` | `mount_css()` 动态 `<link>` |
| 本地 stylesheets | **仅加载绝对 URL**，相对 URL 被 `{#if}` 过滤 | 所有 URL 都 fetch，但相对 URL 的 `prefix_css` 结果被丢弃（Bug） |
| 用户自定义 `css` | `<Blocks>` 内部 `<svelte:head>` + `{@html}` 内联 | `Index.svelte` 手动创建 `<style>` 写入 `textContent` |
| 渲染时机控制 | 无 `css_ready` 检查，`{:else if config && app}` 即渲染 | 有 `css_ready` 检查，`{:else if config && Blocks && css_ready}` 才渲染 |
| SSR 防闪屏 | `+layout.svelte` 内联 body 背景 | 无 SSR，由嵌入方负责 |

---

### 11.3 字体样式加载机制

字体有三种类型，两种模式下加载路径也有差异。

#### 字体类型与后端分流

在 `Base.__init__()` 时就已完成字体的分类分流：

| 字体类型 | 后端处理 | 产物 |
|----------|---------|------|
| `GoogleFont` | 检查本地是否有字体文件 → 有则降级为 LocalFont；无则生成 CDN URL | `_stylesheets[]` 中的 URL 字符串 |
| `LocalFont` | 生成 `@font-face { ... }` CSS 字符串 | `_font_css[]` 中的 CSS 片段 |
| `Font`（系统字体） | 无需加载，仅在 CSS 变量中声明字体族 | 无额外产物 |

#### 独立模式下字体加载

1. **Google Fonts**：通过 `<svelte:head>` 中的 `<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...">` 加载
2. **LocalFont**：内联在 `/theme.css` 开头的 `@font-face` 规则中，随主题 CSS 一起加载
3. **字体变量**：通过 `/theme.css` 的 `:root { --font: "IBM Plex Sans", system-ui, ...; }` 声明

#### 嵌入模式下字体加载

1. **Google Fonts**：通过 `mount_css()` 动态创建 `<link>` 加载，与独立模式等效
2. **LocalFont**：同样内联在 `/theme.css` 中，随 `mount_css("/theme.css")` 加载
3. **内置字体预加载**：main.ts L86-L88 在连接时就遍历 `FONTS` 数组（构建时注入的字体列表）调用 `mount_css()` 预加载
4. **相对路径 stylesheets**：`fetch` 拉取后 `prefix_css()` 加作用域前缀，内联到 `<style>` 标签

#### 关键代码：mount_css() 实现

[css.ts](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/css.ts) L15-L37：

```javascript
export function mount_css(url: string, target: HTMLElement): Promise<void> {
    // CDN 适配：如果页面 origin 与构建 origin 不同，生成绝对 URL
    const base = new URL(import.meta.url).origin;
    var _url = url;
    if (window.location.origin !== base) {
        _url = new URL(url, base).href;
    }

    // 防重复：已存在相同 href 的 link 就跳过
    const existing_link = document.querySelector(`link[href='${_url}']`);
    if (existing_link) return Promise.resolve();

    const link = document.createElement("link");
    link.rel = "stylesheet";
    link.href = _url;

    return new Promise((res, rej) => {
        link.addEventListener("load", () => res());
        link.addEventListener("error", () => {
            console.error(`Unable to preload CSS for ${_url}`);
            res();   // 加载失败不阻塞
        });
        target.appendChild(link);   // 通常是 document.head
    });
}
```

**CDN 适配逻辑**是嵌入模式下的关键：当 Gradio 资源从 CDN 加载但页面在另一个 origin 时，`mount_css` 会把相对 URL 转成 CDN 的绝对 URL，确保样式能正确加载。

---

### 11.4 暗色模式切换：两种模式下的 `.dark` 类挂载差异

这是最容易混淆的部分。`apply_theme()` 在两种模式下把 `.dark` 类挂到**完全不同的 DOM 元素**上，而且**独立模式多了一行 `.theme-loaded` 类的添加**。

#### 核心代码对比

**独立模式版本**（[+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte) L158-L168）：
```javascript
function apply_theme(target: HTMLElement, theme: "dark" | "light"): void {
    const dark_class_element = is_embed ? target.parentElement! : document.body;
    const bg_element = is_embed ? target : target.parentElement!;

    bg_element.style.background = "var(--body-background-fill)";
    dark_class_element.classList.add("theme-loaded");  // ✓ 独立模式有这一行！

    if (theme === "dark")   dark_class_element.classList.add("dark");
    else                    dark_class_element.classList.remove("dark");
}
```

**嵌入模式版本**（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L269-L278）：
```javascript
function apply_theme(target: HTMLDivElement, theme: "dark" | "light"): void {
    const dark_class_element = is_embed ? target.parentElement! : document.body;
    const bg_element = is_embed ? target : target.parentElement!;

    bg_element.style.background = "var(--body-background-fill)";
    // ✗ 嵌入模式没有 .theme-loaded！

    if (theme === "dark") {
        dark_class_element.classList.add("dark");
    } else {
        dark_class_element.classList.remove("dark");
    }
}
```

**关键差异**：独立模式的 `apply_theme` 多了一行 `dark_class_element.classList.add("theme-loaded")`，这是防闪屏机制的核心开关。两份代码除了这一行之外逻辑一致，调用时机略有不同（独立模式在模块初始化时调用，嵌入模式在 `onMount` 时调用）。

**调用时机对比**：
- 独立模式：模块初始化时调用 `handle_theme_mode(document.body)`（[+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte) L172-L174）
- 嵌入模式：`onMount` 中调用 `handle_theme_mode(wrapper)`（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L306-L307）

#### DOM 结构与挂载点

先看最终渲染的 DOM 结构：

**独立模式 DOM 树**：
```
document.body                           ← .dark / .theme-loaded 挂在这里
└── <div class="gradio-container ...">  ← target.parentElement, bg_element
    └── <div class="main ...">          ← target（<Embed> 的 wrapper）
        └── <Blocks ... />
```

| 变量 | 值 | 说明 |
|------|----|------|
| `target` | `<div class="main ...">` | `wrapper` 绑定到 `<Embed>` 的内部 div |
| `dark_class_element` | `document.body` | `.dark` 类挂在 body 上，全局生效 |
| `bg_element` | `target.parentElement` | 背景色设到 `.gradio-container` 上 |

**嵌入模式 DOM 树**：
```
<gradio-app>                           ← .dark 挂在这里（target.parentElement）
└── <div class="gradio-container gradio-container-xxx embed-container">  ← target（wrapper）
    └── <div class="main ...">         ← bg_element，背景色设到这里
        └── <Blocks ... />
```

| 变量 | 值 | 说明 |
|------|----|------|
| `target` | `<div class="gradio-container ...">` | `wrapper` 绑定到 `<Embed>` 的最外层 div |
| `dark_class_element` | `<gradio-app>` 自定义元素 | `.dark` 类挂在自定义元素上，不污染宿主 body |
| `bg_element` | `target`（`.gradio-container`） | 背景色设到容器自身 |

#### CSS 选择器匹配逻辑

主题 CSS 中的暗模式选择器是：
```css
:root.dark, :root .dark {
  --body-background-fill: var(--neutral-950);
  ...
}
```

这是一个逗号分隔的复合选择器，包含两个部分：
- `:root.dark`：匹配本身带有 `.dark` 类的根元素（`<html>`）
- `:root .dark`：匹配根元素的**后代**中带有 `.dark` 类的任何元素

**两种模式的匹配路径**：
- **独立模式**：`.dark` 挂在 `document.body` 上，body 是 `:root`（`<html>`）的后代 → 匹配 `:root .dark`
- **嵌入模式**：`.dark` 挂在 `<gradio-app>` 自定义元素上，它也是 `:root` 的后代 → 匹配 `:root .dark`

**巧妙之处**：无论 `.dark` 类挂在 DOM 树的哪个位置（只要在 `<html>` 之内），`":root .dark"` 这个后代选择器都能匹配并触发暗模式变量覆写。

#### 模式优先级与切换流程

**优先级链**（L122-L137）：
```
theme_mode prop（Svelte 组件传入）
    > URL ?__theme=dark|light
        > "system"（默认）→ matchMedia("(prefers-color-scheme: dark)")
```

**完整切换流程**（[handle_theme_mode](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L228-L248）：
```javascript
function handle_theme_mode(target: HTMLDivElement): "light" | "dark" {
    // 1) 网站模式强制 light（用于 gradio.app 官网展示）
    const force_light = window.__gradio_mode__ === "website";
    if (force_light) new_theme_mode = "light";
    else {
        // 2) 读 URL 参数 ?__theme=
        const url_color_mode = url.searchParams.get("__theme");
        // 3) 优先级: prop > URL > system
        new_theme_mode = theme_mode || url_color_mode || "system";
    }

    if (new_theme_mode === "dark" || new_theme_mode === "light") {
        apply_theme(target, new_theme_mode);  // 直接应用
    } else {
        new_theme_mode = sync_system_theme(target);  // 监听系统主题变化
    }
    return new_theme_mode;
}

function sync_system_theme(target): "light" | "dark" {
    // 初始匹配
    function update_scheme() {
        const _theme = matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light";
        apply_theme(target, _theme);
        return _theme;
    }
    const theme = update_scheme();
    // 监听变化
    matchMedia("(prefers-color-scheme: dark)").addEventListener("change", update_scheme);
    return theme;
}
```

#### theme-loaded 类的防闪屏作用（仅独立模式）

`.theme-loaded` 类是 SSR 防闪屏机制的关键，**仅在独立模式下工作**，嵌入模式完全不参与。

**防闪屏 CSS**（[+layout.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/+layout.svelte) L12-L21）：
```css
:global(body) {
    background: var(--body-background-fill);
    color: var(--body-text-color);
}

@media (prefers-color-scheme: dark) {
    :global(body:not(.theme-loaded)) {
        background: var(--neutral-950);
    }
}
```

**独立模式时序**：
1. 浏览器加载 SSR 输出的 HTML，此时 JS 尚未执行，body 上没有任何类
2. 若系统是深色模式，`body:not(.theme-loaded)` 生效，body 背景强制设为 `--neutral-950`（深色），避免白色闪烁
3. 同时 `body` 的默认背景是 `var(--body-background-fill)`，但此时主题 CSS 变量可能还未加载完成
4. JS 加载完成，模块初始化时调用 `handle_theme_mode(document.body)` → 触发 `apply_theme()`
5. `apply_theme()` 给 `document.body` 加上 `.theme-loaded` 和 `.dark` 两个类
6. `.theme-loaded` 加上后，`:not(.theme-loaded)` 伪类不匹配，兜底深色背景规则失效
7. 此时 `/theme.css` 已加载完成，主题 CSS 中的 `:root .dark` 规则接管，暗模式变量正常生效
8. 背景色平滑过渡，用户看不到白屏闪烁

**嵌入模式为什么不参与**：
1. 嵌入模式是纯客户端渲染，没有 SSR 输出的 HTML，不存在"先看到 SSR 白色背景"的问题
2. 嵌入模式的 `apply_theme()` 没有添加 `.theme-loaded` 类的代码（L269-278）
3. 嵌入模式使用 `css_ready` 机制，`Blocks` 组件在 `css_ready === true` 时才渲染，从源头避免 FOUC
4. `+layout.svelte` 属于 SvelteKit 项目，嵌入模式（SPA）根本不会加载这个文件

---

### 11.5 prefix_css：CSS 作用域隔离机制

[prefix_css()](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/css.ts) L39-L119 是嵌入模式下防止样式污染宿主页面的核心机制。

#### 启用条件（特性检测）

```javascript
let supports_adopted_stylesheets = false;
if (
    typeof window !== "undefined" &&
    "attachShadow" in Element.prototype &&
    "adoptedStyleSheets" in Document.prototype
) {
    const shadow_root_test = document.createElement("div").attachShadow({ mode: "open" });
    supports_adopted_stylesheets = "adoptedStyleSheets" in shadow_root_test;
}
```

必须同时支持 **Shadow DOM** 和 **adoptedStyleSheets** 才启用。不支持时 `prefix_css` 直接返回原字符串（L44）。

#### 核心处理逻辑

对于传入的 CSS 字符串，`prefix_css` 做了四件事：

**1. @import 抽离前置**（L53-L57）：
```javascript
let importString = "";
string = string.replace(/@import\s+url\((.*?)\);\s*/g, (match, url) => {
    importString += `@import url(${url});\n`;
    return "";
});
```
CSS 语法要求 `@import` 必须在最前面，所以先抽出来。

**2. 构造前缀**（L62）：
```javascript
let gradio_css_infix = `.gradio-container.gradio-container-${version} .contain `;
```
版本号用于多版本共存时的隔离。

**3. 按规则类型遍历处理**（L64-L117）：

| 规则类型 | 处理方式 | 示例 |
|----------|---------|------|
| **CSSStyleRule**（普通样式） | 每个选择器前加前缀；`.dark` 移到最前面 | `.dark .foo` → `.dark .gradio-container.x .contain .foo` |
| **CSSMediaRule**（媒体查询） | 内部规则同样加前缀，保留媒体查询 | `@media (max-width:600px) { .foo {...} }` → 包裹前缀版本 |
| **CSSKeyframesRule**（关键帧） | 原样保留，不做前缀 | `@keyframes spin { ... }` 直接输出 |
| **CSSFontFaceRule**（字体） | 原样保留 | `@font-face { ... }` 直接输出 |

**4. 普通选择器前缀注入**（L68-L82）：
```javascript
const selector = rule.selectorText;
const new_selector = selector
    .replace(".dark", "")          // 先去掉 .dark
    .split(",")                    // 多选择器拆分
    .map((s) =>
        `${is_dark_rule ? ".dark" : ""} ${gradio_css_infix} ${s.trim()} `
    )                               // 每个选择器前加前缀，.dark 移到最前
    .join(",");
css_string += rule.cssText;          // 保留原始规则（向后兼容）
css_string += rule.cssText.replace(selector, new_selector);  // 追加前缀版本
```

**注意**：原始规则和前缀版本都会输出，这是为了向后兼容。

#### 效果示例

输入 CSS：
```css
.dark .foo { color: red; }
.bar { font-size: 14px; }
```

输出 CSS（version=4.0.0）：
```css
/* 原始规则保留 */
.dark .foo { color: red; }
.bar { font-size: 14px; }
/* 前缀版本追加 */
.dark .gradio-container.gradio-container-4.0.0 .contain .foo { color: red; }
.gradio-container.gradio-container-4.0.0 .contain .bar { font-size: 14px; }
```

这样用户自定义的 CSS 只会作用于 `.gradio-container` 内部，不会外泄到宿主页面。

#### 调用时机与位置

`prefix_css` 在两种模式下都被调用，但调用方式和注入路径不同：

**独立模式**（[Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/Blocks.svelte) L463-L469）：
```svelte
<svelte:head>
    {#if css}
        {@html `<style>${prefix_css(css, version)}</style>`}
    {/if}
</svelte:head>
```
- 只传 2 个参数，`style_element` 为 `undefined`
- 内部临时创建 `<style>` 解析 CSSOM，解析完 `remove()`
- 返回字符串由 `{@html}` 直接写入 `<style>` 标签

**嵌入模式**（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L139-L148）：
```javascript
if (css_string) {
    if (!css_text_stylesheet) {
        css_text_stylesheet = document.createElement("style");
        document.head.appendChild(css_text_stylesheet);
    }
    css_text_stylesheet.textContent = prefix_css(
        css_string, version, css_text_stylesheet
    );
}
```
- 传 3 个参数，传入已创建的 `<style>` 元素
- `prefix_css` 内部先 `style_element.remove()` 从 DOM 中移除，避免解析过程中样式泄漏
- 返回字符串赋值给 `style_element.textContent`，但元素已不在 DOM 中？实际上由于 Svelte 的 reactivity，赋值后元素可能被重新加入，或者这里存在设计上的冗余。

**⚠️ 嵌入模式本地 stylesheets Bug**（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte) L164-L168）：
```javascript
return fetch(config.root + "/" + stylesheet)
    .then((response) => response.text())
    .then((css_string) => {
        prefix_css(css_string, version);  // 返回值被丢弃！没有写入任何 <style>
    });
```
`prefix_css` 的返回值没有被赋值给任何元素的 `textContent`，也没有创建新的 `<style>` 标签。相对路径的 stylesheets 在嵌入模式下**实际上不会生效**。

---

### 11.6 SSR 与预计算主题值

后端在 `get_config()` 时会预计算 4 个关键主题变量的字面量，用于 SSR：

[blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/blocks.py) L2405-L2416：
```python
"body_css": {
    "body_background_fill": self.theme._get_computed_value("body_background_fill"),
    "body_text_color": self.theme._get_computed_value("body_text_color"),
    "body_background_fill_dark": self.theme._get_computed_value("body_background_fill_dark"),
    "body_text_color_dark": self.theme._get_computed_value("body_text_color_dark"),
}
```

调用 `_get_computed_value()` 递归解析所有 `*引用`，得到最终的颜色字面量（如 `"#ffffff"`）。

这部分数据目前主要用于：
1. **嵌入卡片的背景色预览**：`<Embed>` 组件可在主题加载前用这些值设置占位背景
2. **SSR 模板内联**：供服务端渲染时直接内联到 `<body style="...">` 上

目前 SvelteKit 的实现中，`body_css` 实际上**没有被直接使用**，而是通过 `+layout.svelte` 中的 CSS 变量方式设置背景色。这是因为 CSS 变量方案更灵活，支持运行时切换。

---

### 11.7 嵌入专属主题变量：embed_radius

主题系统中专门有一个变量供嵌入模式使用：

[base.py](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/gradio/themes/base.py) L536, L1088：
```python
self.embed_radius = embed_radius or getattr(self, "embed_radius", "*radius_sm")
```

在 [Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/Embed.svelte) L194-L198 中使用：
```css
.embed-container {
    margin: var(--size-4) 0px;
    border: 1px solid var(--button-secondary-border-color);
    border-radius: var(--embed-radius);  /* 专属圆角变量 */
}
```

当 `display=true`（嵌入且有边框时），`.embed-container` 会应用这个圆角。这就是为什么主题变量列表中有一个看似孤立的 `embed_radius`。

---

### 11.8 两种模式链路总结对比表

| 环节 | 独立模式（App） | 嵌入模式（SPA） |
|------|-----------------|-----------------|
| **入口** | SvelteKit `/[...catchall]` 路由 | `<gradio-app>` 自定义元素 |
| **`is_embed` 默认** | `false` | `true` |
| **基础样式加载** | 构建时 import 打包进应用（`+layout.svelte` L1-L6） | main.ts 启动时 `mount_css(ENTRY_CSS)` + `mount_css(FONTS)` |
| **主题 CSS `/theme.css`** | `<svelte:head>` 静态 `<link>`（`+page.svelte` L423） | `mount_css()` 动态 `<link>`（`Index.svelte` L150） |
| **Google Fonts** | `<svelte:head>` 静态 `<link>`（`+page.svelte` L424-L428） | `mount_css()` 动态 `<link>`（`Index.svelte` L160-L161） |
| **本地 stylesheets** | 仅加载绝对 URL，相对 URL 被 `{#if}` 过滤（`+page.svelte` L426） | 所有 URL 都 fetch，但相对 URL 的 `prefix_css` 结果被丢弃（Bug，L164-L168） |
| **用户自定义 CSS** | `<Blocks>` 内部 `<svelte:head>` + `{@html}` 内联（`Blocks.svelte` L463-L469） | `Index.svelte` 手动创建 `<style>` 写入 `textContent`（`Index.svelte` L139-L148） |
| **`.dark` 挂载点** | `document.body`（`+page.svelte` L159） | `<gradio-app>` 自定义元素（`Index.svelte` L270） |
| **背景色设置** | `target.parentElement`（`.gradio-container`） | `target`（`.main` 内部 div） |
| **`.theme-loaded`** | 有，`apply_theme` 中添加（`+page.svelte` L162） | 无，`apply_theme` 中缺失对应代码 |
| **防闪屏机制** | `+layout.svelte` 中 `@media` 兜底 + `body:not(.theme-loaded)` | 无，使用 `css_ready` 机制延迟渲染避免 FOUC |
| **`css_ready` 作用** | 标记变量，不阻塞渲染（`+page.svelte` L255, L299） | `<Blocks>` 渲染前置条件，等 CSS 加载完才渲染（`Index.svelte` L595） |
| **CSS 作用域隔离** | `prefix_css()` 加前缀（`Blocks.svelte` L468） | `prefix_css()` 加前缀（`Index.svelte` L144） |
| **CDN 适配** | SvelteKit 构建时处理 | `mount_css()` 运行时判断 origin，自动转绝对 URL（`css.ts` L16-L21） |
| **SSR 支持** | 完整支持 | 不适用（客户端渲染） |

### 关键理解点（修正版）

1. **两套 Svelte 入口 + 一套共享组件**：Gradio 有两个前端入口——`js/app`（SvelteKit SSR，独立模式）和 `js/spa`（纯客户端 SPA，嵌入模式），各自有独立的主题加载逻辑，但共享 `@gradio/core` 中的 `<Blocks>`、`prefix_css()`、`mount_css()` 等核心组件和工具。

2. **CSS 变量的跨模式复用**：主题 CSS 中的 `:root.dark, :root .dark` 选择器设计非常巧妙——`.dark` 类挂在 body 上匹配 `:root .dark`，挂在自定义元素上也匹配 `:root .dark`，同一套 CSS 适配两种挂载模式。

3. **用户自定义 CSS 的两条注入路径**：
   - 独立模式：`<Blocks>` 组件内部通过 `<svelte:head>` + `{@html}` 注入
   - 嵌入模式：`Index.svelte` 中手动创建 `<style>` 元素并设置 `textContent`

4. **⚠️ 嵌入模式本地 stylesheets Bug**：相对路径的 stylesheets 在 `fetch` 后调用 `prefix_css()` 但返回值被丢弃，没有写入任何 `<style>` 标签，**实际上不会生效**。这是一个需要修复的代码 Bug。

5. **⚠️ prefix_css 中的冗余 remove()**：`prefix_css()` 内部会先调用 `style_element.remove()` 将传入的 `<style>` 从 DOM 中移除，但之后将处理后的字符串赋值给 `style_element.textContent`。在支持 `adoptedStyleSheets` 的浏览器中，元素已不在 DOM 中，样式可能无法生效。这可能是历史遗留的设计冗余。

6. **两种防闪屏机制**：
   - 独立模式：**`.theme-loaded` 类 + 媒体查询兜底** —— SSR 输出的 HTML 先显示深色背景，JS 执行后加 `.theme-loaded` 解除兜底
   - 嵌入模式：**`css_ready` 延迟渲染** —— 等 CSS 全部加载完成后才渲染 `<Blocks>` 组件，从源头避免 FOUC

7. **CDN 适配的关键**：`mount_css()` 中的 origin 判断是嵌入模式下的隐形基础设施——当 Gradio 静态资源从 CDN 提供时，能正确把 `/theme.css` 转成 `https://cdn.xxx.com/theme.css`。

8. **prefix_css 的双重输出**：同时输出原始规则和前缀版本，保证了向后兼容——旧的自定义 CSS 即使不做前缀也能工作，同时新的前缀版本确保不污染宿主页面。

---

### 附：已确认的代码 Bug 列表

| Bug 位置 | 问题描述 | 影响范围 |
|---------|---------|---------|
| [Index.svelte L164-L168](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/spa/src/Index.svelte#L164-L168) | 相对路径 stylesheets 的 `prefix_css` 返回值被丢弃，没有写入 DOM | 嵌入模式下本地自定义样式表不生效 |
| [+page.svelte L424-L428](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/app/src/routes/[...catchall]/+page.svelte#L424-L428) | 相对路径 stylesheets 被 `{#if}` 过滤，完全不加载 | 独立模式下本地自定义样式表不生效 |
| [css.ts L48](file:///d:/fz/0601/solo-dogfeeding/code/246-gradio/js/core/src/css.ts#L48) | `prefix_css` 内部 `style_element.remove()` 可能导致支持 `adoptedStyleSheets` 的浏览器中样式不生效 | 两种模式下自定义 CSS 可能失效 |

> **说明**：独立模式下相对路径 stylesheets 被忽略可能是有意设计（SvelteKit 有自己的静态资源处理机制），但嵌入模式下的 Bug 明显是代码遗漏。
