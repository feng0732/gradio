# DataFrame 组件编辑后的数据回传机制

## 一、整体数据流向

### 1.1 数据流转概览

```
前端交互（单元格编辑/行列增删）
        ↓
Table.svelte 内部状态更新（values, headers）
        ↓
push_change() 触发 onchange 事件
        ↓
Index.svelte handle_change() 分发 change 事件
        ↓
后端 preprocess() 接收 DataframeData 载荷
        ↓
转换为 pandas/numpy/polars/array 格式
```

### 1.2 核心文件与角色

| 文件 | 角色 | 关键功能 |
|------|------|----------|
| [dataframe.py](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py) | 后端组件 | `preprocess()` 反序列化、`postprocess()` 序列化、`set_auto_datatype()` 类型推断 |
| [Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte) | 前端核心表 | 编辑处理、行列增删、`push_change()` 触发回传 |
| [EditableCell.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/EditableCell.svelte) | 单元格编辑 | 文本编辑、blur 事件提交变更 |
| [utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts) | 类型工具 | `cast_value_to_type()` 前端类型转换 |
| [Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte) | 前端包装 | 事件分发、datatype 对齐 |

---

## 二、表格序列化流程

### 2.1 数据结构定义

**前端数据格式** ([utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L19-L23)):
```typescript
export type DataframeValue = {
    data: CellValue[][];      // 二维数组：string | number | boolean
    headers: string[];        // 列名数组
    metadata?: Metadata;      // 可选元数据（样式、显示值等）
};
```

**后端数据模型** ([dataframe.py](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L45-L49)):
```python
class DataframeData(GradioModel):
    headers: list[Any]
    data: Union[list[list[Any]], list[tuple[Any, ...]]]
    metadata: dict[str, list[Any] | None] | None = None
```

### 2.2 序列化触发点

**`push_change()` 函数** ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L363-L374)):

```typescript
function push_change(
    new_values?: CellValue[][],
    new_headers?: (string | null)[]
): void {
    onchange?.({
        data: new_values ?? values,
        headers: (new_headers ?? resolved_headers) as string[],
        metadata: null
    });
    if (!value_is_output) oninput?.();
    value_is_output = false;
}
```

