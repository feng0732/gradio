# DataFrame 组件编辑后的数据回传机制

## 一、整体数据流向

### 1.1 数据流转全链路

```
后端初始化（postprocess 序列化）
        ↓  headers + data + metadata
前端接收 props.value
        ↓
values 状态 ←→ row_data（cast 转换） ← display_value（metadata）
        ↓
前端交互（编辑/增删行列）
        ↓
push_change() 触发 onchange
        ↓  data + headers + metadata: null
Index.svelte handle_change() 去重分发 change 事件
        ↓
后端 preprocess() 接收 DataframeData 载荷
        ↓
转换为 pandas / numpy / polars / list 格式
```

### 1.2 核心文件与角色

| 文件 | 角色 | 关键功能 |
|------|------|----------|
| [dataframe.py](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py) | 后端组件 | `preprocess()` 反序列化、`postprocess()` 序列化、`set_auto_datatype()` 类型推断 |
| [Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte) | 前端核心表 | 编辑处理、行列增删、`push_change()` 触发回传 |
| [EditableCell.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/EditableCell.svelte) | 单元格编辑 | 文本/布尔编辑、blur 事件提交变更 |
| [utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts) | 类型工具 | `cast_value_to_type()` 前端类型转换 |
| [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte) | 前端包装 | 事件分发、datatype 对齐、change 去重 |

---

## 二、序列化初始过程（postprocess → 前端）

### 2.1 组件初始化调用链