**触发场景**：
1. **单元格编辑完成** - `handle_blur()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L470))
2. **添加行** - `add_row()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L567))
3. **添加列** - `add_col()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L585))
4. **删除行** - `delete_row_at()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L592))
5. **删除列** - `delete_col_at()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L608))
6. **全选布尔列** - `handle_select_all()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L633))
7. **删除单元格** - 键盘 Delete/Backspace ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L794))
8. **应用筛选** - `commit_filter()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L659))
9. **文件上传** - `on_file_upload()` 中调用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L853))

### 2.3 单元格编辑提交流程

**编辑触发路径**：
1. 双击单元格 → `handle_cell_dblclick()` → 设置 `editing = [row, col]` ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L433-L446))
2. EditableCell 渲染 textarea 进入编辑模式
3. 失焦或按 Enter → `handle_blur()` 提交 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L448-L472))

**`handle_blur()` 核心逻辑**：
```typescript
function handle_blur(detail: {
    blur_event: FocusEvent;
    coords: [number, number];
}): void {
    const { coords } = detail;
    const input_el = detail.blur_event.target as HTMLTextAreaElement;
    
    const [row, col] = coords;
    const old_value = values?.[row]?.[col];
    const new_value = input_el.value;

    if (String(old_value) !== String(new_value)) {
        // 1. 创建新数组（不可变更新）
        const new_values = values.map((r) => [...r]);
        new_values[row][col] = new_value;
        values = new_values;

        // 2. 触发 edit 事件（携带新旧值）
        onedit?.({
            index: [row, col],
            value: new_value,
            previous_value: String(old_value ?? "")
        });
        
        // 3. 触发 change 事件（完整数据回传）
        push_change(new_values);
    }
}
```

**EditData 事件结构** ([utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L54-L58)):
```typescript
export interface EditData {
    index: number | [number, number];  // 单元格坐标 [row, col]
    value: string;                      // 新值
    previous_value: string;             // 旧值
}
```

### 2.4 后端反序列化（preprocess）

**`preprocess()` 方法** ([dataframe.py](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L272-L310)):

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

> **注意**：空数据特殊处理 - 当 `payload.data == [[]]` 时，pandas/polars 会创建空 DataFrame 而非包含一个空行的 DataFrame。

---

## 三、类型推断机制

### 3.1 前端类型转换（cast_value_to_type）

**`cast_value_to_type()` 函数** ([utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)):

```typescript
export function cast_value_to_type(v: any, t: Datatype): CellValue {
    if (v === null || v === undefined) {
        return v;
    }
    if (t === "number") {
        const n = Number(v);
        return isNaN(n) ? v : n;  // 转换失败返回原值
    }
    if (t === "bool") {
        if (typeof v === "boolean") return v;
        if (typeof v === "number") return v !== 0;
        const s = String(v).toLowerCase();
        if (s === "true" || s === "1") return true;
        if (s === "false" || s === "0") return false;
        return v;  // 转换失败返回原值
    }
    if (t === "date") {
        const d = new Date(v);
        return isNaN(d.getTime()) ? v : d.toISOString();  // 转换失败返回原值
    }
    return v;  // str/markdown/html/image 直接返回
}
```

**类型转换时机** - 在 `row_data` 派生值中应用 ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L117-L126)):

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

> **设计要点**：类型转换是"尽力而为"的，转换失败时保持原值不变，而非抛出错误。

### 3.2 后端自动类型推断（set_auto_datatype）

**`set_auto_datatype()` 方法** ([dataframe.py](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L519-L606)):

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

1. **pandas DataFrame** - 使用 `df.dtypes`
   ```python
   self.datatype = [
       dtype_mapping.get(
           numbers_re.sub("", brackets_re.sub("", str(dtype))).lower(), "str"
       )
       for dtype in value.dtypes
   ]
   ```

2. **numpy ndarray** - 使用 `type(value[0, i]).__name__`
   ```python
   self.datatype = [
       dtype_mapping.get(
           numbers_re.sub("", brackets_re.sub("", str(type(value[0, i]).__name__))).lower(),
           "str",
       )
       for i in range(value.shape[1])
   ]
   ```

3. **Python list** - 使用 `type(val).__name__`（检查第一行）
   ```python
   self.datatype = [
       dtype_mapping.get(
           numbers_re.sub("", brackets_re.sub("", str(type(val).__name__)).lower(),
           "str",
       )
       for val in value[0]
   ]
   ```

4. **polars DataFrame** - 使用 `value.dtypes`
   ```python
   self.datatype = [
       dtype_mapping.get(
           numbers_re.sub("", brackets_re.sub("", str(dtype))).lower(),
           "str",
       )
       for dtype in value.dtypes
   ]
   ```

**类型名称清洗**：
- 使用正则 `\[.*?\]|\(.*?\)` 移除括号及内容（如 `int64` → `int`）
- 使用正则 `\s*\d+\s*$` 移除尾部数字（如 `datetime64` → `datetime`）
- 转为小写后查表，未匹配则默认 `"str"`

### 3.3 前端 datatype 对齐

**`aligned_datatype` 派生值** ([Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L24-L39)):

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

> **设计目的**：当列被隐藏、重排或增删时，通过表头名称映射保持 datatype 与列的正确对应关系，而非依赖位置索引。

---

## 四、行列变更处理逻辑

### 4.1 配置模式：fixed vs dynamic

**行列配置格式**：
```typescript
type CountConfig = [number, "fixed" | "dynamic"];
```

- **`[n, "fixed"]`**：固定 n 行/列，不允许增删
- **`[n, "dynamic"]`**：初始 n 行/列，允许增删

**处理逻辑**：所有增删操作前都会检查模式，例如 `add_row()` ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L556)):
```typescript
function add_row(index?: number): void {
    if (row_count[1] !== "dynamic") return;  // fixed 模式直接返回
    // ... 执行添加
}
```

**static_columns 特殊处理** ([standalone/Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/standalone/Index.svelte#L92-L104)):
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

### 4.2 添加行（add_row）

**`add_row()` 函数** ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L556-L570)):

```typescript
function add_row(index?: number): void {
    if (row_count[1] !== "dynamic") return;
    
    // 新行长度 = 第一行长度 或 表头长度 或 1
    const col_len = values[0]?.length || resolved_headers.length || 1;
    const new_row: CellValue[] = Array(col_len).fill("");
    
    const new_values = [...values];
    if (index !== undefined) {
        new_values.splice(index, 0, new_row);  // 插入指定位置
    } else {
        new_values.push(new_row);  // 追加到末尾
    }
    
    values = new_values;
    push_change(new_values);
    
    // 选中新行第一列
    selected = [index ?? new_values.length - 1, 0];
    parent?.focus();
}
```

**调用场景**：
- 空表格时点击 `EmptyRowButton`
- 右键菜单 "Add row above" → `add_row_at(index, "above")`
- 右键菜单 "Add row below" → `add_row_at(index, "below")`

### 4.3 添加列（add_col）

**`add_col()` 函数** ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L572-L587)):

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
        // 将最后一个（刚添加的）插入到指定位置
        new_headers.splice(index, 0, new_headers.pop()!);
        new_values.forEach((row) => row.splice(index, 0, row.pop()!));
    }
    
    values = new_values;
    headers = new_headers;
    push_change(new_values, new_headers);
    parent?.focus();
}
```

**调用场景**：
- 表头右键菜单 "Add column to the left" → `add_col_at(index, "left")`
- 表头右键菜单 "Add column to the right" → `add_col_at(index, "right")`

### 4.4 删除行（delete_row_at）

**`delete_row_at()` 函数** ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L589-L595)):

```typescript
function delete_row_at(index: number): void {
    if (values.length <= 1) return;  // 至少保留一行
    
    values = [...values.slice(0, index), ...values.slice(index + 1)];
    push_change(values);
    
    active_cell_menu = null;
    active_header_menu = null;
}
```

### 4.5 删除列（delete_col_at）

**`delete_col_at()` 函数** ([Table.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L597-L614)):

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

### 4.6 数据完整性约束

| 操作 | 约束条件 | 违反时行为 |
|------|----------|------------|
| 添加行 | `row_count[1] === "dynamic"` | 静默忽略 |
| 添加列 | `col_count[1] === "dynamic"` | 静默忽略 |
| 删除行 | `values.length > 1` | 静默忽略 |
| 删除列 | `col_count[1] === "dynamic"` 且列数 > 1 | 静默忽略 |
| 编辑静态列 | `!static_columns.includes(col)` | 阻止编辑，无 textarea |

---

## 五、事件系统

### 5.1 事件类型与触发时机

| 事件 | 触发时机 | 载荷类型 |
|------|----------|----------|
| `change` | 数据变更完成后（编辑、增删行列等） | `DataframeValue`（完整数据） |
| `input` | 用户输入时（同 change，但仅非输出模式） | `never`（无载荷） |
| `edit` | 单元格值变更时 | `EditData`（坐标、新旧值） |
| `select` | 单元格被选中时 | `SelectData`（坐标、值、行值、列值） |

### 5.2 change 事件去重

**Index.svelte 中的去重逻辑** ([Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L41-L75)):

```typescript
let old_value = $state(
    gradio.props.value ? JSON.stringify(gradio.props.value) : null
);

function handle_change(detail: any): void {
    gradio.props.value = detail;
    const serialized = JSON.stringify(detail);
    if (serialized !== old_value) {  // 仅在实际变更时分发
        old_value = serialized;
        gradio.dispatch("change", detail);
    }
}

// 监听外部 value 变更
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

> **设计要点**：通过 JSON 序列化比较实现去重，避免重复触发 change 事件。

---

## 六、关键代码路径索引

### 6.1 单元格编辑路径
1. 双击: [Table.svelte#L433-L446](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L433-L446)
2. 失焦提交: [Table.svelte#L448-L472](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L448-L472)
3. 类型转换: [utils.ts#L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)
4. 后端反序列化: [dataframe.py#L272-L310](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L272-L310)

### 6.2 行列变更路径
1. 添加行: [Table.svelte#L556-L570](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L556-L570)
2. 添加列: [Table.svelte#L572-L587](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L572-L587)
3. 删除行: [Table.svelte#L589-L595](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L589-L595)
4. 删除列: [Table.svelte#L597-L614](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/Table.svelte#L597-L614)

### 6.3 类型推断路径
1. 前端类型转换: [utils.ts#L31-L52](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/shared/utils/utils.ts#L31-L52)
2. 后端自动推断: [dataframe.py#L519-L606](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/gradio/components/dataframe.py#L519-L606)
3. datatype 对齐: [Index.svelte#L24-L39](file:///d:/fz/0601/solo-dogfeeding/code/258-gradio/js/dataframe/Index.svelte#L24-L39)