**后端初始化入口** 在 Component 基类构造函数中调用 ([base.py#L207-L219](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/base.py#L207-L219)):

```python
# 1. 解析初始值或调用 load_fn
load_fn, initial_value = self.get_load_fn_and_initial_value(value, inputs)

# 2. 调用 postprocess 进行序列化
initial_value = self.postprocess(initial_value)

# 3. 转为 JSON 字典（DataframeData 是 GradioModel，走 model_dump）
if isinstance(initial_value, BaseModel):
    initial_value = initial_value.model_dump()

# 4. 移动文件到缓存（dataframe 无文件资源，直接返回）
self.value = move_files_to_cache(
    initial_value,
    self,
    postprocess=True,
    keep_in_cache=True,
)
```

> **关键点**：postprocess 在组件构造时就会被调用，结果存储在 `self.value` 中，通过 config 传递给前端作为初始 props.value。

### 2.2 postprocess 详细流程

**`postprocess()` 方法** ([dataframe.py#L461-L517](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L461-L517)):

```python
def postprocess(self, value) -> DataframeData:
    # 1. 获取表头（优先从数据中取，否则用组件配置的 headers）
    headers = self.get_headers(value) or self.headers
    
    # 2. 获取单元格数据
    data = [] if self.is_empty(value) else self.get_cell_data(value)
    
    # 3. 空数据直接返回
    if len(data) == 0:
        return DataframeData(headers=headers, data=[], metadata=None)
    
    # 4. 表头与数据列数对齐
    if len(headers) > len(data[0]):
        headers = headers[: len(data[0])]       # 表头多 → 截断
    elif len(headers) < len(data[0]):
        headers = [
            *headers,
            *[str(i) for i in range(len(headers) + 1, len(data[0]) + 1)],
        ]                                     # 表头少 → 用数字序号补齐
    
    # 5. 获取 metadata（Styler 自动生成，或 dict 直接携带，其他为 None）
    metadata = self.get_metadata(value)
    
    return DataframeData(
        headers=headers,
        data=data,
        metadata=metadata,
    )
```

### 2.3 metadata 的来源

**`get_metadata()` 方法是所有 metadata 的统一入口** ([dataframe.py#L436-L459](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L436-L459)):

```python
@staticmethod
def get_metadata(value):
    from pandas.io.formats.style import Styler

    if isinstance(value, Styler):
        return Dataframe.__extract_metadata(
            value, [int(c) for c in getattr(value, "hidden_columns", [])]
        )
    elif isinstance(value, dict):
        return value.get("metadata", None)
    return None
```

**两大来源的处理方式**：

1. **来源 1：pandas Styler 对象** → 调用 `__extract_metadata()` 自动生成
   - 代码位置: [dataframe.py#L609-L639](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L609-L639)
   - 内部调用 pandas `_compute()._translate()` 渲染样式
   - 提取 `display_value`（格式化字符串）和 `styling`（CSS 样式）

2. **来源 2：Python dict 输入** → 直接读取 `metadata` 键
   - 代码位置: [dataframe.py#L457-L458](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L457-L458)
   - 用户可主动提供 `metadata.display_value` 和 `metadata.styling`
   - 格式与 Styler 生成的完全一致

> **重要结论**：
> - **metadata 两大来源**：1) pandas Styler 对象自动生成；2) Python dict 输入直接携带 `metadata` 键
> - metadata 包含两个键：`display_value`（二维字符串数组）和 `styling`（二维 CSS 字符串数组）
> - 普通 DataFrame / list / numpy / polars / str 的 metadata 均为 `None`
> - 第三章详细分析四大场景（Styler、dict、普通数据、编辑回传）对 display_value 的影响

---

## 三、metadata 来源边界与 display_value 保留关系

### 3.1 三层数据模型

前端表格内部存在三层数据表示，各自用途不同、来源不同：

| 层级 | 变量 | 类型 | 来源 | 用途 | 编辑后是否保留 |
|------|------|------|------|------|---------------|
| L1 原始值 | `values` | `CellValue[][]` | 后端 `data` 字段 | 编辑、回传后端 | ✅ 是（直接修改） |
| L2 类型转换值 | `row_data` | `GradioRow[]` | `values` + `cast_value_to_type` | 展示、排序、过滤 | ✅ 是（派生自 values） |
| L3 格式化显示值 | `display_value` | `string[][] \| null` | 后端 `metadata.display_value` | 非编辑模式展示 | ❌ 否（编辑后丢失） |

### 3.2 metadata 的四大来源场景

**`get_metadata()` 方法**是所有 metadata 的入口 ([dataframe.py#L436-L459](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L436-L459)):

```python
@staticmethod
def get_metadata(value):
    from pandas.io.formats.style import Styler

    if isinstance(value, Styler):
        return Dataframe.__extract_metadata(
            value, [int(c) for c in getattr(value, "hidden_columns", [])]
        )
    elif isinstance(value, dict):
        return value.get("metadata", None)  # ← dict 直接取 metadata 键
    return None
```

四种输入场景的 metadata 来源与结果：

| 场景 | 输入类型 | metadata 来源 | metadata 结果 | display_value 状态 |
|------|----------|--------------|--------------|-------------------|
| 场景 1 | pandas Styler | `__extract_metadata()` 生成 | `{"display_value": [[...]], "styling": [[...]]}` | ✅ 有值 |
| 场景 2 | Python dict | `value.get("metadata", None)` | dict 中的 metadata（如有） | ✅ 有值（若 dict 提供） |
| 场景 3 | DataFrame/list/numpy/polars/str/csv | 不匹配任何条件，返回 `None` | `None` | ❌ null |
| 场景 4 | 前端编辑回传 | `push_change()` 硬编码 `metadata: null` | `null` | ❌ null |

---

### 3.3 场景 1：Styler 输入的 metadata 路径

**调用链**：

```
用户传入 Styler 对象
    ↓
postprocess(value)
    ↓
get_metadata(value)  → isinstance(value, Styler) == True
    ↓
__extract_metadata(value, hidden_columns)
    ↓
df._compute()._translate(None, None)  → 调用 pandas Styler 内部渲染
    ↓
提取 style_data["body"] 中每格的 display_value 和 styling
    ↓
返回 {"display_value": string[][], "styling": string[][]}
    ↓
DataframeData(metadata=...)
    ↓
前端 props.value.metadata.display_value → Table 的 display_value prop
```

**`__extract_metadata()` 详细流程** ([dataframe.py#L609-L639](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L609-L639)):

```python
@staticmethod
def __extract_metadata(df: Styler, hidden_cols=None) -> dict:
    style_data = df._compute()._translate(None, None)
    cell_styles = style_data.get("cellstyle", [])
    # 1. 构建 cell_id → CSS 样式字符串 的映射
    style_dict = {}
    for style in cell_styles:
        for selector in style.get("selectors", []):
            style_dict[selector] = "; ".join(
                f"{prop}: {v}" for prop, v in style.get("props", [])
            )
    
    metadata = {"display_value": [], "styling": []}
    
    # 2. 遍历 body 行，提取 display_value 和样式
    for row in style_data["body"]:
        row_display = []
        row_styling = []
        cells = [cell for cell in row if cell["type"] == "td"]
        # 过滤隐藏列，保持列索引正确
        cells = [cell for col_idx, cell in enumerate(cells)
                 if col_idx not in hidden_cols_set]
        for cell in cells:
            row_display.append(cell["display_value"])   # 格式化显示值
            row_styling.append(style_dict.get(cell["id"], ""))
        metadata["display_value"].append(row_display)
        metadata["styling"].append(row_styling)
    return metadata
```

**对 display_value 的影响**：
- Styler 的 `display_value` 是 pandas 格式化后的字符串（如 `"1.23%"`、`"¥100.00"` 等）
- 非编辑模式下优先显示这些格式化值，而非原始数值
- 编辑模式下切换为原始值（`values` 中的实际数据）
- 编辑后 metadata 丢失，重新显示 `values` 中的值

---

### 3.4 场景 2：dict 输入携带 metadata 的路径

**dict 输入格式**（也是 DataframeData 的结构）：

```python
{
    "headers": ["列A", "列B", "列C"],
    "data": [[1, 2, 3], [4, 5, 6]],
    "metadata": {                          # ← 可选，直接传递
        "display_value": [["1%", "2%", "3%"], ["4%", "5%", "6%"]],
        "styling": [["color:red", "", ""], ["", "color:blue", ""]]
    }
}
```

**完整调用链**：

```
用户传入 dict（含 metadata 键）
    ↓
postprocess(value)
    ↓
get_headers(value)  → value.get("headers", [])     [dataframe.py#L379-L380]
    ↓
get_cell_data(value) → value.get("data", [[]])     [dataframe.py#L403-L404]
    ↓
is_empty(value)     → len(value["data"]) == 0      [dataframe.py#L344-L347]
    ↓
get_metadata(value) → value.get("metadata", None)  [dataframe.py#L457-L458]
    ↓
DataframeData(headers=..., data=..., metadata=...)
    ↓
前端 props.value.metadata.display_value / styling
```

**关键代码位置**：
- headers 提取: [dataframe.py#L379-L380](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L379-L380)
- data 提取: [dataframe.py#L403-L404](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L403-L404)
- metadata 提取: [dataframe.py#L457-L458](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L457-L458)
- 空值判断: [dataframe.py#L344-L347](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L344-L347)

**对 display_value 的影响**：
- dict 提供了 `metadata.display_value` 时，前端行为与 Styler 完全一致
- 非编辑模式显示格式化值，编辑模式显示原始值
- 编辑后 metadata 同样丢失

---

### 3.5 场景 3：普通数据输入（DataFrame/list/numpy/polars/str）

**调用链**：

```
用户传入 DataFrame / list / numpy / polars / csv路径
    ↓
postprocess(value)
    ↓
get_metadata(value)
    ↓  不是 Styler，也不是 dict
返回 None
    ↓
DataframeData(metadata=None)
    ↓
前端 display_value = props.value?.metadata?.display_value ?? null
    ↓
display_value = null
```

**对 display_value 的影响**：
- `display_value` 始终为 `null`
- 所有单元格（无论是否编辑）都直接显示 `values` 中的原始值
- 不存在编辑前/后的显示差异

---

### 3.6 场景 4：编辑回传 metadata: null

**前端 `push_change()` 硬编码丢弃 metadata** ([Table.svelte#L363-L374](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L363-L374)):

```typescript
function push_change(new_values?, new_headers?): void {
    onchange?.({
        data: new_values ?? values,
        headers: (new_headers ?? resolved_headers) as string[],
        metadata: null  // ← 硬编码为 null，丢弃所有 metadata
    });
    // ...
}
```

**触发 push_change 的所有操作**都会丢弃 metadata：

| 操作 | 代码位置 | metadata 状态 |
|------|----------|--------------|
| 单元格文本编辑提交 | [Table.svelte#L470](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L470) | null |
| 布尔列全选切换 | [Table.svelte#L633](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L633) | null |
| 添加行 | [Table.svelte#L567](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L567) | null |
| 添加列 | [Table.svelte#L585](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L585) | null |
| 删除行 | [Table.svelte#L592](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L592) | null |
| 删除列 | [Table.svelte#L608](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L608) | null |
| Delete/Backspace 清空单元格 | [Table.svelte#L794](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L794) | null |
| 应用筛选条件 | [Table.svelte#L659](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L659) | null |
| 文件上传导入 | [Table.svelte#L853](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L853) | null |

**对 display_value 的影响**：
- 一旦用户执行任何上述操作，前端 props.value.metadata 变为 `null`
- display_value 随之变为 `null`，所有单元格切换为显示 `values` 原始值
- Styler 或 dict 提供的格式化显示效果全部消失

---

### 3.7 四大场景对 display_value 影响对比

| 场景 | metadata | display_value | 非编辑模式显示 | 编辑模式显示 | 编辑后状态 |
|------|----------|--------------|---------------|-------------|-----------|
| Styler 输入 | 有（自动生成） | string[][] | 格式化值（display_value） | 原始值（values） | metadata 丢失，全部显示原始值 |
| dict 输入带 metadata | 有（用户提供） | string[][] | 格式化值（display_value） | 原始值（values） | metadata 丢失，全部显示原始值 |
| dict 输入不带 metadata | None | null | 原始值（values） | 原始值（values） | 无变化，始终显示原始值 |
| DataFrame/list/numpy/polars | None | null | 原始值（values） | 原始值（values） | 无变化，始终显示原始值 |
| 编辑回传后（任何场景） | null | null | 原始值（values） | 原始值（values） | 已丢失，等后端重新计算 |

### 3.8 display_value 的使用优先级

**EditableCell 中的显示逻辑** ([EditableCell.svelte#L90-L92](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/EditableCell.svelte#L90-L92)):

```typescript
let display_content = $derived(
    editable ? value : display_value !== null ? display_value : value
);
```

**Table.svelte 中的 get_display_value 函数** ([Table.svelte#L353-L357](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L353-L357)):

```typescript
function get_display_value(row: number, col: number): string {
    if (display_value?.[row]?.[col] !== undefined)
        return display_value[row][col];
    return String(values?.[row]?.[col] ?? "");
}
```

**显示优先级规则**（从高到低）：
1. **编辑模式（editable=true）**：始终使用 `value`（row_data 中的类型转换值）
2. **展示模式 + display_value 非 undefined**：使用 `display_value`（metadata 中的格式化值）
3. **展示模式 + display_value 为 null/undefined**：使用 `values`（原始值，转字符串）
4. **空字符串支持**（v0.17.7+，PR #11033）：通过 `!== undefined` 判断，允许 display_value 为空字符串

### 3.9 metadata 生命周期

| 阶段 | metadata 状态 | 原因 | display_value 效果 |
|------|------------|------|-------------------|
| 初始加载（Styler/dict） | 有 | 后端 postprocess 生成/提取 | 非编辑模式显示格式化值 |
| 初始加载（普通数据） | None | get_metadata 返回 None | 始终显示原始值 |
| 任何编辑操作后 | null | push_change 设为 null | 全部显示原始值 |
| 下一轮后端返回 | 重新生成/提取 | 后端重新 postprocess | 恢复格式化显示 |

> **设计意图**：metadata 是后端根据完整数据计算的派生数据（样式、格式化等），前端无法在数据变更后正确维护 metadata，因此编辑后直接丢弃，等待后端根据新数据重新计算。

---

## 四、原始编辑值与展示类型转换的界限

### 4.1 三层数据的职责划分

```
values (原始编辑层)
    │  存储类型：混合 string + boolean (bool 列)
    │  用于：编辑写入、回传后端
    │  写入入口：handle_blur / handle_select_all / add_row / add_col
    ▼
row_data (类型转换层)
    │  派生方式：$derived(values → cast_value_to_type)
    │  存储类型：CellValue（number / boolean / string / null / undefined）
    │  用于：TanStack Table 内部、排序、过滤、BooleanCell 渲染
    ▼
display_value (格式化显示层)
       来源：metadata.display_value（Styler 自动生成 或 dict 输入携带）
       存储类型：string[][] | null
       用于：非编辑模式下的格式化展示
```

### 4.2 values：原始编辑层

**存储位置**：`values` 响应式变量 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte))

**数据类型**：`CellValue = string | number | boolean | null | undefined`

**值的来源与更新**：

| 操作 | 值的类型 | 代码位置 |
|------|----------|----------|
| 文本编辑提交 | `string` | [Table.svelte#L456-L464](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L456-L464) |
| 布尔列点击切换 | `boolean` | [Table.svelte#L626-L634](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L626-L634) |
| 新增行/列填充 | `""` (空字符串) | [Table.svelte#L561](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L561) |

**关键特性**：
- **直接回传后端**：push_change 直接使用 values，不经过类型转换
- **混合类型**：bool 列存 boolean，其他列存 string，values 数组是混合类型的
- **不可变更新**：每次修改都创建新数组（`values.map(r => [...r])`）

### 4.3 row_data：类型转换层

**派生方式**：`$derived` 响应式派生 ([Table.svelte#L117-L126](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L117-L126)):

```typescript
let row_data: GradioRow[] = $derived(
    (values ?? []).map((row, i) => {
        const obj: GradioRow = { _index: i };
        (row ?? []).forEach((val, j) => {
            const dtype = Array.isArray(datatype) ? datatype[j] : datatype;
            obj[`col_${j}`] = cast_value_to_type(val, dtype);
        });
        return obj;
    })
);
```

**转换函数**：`cast_value_to_type()` ([utils.ts#L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)):

| datatype | 转换方式 | 失败回退 |
|----------|----------|----------|
| `"str"` | 不转换，直接返回 | - |
| `"number"` | `Number(v)` | isNaN → 返回原值（string） |
| `"bool"` | 多种方式：boolean 原值 / number 非零 / 字符串 "true"/"1"/"false"/"0" | 不匹配 → 返回原值 |
| `"date"` | `new Date(v).toISOString()` | 无效日期 → 返回原值 |
| `"markdown"`, `"html"`, `"image"` | 不转换，直接返回 | - |

**使用场景**：
- TanStack Table 的 `data` 源（排序、过滤都基于 row_data）
- 布尔列渲染条件（`typeof value === "boolean"`）
- 数字列排序（按数值排序而非字典序）

### 4.4 界限总结

| 维度 | values（原始层） | row_data（转换层） | display_value（格式化层） |
|------|-----------------|-------------------|------------------------|
| 写入方向 | ✅ 可写 | ❌ 只读派生 | ❌ 只读 |
| 回传后端 | ✅ 直接使用 | ❌ 不使用 | ❌ 不使用 |
| 类型准确性 | 原始输入，未转换 | 尽力转换，可能失败 | 完全格式化 |
| 触发变更 | 用户交互 | values 变化 | 后端返回 |
| 数据生命周期 | 整个会话 | 每次渲染重新计算 | 编辑后丢失 |

> **核心界限**：
> - **写入** 始终操作 `values`
> - **展示** 使用 `row_data` 或 `display_value`
> - **回传** 使用 `values`
> - 类型转换是**只读副作用**，不污染原始值

---

## 五、类型推断机制

### 5.1 前端类型转换（cast_value_to_type）

**`cast_value_to_type()` 函数** ([utils.ts#L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)):

```typescript
export function cast_value_to_type(v: any, t: Datatype): CellValue {
    if (v === null || v === undefined) return v;
    if (t === "number") {
        const n = Number(v);
        return isNaN(n) ? v : n;
    }
    if (t === "bool") {
        if (typeof v === "boolean") return v;
        if (typeof v === "number") return v !== 0;
        const s = String(v).toLowerCase();
        if (s === "true" || s === "1") return true;
        if (s === "false" || s === "0") return false;
        return v;
    }
    if (t === "date") {
        const d = new Date(v);
        return isNaN(d.getTime()) ? v : d.toISOString();
    }
    return v;
}
```

> **设计要点**：类型转换是"尽力而为"的，转换失败时保持原值不变，而非抛出错误。

### 5.2 后端自动类型推断（set_auto_datatype）

**`set_auto_datatype()` 方法** ([dataframe.py#L519-L606](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L519-L606)):

**类型映射表**：
```python
dtype_mapping = {
    "str": "str", "object": "str", "string": "str", "utf": "str",
    "int": "number", "float": "number",
    "bool": "bool", "boolean": "bool",
    "datetime": "date", "date": "date", "timedelta": "date", "timestamp": "date",
    "category": "str", "categorical": "str",
}
```

**推断规则**（按输入类型）：

| 输入类型 | 类型来源 | 备注 |
|----------|----------|------|
| pandas DataFrame | `df.dtypes` | 每列独立推断 |
| pandas Styler | `value.data.dtypes` | 取内部 DataFrame 的 dtypes |
| numpy ndarray | `type(value[0, i]).__name__` | 检查第一行各列的 Python 类型 |
| Python list | `type(val).__name__` | 检查第一行各元素的 Python 类型 |
| polars DataFrame | `value.dtypes` | 每列独立推断 |

**类型名称清洗**：
1. 正则 `\[.*?\]|\(.*?\)` → 移除括号及内容（如 `int64` → `int`, `DatetimeIndex` → 保留）
2. 正则 `\s*\d+\s*$` → 移除尾部数字（如 `datetime64` → `datetime`）
3. 转小写 → 查表，未匹配默认 `"str"`

### 5.3 前端 datatype 对齐

**`aligned_datatype` 派生值** ([Index.svelte#L24-L39](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L24-L39)):

```typescript
let aligned_datatype = $derived.by(() => {
    const dt = gradio.props.datatype;
    if (!Array.isArray(dt)) return dt;

    const config_headers: string[] | undefined = (gradio.props as any).headers;
    const current_headers = gradio.props.value?.headers;
    if (!config_headers || !current_headers) return dt;

    // 建立 表头名 → datatype 的映射
    const map = new Map<string, string>();
    for (let i = 0; i < Math.min(config_headers.length, dt.length); i++) {
        map.set(config_headers[i], dt[i]);
    }
    
    // 按当前表头顺序重新排列 datatype
    return current_headers.map(
        (h: string, i: number) => map.get(h) ?? dt[i] ?? "str"
    );
});
```

> **设计目的**：当列被隐藏、重排时，通过表头名称映射保持 datatype 与列的对应关系，而非依赖位置索引。

---

## 六、行列变更处理逻辑

### 6.1 配置模式：fixed vs dynamic

**行列配置格式**：
```typescript
type CountConfig = [number, "fixed" | "dynamic"];
```

| 模式 | 含义 | 增删操作 |
|------|------|----------|
| `[n, "fixed"]` | 固定 n 行/列 | 禁止增删，静默忽略 |
| `[n, "dynamic"]` | 初始 n 行/列 | 允许增删 |

**static_columns 特殊处理** ([standalone/Index.svelte#L92-L104](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/standalone/Index.svelte#L92-L104)):

```typescript
let resolved_col_count = $derived.by(() => {
    if (
        static_columns &&
        static_columns.length > 0 &&
        resolved_col_count_base[1] !== "fixed"
    ) {
        return [resolved_col_count_base[0], "fixed"];  // 有静态列时强制 fixed
    }
    return resolved_col_count_base;
});
```

> **注意**：设置 `static_columns` 会自动将列模式转为 `"fixed"`，禁止列增删。

### 6.2 添加行（add_row）

**`add_row()` 函数** ([Table.svelte#L556-L570](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L556-L570)):

```typescript
function add_row(index?: number): void {
    if (row_count[1] !== "dynamic") return;
    
    const col_len = values[0]?.length || resolved_headers.length || 1;
    const new_row: CellValue[] = Array(col_len).fill("");
    
    const new_values = [...values];
    if (index !== undefined) {
        new_values.splice(index, 0, new_row);  // 插入指定位置
    } else {
        new_values.push(new_row);              // 追加到末尾
    }
    
    values = new_values;
    push_change(new_values);
    
    selected = [index ?? new_values.length - 1, 0];
    parent?.focus();
}
```

### 6.3 添加列（add_col）

**`add_col()` 函数** ([Table.svelte#L572-L587](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L572-L587)):

```typescript
function add_col(index?: number): void {
    if (col_count[1] !== "dynamic") return;
    
    // 新表头名：Header N（自动递增）
    const new_headers = [
        ...(headers ?? []),
        `Header ${(headers?.length ?? 0) + 1}`
    ];
    
    // 每行追加空字符串
    const new_values = values.map((row) => [...row, ""]);
    
    if (index !== undefined) {
        // 将刚添加的（末尾）移动到指定位置
        new_headers.splice(index, 0, new_headers.pop()!);
        new_values.forEach((row) => row.splice(index, 0, row.pop()!));
    }
    
    values = new_values;
    headers = new_headers;
    push_change(new_values, new_headers);
    parent?.focus();
}
```

### 6.4 删除行（delete_row_at）

**`delete_row_at()` 函数** ([Table.svelte#L589-L595](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L589-L595)):

```typescript
function delete_row_at(index: number): void {
    if (values.length <= 1) return;  // 至少保留一行
    
    values = [...values.slice(0, index), ...values.slice(index + 1)];
    push_change(values);
    
    active_cell_menu = null;
    active_header_menu = null;
}
```

### 6.5 删除列（delete_col_at）

**`delete_col_at()` 函数** ([Table.svelte#L597-L614](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L597-L614)):

```typescript
function delete_col_at(index: number): void {
    if (col_count[1] !== "dynamic") return;
    if ((values[0]?.length ?? 0) <= 1) return;  // 至少保留一列
    
    // 每行移除指定列
    values = values.map((row) => [
        ...row.slice(0, index),
        ...row.slice(index + 1)
    ]);
    
    // 表头也移除对应位置
    headers = [
        ...(headers ?? []).slice(0, index),
        ...(headers ?? []).slice(index + 1)
    ];
    
    push_change(values, headers as string[]);
    
    active_cell_menu = null;
    active_header_menu = null;
    selected = false;
    selected_cells = [];
    editing = false;
}
```

### 6.6 增删列后 datatype 对齐的异常情况

#### 6.6.1 正常情况（config_headers 存在）

当 `config_headers`（即组件初始化时的 `headers` prop）存在时，`aligned_datatype` 通过表头名映射工作。

**删除列**：✅ 正常
- 被删列的 datatype 自然消失
- 剩余列通过名称正确对应，不会错位

**添加列**：⚠️ 类型回退
- 新列名 `"Header N"` 不在 `config_headers` 映射中
- 第一层回退：`dt[i]`（按位置取原始 datatype）
- 第二层回退：`"str"`（超出原始列数时）
- 结果：新增列始终是 `str` 类型

```
示例：
初始：headers=["A","B"], datatype=["number","bool"]
新增一列后：headers=["A","B","Header 3"]
aligned_datatype = ["number", "bool", "str"]
                              ↑ 第三列回退到 str
```

#### 6.6.2 异常1：config_headers 缺失

**触发条件**：`config_headers` 或 `current_headers` 任一不存在时
- 未设置 `headers` prop（组件配置时 headers 为 undefined）
- value.headers 为空

**行为**：直接返回 `dt`（原始 datatype 数组），按位置对齐

**后果**：
- 删除列后，所有后续列的 datatype 错位
- 新增列后，datatype 长度与列数不匹配

```
示例：
初始：datatype=["number","bool","date"]
删除第1列后：
aligned_datatype = ["number", "bool"]  ← 按位置取 dt[0], dt[1]
实际应该是：["bool", "date"]           ← 错误！全部错位
```

#### 6.6.3 异常2：表头重名

**触发条件**：两列或多列表头名称相同

**行为**：`Map.set()` 后写入的覆盖先写入的

```
示例：
config_headers = ["value", "value"]
datatype = ["number", "bool"]
map = { "value" → "bool" }  // 第二个覆盖了第一个
```

**后果**：两列都用最后一列的 datatype，第一列类型错误。

#### 6.6.4 异常3：表头重命名

**触发条件**：用户在前端编辑了表头名称（如果支持）

**行为**：重命名的列名不在 `config_headers` 映射中，回退到 `dt[i]`

**后果**：重命名列的 datatype 可能错误（如果该列之前被增删操作改变过位置）。

#### 6.6.5 异常4：新增列再删除中间列

**行为**：
- 新增列的类型都是 `str`
- 删除中间列后，剩余新增列的类型还是 `str`
- 但位置变化后 `dt[i]` 回退可能指向不同的原始类型

**后果**：新增列删除顺序不同，最终剩余列的类型可能不一致。

### 6.7 数据完整性约束

| 操作 | 约束条件 | 违反时行为 |
|------|----------|------------|
| 添加行 | `row_count[1] === "dynamic"` | 静默忽略 |
| 添加列 | `col_count[1] === "dynamic"` | 静默忽略 |
| 删除行 | `values.length > 1` | 静默忽略 |
| 删除列 | `col_count[1] === "dynamic"` 且列数 > 1 | 静默忽略 |
| 编辑静态列 | `!static_columns.includes(col)` | 阻止编辑，不渲染 textarea |

---

## 七、事件系统

### 7.1 事件类型与触发时机

| 事件 | 触发时机 | 载荷类型 |
|------|----------|----------|
| `change` | 数据变更完成后（编辑、增删行列、上传等） | `DataframeValue`（完整 data + headers + metadata） |
| `input` | 用户主动输入时（同 change 时机，但仅非输出模式） | 无载荷 |
| `edit` | 单元格值变更时（细粒度） | `EditData`（坐标、新值、旧值） |
| `select` | 单元格被选中时 | `SelectData`（坐标、值、行值、列值） |

### 7.2 value_is_output 标志

**作用**：区分值是否来自后端输出，控制 `input` 事件的触发。

```typescript
// push_change 中
if (!value_is_output) oninput?.();
value_is_output = false;
```

**状态流转**：
- 初始值：由父组件传入（通常初始为 false，后端更新后为 true）
- 每次 push_change 后：设为 `false`（标记为用户输入产生的值）

**设计意图**：
- `input` 事件仅在用户主动交互时触发
- 后端返回的新值（value_is_output=true）不触发 input 事件
- 避免"后端更新 → change → input → 又触发后端"的循环

### 7.3 change 事件去重

**Index.svelte 中的去重逻辑** ([Index.svelte#L41-L75](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L41-L75)):

```typescript
let old_value = $state(
    gradio.props.value ? JSON.stringify(gradio.props.value) : null
);

function handle_change(detail: any): void {
    gradio.props.value = detail;
    const serialized = JSON.stringify(detail);
    if (serialized !== old_value) {
        old_value = serialized;
        gradio.dispatch("change", detail);
    }
}

// 监听外部 value 变更（后端更新）
$effect(() => {
    const v = gradio.props.value;
    if (v) {
        const serialized = JSON.stringify(v);
        if (serialized !== old_value) {
            old_value = serialized;
            gradio.dispatch("change", v);
        }
    }
});
```

> **设计要点**：
> - 通过 JSON 序列化比较实现去重
> - 双向监听：内部编辑（handle_change）和外部更新（$effect）都走同一个去重逻辑
> - 外部更新通过 props.value 变化触发，确保内外状态一致

---

## 八、后端反序列化（preprocess）

**`preprocess()` 方法** ([dataframe.py#L272-L310](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L272-L310)):

```python
def preprocess(
    self, payload: DataframeData
) -> pd.DataFrame | np.ndarray | pl.DataFrame | list[list]:
    import pandas as pd

    if self.type == "pandas":
        if payload.headers is not None:
            return pd.DataFrame(
                [] if payload.data == [[]] else payload.data,
                columns=payload.headers,
            )
        else:
            return pd.DataFrame(payload.data)
            
    if self.type == "polars":
        polars = _import_polars()
        if payload.headers is not None:
            return polars.DataFrame(
                [] if payload.data == [[]] else payload.data,
                schema=payload.headers,
                orient="row",
            )
        else:
            return polars.DataFrame(payload.data)
            
    if self.type == "numpy":
        return np.array(payload.data)
    elif self.type == "array":
        return payload.data
```

> **空数据特殊处理**：当 `payload.data == [[]]` 时，pandas/polars 会创建空 DataFrame（0 行），而非包含一个空行的 DataFrame。

**类型推断注意**：
- 前端回传的 data 是混合类型的（bool 列是 boolean，其他是 string）
- pandas 会根据数据自动推断列类型
- bool 列传的是 boolean 值，pandas 识别为 bool 类型
- 数字列传的是 string，pandas 可能推断为 object 或自动转换

---

## 九、关键代码路径索引

### 9.1 序列化初始路径
1. 组件初始化入口: [base.py#L207-L219](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/base.py#L207-L219)
2. postprocess 主流程: [dataframe.py#L461-L517](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L461-L517)
3. metadata 统一入口 `get_metadata()`: [dataframe.py#L436-L459](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L436-L459)
4. metadata 分支 1：Styler 自动生成 `__extract_metadata()`: [dataframe.py#L609-L639](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L609-L639)
5. metadata 分支 2：dict 直接携带 `value.get("metadata", None)`: [dataframe.py#L457-L458](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L457-L458)

### 9.2 单元格编辑路径
1. 双击进入编辑: [Table.svelte#L433-L446](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L433-L446)
2. 失焦提交变更: [Table.svelte#L448-L472](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L448-L472)
3. 类型转换（展示层）: [utils.ts#L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)
4. 后端反序列化: [dataframe.py#L272-L310](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L272-L310)

### 9.3 行列变更路径
1. 添加行: [Table.svelte#L556-L570](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L556-L570)
2. 添加列: [Table.svelte#L572-L587](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L572-L587)
3. 删除行: [Table.svelte#L589-L595](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L589-L595)
4. 删除列: [Table.svelte#L597-L614](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L597-L614)

### 9.4 类型推断与对齐路径
1. 前端类型转换: [utils.ts#L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)
2. 后端自动推断: [dataframe.py#L519-L606](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L519-L606)
3. datatype 对齐: [Index.svelte#L24-L39](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L24-L39)

### 9.5 metadata 来源与 display_value 路径

**后端 metadata 三条来源路径**：
1. 统一入口 `get_metadata()`: [dataframe.py#L436-L459](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L436-L459)
2. Styler 自动生成分支 `__extract_metadata()`: [dataframe.py#L609-L639](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L609-L639)
3. dict 直接携带分支 `value.get("metadata", None)`: [dataframe.py#L457-L458](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L457-L458)

**前端 display_value 消费路径**：
4. 前端接收并拆分 metadata → display_value/styling: [standalone/Index.svelte#L123-L124](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/standalone/Index.svelte#L123-L124)
5. 显示优先级逻辑（EditableCell）: [EditableCell.svelte#L90-L92](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/EditableCell.svelte#L90-L92)
6. get_display_value 函数（Table）: [Table.svelte#L353-L357](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L353-L357)

**编辑回传 metadata 丢弃路径**：
7. push_change 中 metadata 置空: [Table.svelte#L363-L374](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L363-L374)

### 9.6 事件系统路径
1. push_change 触发点: [Table.svelte#L363-L374](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L363-L374)
2. change 去重逻辑: [Index.svelte#L41-L75](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L41-L75)
3. value_is_output 标记使用: [Table.svelte#L63](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L63)
