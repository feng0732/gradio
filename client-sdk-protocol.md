# 客户端 SDK 与服务端预测协议配合说明

本文档梳理 Gradio 客户端 SDK 与服务端之间的预测协议配合机制，包括调用参数格式、任务提交流程、结果接收方式，以及各版本 SSE 协议的差异。

---

## 1. 核心架构概览

### 1.1 协议版本：代码事实

#### 1.1.1 类型定义 vs 运行时

协议字段的**类型定义**在 [types.ts:202](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/types.ts#L202) 和 [data_classes.py:403](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L403) 中声明了 6 种候选值：

```typescript
// 仅为 TypeScript/Pydantic 类型声明，不代表运行时全部出现
protocol: "ws" | "sse" | "sse_v1" | "sse_v2" | "sse_v2.1" | "sse_v3"
```

但在**运行时**，服务端只在一个地方给 `config.protocol` 赋值，且硬编码为 `sse_v3`：

[blocks.py:2404](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/blocks.py#L2404)：
```python
"protocol": "sse_v3",   # 唯一赋值点，没有条件分支
```

全局搜索 `gradio/` 目录下所有 Python 文件，`protocol.*=.*"sse_` 仅此一处出现。因此**当前版本运行时只使用 `sse_v3`**，无论什么启动参数、环境变量都无法改变。

#### 1.1.2 各版本在代码中的真实地位

根据代码可达性分析，各版本的状态如下：

| 版本 | 运行时可达？ | 在代码中的角色 |
|------|------------|--------------|
| **ws** | ❌ 不可达 | 死分支，客户端一遇到就抛异常 [submit.ts:78-79](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L78-L79) |
| **sse** | ❌ 不可达 | **历史兼容分支**，服务端没有任何代码能返回此值，详见 [3.3 节](#33-旧版-sse-纯历史兼容分支当前不可达) |
| **sse_v1** | ❌ 不可达 | **历史兼容分支**，与 sse_v2/sse_v3 共用同一提交入口，但运行时服务端不返回此值 |
| **sse_v2/v2.1** | ❌ 不可达 | **历史兼容分支**，同上。与 sse_v1 的差异仅在 3 处条件判断（见下表） |
| **sse_v3** | ✅ 唯一活跃 | 当前实际使用的协议 |

> **为什么客户端保留了 sse_v1/sse_v2 的条件判断？**
>
> 这是**向后兼容旧服务端**的设计。如果客户端连接到较老版本的 Gradio Server（其 config.protocol 返回 sse_v1 或 sse_v2），客户端仍能正常工作。但**在本代码库内**，服务端只发 sse_v3。

#### 1.1.3 sse_v1 vs sse_v2 vs sse_v3 在客户端的实际差异

四者共用同一大分支（[submit.ts:395-631](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L395-L631)），区别仅在 3 处条件判断：

| 差异点 | 代码位置 | sse_v1 | sse_v2/v2.1 | sse_v3 |
|-------|---------|--------|------------|--------|
| **生成器增量 diff** | [submit.ts:551-557](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L551-L557) | ❌ 不应用 | ✅ 应用 `apply_diff_stream` | ✅ 应用 |
| **回调异常时是否关 SSE** | [submit.ts:611-615](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L611-L615) | ❌ 只关迭代器 | ✅ 关 SSE 连接 | ✅ 关 SSE 连接 |
| **process_completed 删 unclosed_events** | [stream.ts:55-62](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L55-L62) | ✅ 删除 | ✅ 删除 | ✅ 删除 |

> 注意注释（[submit.ts:402](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L402)）声称 "v3 only closes the stream when the backend sends the close stream message"，但**代码中 sse_v2 和 sse_v3 在上述 3 处判断中行为完全相同**，没有单独针对 v3 的分支。该注释与代码事实不一致。实际行为：**sse_v1 异常时不关 SSE；sse_v2+（含 v2, v2.1, v3）异常时都关 SSE**。正常完成时所有版本都由服务端发 `close_stream` 关闭。

### 1.2 核心模块

| 模块 | 位置 | 职责 |
|------|------|------|
| Client 类 | [client.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/client.ts) | SDK 入口，管理连接、会话、流状态 |
| submit 函数 | [submit.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts) | 任务提交核心，区分协议版本处理 |
| predict 函数 | [predict.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/predict.ts) | 基于 submit 的 Promise 封装 |
| open_stream | [stream.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts) | 建立会话级 SSE 长连接，消息分发 |
| handle_message | [api_info.ts:234-404](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/helpers/api_info.ts#L234-L404) | 服务端消息到客户端事件的转换 |
| Queue 类 | [queueing.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py) | 服务端队列管理，消息推送 |
| API 路由 | [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py) | HTTP 接口定义，SSE 响应 |

### 1.3 常量映射

[constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/constants.ts) 中定义的端点常量，容易混淆：

| 常量名 | 值 | 用途 | 使用者 |
|--------|---|------|--------|
| `SSE_DATA_URL` | `"queue/join"` | **提交**任务的 POST 端点 | sse_v1+ 分支 |
| `SSE_URL` | `"queue/data"` | **接收**结果的 SSE GET 端点 | sse / sse_v1+ 分支 |
| `SSE_URL_V0` | `"queue/join"` | 旧版提交端点 | **未被引用，死常量** |
| `SSE_DATA_URL_V0` | `"queue/data"` | 旧版数据端点 | **未被引用，死常量** |

### 1.4 核心数据结构

**客户端会话状态**（Client 类成员）：

```typescript
session_hash: string                        // 客户端会话标识，随机生成
stream_status: { open: boolean }            // SSE 流是否打开
event_callbacks: Record<event_id, callback> // 事件ID -> 回调函数
pending_stream_messages: Record<event_id, msg[]> // 早到消息缓存
unclosed_events: Set<event_id>              // 未关闭的事件集合
pending_diff_streams: Record<event_id, data[]>   // diff 流的累积状态
abort_controller: AbortController | null    // 用于中止 fetch 请求
```

**服务端会话状态**（Queue 类成员）：

```python
pending_messages_per_session: LRUCache[session_hash, AsyncQueue]  # 每个会话的消息队列
pending_event_ids_session: dict[session_hash, set[event_id]]     # 会话内未完成事件
event_ids_to_events: dict[event_id, Event]                        # event_id -> Event 对象
```

---

## 2. 调用参数详解

### 2.1 客户端入口

**Client.connect()** - 建立连接：

```typescript
static async connect(
    app_reference: string,           // URL 或 HF Space 名称
    options: ClientOptions = {
        events: ["data"],             // 订阅的事件类型
        token?: `hf_${string}`,       // HF token
        auth?: [string, string],      // 用户名密码
        session_hash?: string         // 自定义会话哈希
    }
): Promise<Client>
```

初始化流程 [client.ts:223-236](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/client.ts#L223-L236)：
1. 解析 endpoint，获取 host 和 protocol
2. 获取 config（包含组件、依赖、协议版本信息）
3. 连接心跳（如需 state 或 unload 事件）
4. 获取 API info（端点参数定义）
5. 建立 api_name -> fn_index 映射

### 2.2 预测调用参数

**predict() 方法** [predict.ts:4-51](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/predict.ts#L4-L51)：

```typescript
predict<T = unknown>(
    endpoint: string | number,    // API 名称或 fn_index
    data: unknown[] | Record<string, unknown> = {}  // 参数
): Promise<PredictReturn<T>>
```

**submit() 方法** [submit.ts:32-716](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L32-L716)：

```typescript
submit(
    endpoint: string | number,
    data: unknown[] | Record<string, unknown> = {},
    event_data?: unknown,
    trigger_id?: number | null,
    all_events?: boolean
): SubmitIterable<GradioEvent>
```

### 2.3 参数映射机制

**参数解析** [api_info.ts:429-480](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/helpers/api_info.ts#L429-L480)：

- **位置参数**：`data: [value1, value2]` 按顺序映射
- **命名参数**：`data: { param1: value1, param2: value2 }` 按键名映射
- 默认值填充：未提供的参数使用 `parameter_default`
- 验证：必填参数缺失时抛出错误

### 2.4 请求数据结构

**PredictBody** [data_classes.py:90-119](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L90-L119)：

```python
class PredictBody(BaseModel):
    session_hash: str | None = None      # 会话标识
    event_id: str | None = None          # 事件ID（由服务端生成）
    data: list[Any]                      # 输入数据数组
    event_data: Any | None = None        # 事件特定数据
    fn_index: int | None = None          # 函数索引
    trigger_id: int | None = None        # 触发器ID
    simple_format: bool = False
    batched: bool | None = False         # 是否批量请求
```

### 2.5 文件处理机制

**文件上传流程** [handle_blob.ts:16-62](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/handle_blob.ts#L16-L62)：

1. `walk_and_store_blobs()` - 递归遍历数据，收集 Blob/Buffer/File 对象
2. `upload_files()` - 上传文件到 `/upload` 端点，获取服务器文件路径
3. 替换原始 Blob 为 FileData 对象，包含 `path` 和 `url`

**FileData 结构** [data_classes.py:230-251](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/data_classes.py#L230-L251)：

```python
class FileData(GradioModel):
    path: str                    # 服务器文件路径
    url: str | None = None       # 访问URL
    size: int | None = None
    orig_name: str | None = None # 原始文件名
    mime_type: str | None = None
    is_stream: bool = False
    meta: FileDataMeta = {"_type": "gradio.FileData"}
```

---

## 3. 任务提交：各协议版本对比

### 3.1 总览：提交入口、结果接收、收尾时机

**按代码事实列出所有分支（含不可达的历史兼容分支）：**

| 分支条件 | 提交入口 | 结果接收 | 单请求迭代器 `close()` 时机 | SSE 连接关闭时机 |
|---------|---------|---------|-------------------------|----------------|
| **非队列** <br/>`skip_queue() = true` | `POST /run/{api}` | 同步 HTTP 响应 | 响应返回即 `close()` | 无 SSE 连接 |
| **sse（旧版）** <br/>`protocol === "sse"` | `GET /queue/data?fn_index=X&session_hash=Y` 建 SSE 即提交 | 同一 SSE 连接上收消息 | 收到 `process_completed` 且 data 到达后 `close()` | 回调中主动调用 `stream.close()` |
| **sse_v1** <br/>`protocol === "sse_v1"` | `POST /queue/join` → event_id | 共享 `GET /queue/data?session_hash=Y` | callback 检测 complete/error → `close()` | **正常**：服务端发 `close_stream` <br/>**异常**：只 `close()` 迭代器，SSE 不关 |
| **sse_v2/v2.1** <br/>`protocol === "sse_v2\|sse_v2.1"` | 同 sse_v1 | 同 sse_v1 | 同 sse_v1 + 生成器中间结果应用 `apply_diff_stream` | **正常**：服务端发 `close_stream` <br/>**异常**：立即调 `close_stream()` 关 SSE |
| **sse_v3** <br/>`protocol === "sse_v3"`（**当前运行时唯一可达**） | 同 sse_v1 | 同 sse_v1 | 同 sse_v2 | 与 sse_v2 **代码完全相同**，无独立分支 |

> **关键区分：`close()` vs 关闭 SSE 连接**
>
> | 操作 | 代码位置 | 作用 | 是否影响 SSE |
> |------|---------|------|------------|
> | `close()` | [submit.ts:640-647](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L640-L647) | 设 `done=true`，resolve 迭代器 Promise | ❌ 不影响 |
> | `close_stream()` | [stream.ts:91-99](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L91-L99) | 设 `open=false` + `abort_controller.abort()` | ✅ 关闭 SSE 连接 |
> | 收到 `close_stream` 消息 | [stream.ts:43-45](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L43-L45) | 调用 `close_stream()` | ✅ 关闭 SSE 连接 |

### 3.2 非队列模式（直接调用）

**适用条件**：`skip_queue(fn_index, config)` 返回 true

**提交** [submit.ts:188-265](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L188-L265)：

```
客户端 POST /run/{endpoint}
   body: { data, session_hash, fn_index, event_data, trigger_id }
   └─> 服务端直接执行函数 call_process_api()
   └─> 同步返回 { data: [...], average_duration, ... }
```

**服务端路由** [routes.py:1269-1316](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1269-L1316)：

```python
@router.post("/run/{api_name}")
async def predict(api_name, body, request, username):
    fn = route_utils.get_fn(...)
    output = await route_utils.call_process_api(...)
    return ORJSONResponse(output)
```

**收尾**：
- 一次性请求，响应返回即结束
- 无状态维护，无 SSE 连接
- 不支持生成器函数的中间输出

### 3.3 旧版 SSE：纯历史兼容分支，当前不可达

**代码位置** [submit.ts:266-394](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L266-L394)

**当前代码库内，服务端没有任何代码路径能让此分支被执行。** 它是为兼容更早版本 Gradio 服务端保留的历史遗留。以下是代码证据：

| 证据 | 文件 | 事实 |
|------|------|------|
| config.protocol 唯一赋值点 | [blocks.py:2404](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/blocks.py#L2404) | 硬编码 `"sse_v3"`，永远不会返回 `"sse"` |
| 服务端消息类型 | [server_messages.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/server_messages.py) | 只有 9 种消息，**无** `send_hash` / `send_data`（客户端此分支依赖它们） |
| `GET /queue/data` 参数 | [routes.py:1463-1467](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1463-L1467) | 只接收 `session_hash`，**不接收** `fn_index`。此分支的 URL 带 `fn_index` 参数，但服务端会忽略，导致无法入队 |
| `SSE_URL_V0` / `SSE_DATA_URL_V0` | [constants.ts:4-5](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/constants.ts#L4-L5) | 定义了但无任何代码引用，是死常量 |

**此分支的设计意图**（基于代码推测，仅对更老服务端有效）：

1. 客户端直接 `GET /queue/data?fn_index=X&session_hash=Y` 建立 SSE 连接，相当于把 "提交" 和 "接收" 合并成一步
2. 服务端推送 `send_hash` / `send_data` 消息，客户端响应后真正入队
3. 每个请求独立一条 SSE 连接，无共享
4. 收到 `process_completed` 后客户端主动调 `stream.close()` 关闭本连接
5. `handle_message` 中 `send_hash` / `send_data` 的 case [api_info.ts:256-259](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/helpers/api_info.ts#L256-L259) 就是为此分支预留的

**结论**：此分支是为兼容早期 Gradio 版本（可能是 v0.x 的 v0 协议格式）保留的前置兼容代码。在本代码库部署的服务端上，`protocol` 永远是 `sse_v3`，所以此分支在**当前版本运行时不可达**。

### 3.4 SSE v1+：当前唯一有效的队列协议

**代码位置** [submit.ts:395-631](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L395-L631)

sse_v1、sse_v2、sse_v2.1、sse_v3 共用同一个分支，核心流程一致：

1. **提交和接收分离**：先 POST 提交拿 event_id，再通过共享 SSE 流接收
2. **会话级连接复用**：同一个 `session_hash` 共用一条 SSE 连接
3. **消息带 event_id**：客户端按 event_id 分发到对应回调

**提交流程**：

```
客户端                                服务端
   |                                     |
   |--- POST /queue/join ---------------->|
   |    { data, session_hash, fn_index }  |
   |                                     |
   |<--- { event_id: "abc123" } ---------|   返回事件ID
   |                                     |
   |  [如果 SSE 流未打开]                 |
   |--- GET /queue/data?session_hash=Y ->|   建立会话级长连接
   |                                     |
   |<--- msg: "estimation"  (event_id: "abc123")
   |<--- msg: "process_starts" (event_id: "abc123")
   |<--- msg: "process_completed" (event_id: "abc123")
```

**提交代码** [submit.ts:429-436](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L429-L436)：

```typescript
post_data(`${config.root}${api_prefix}/${SSE_DATA_URL}?${url_params}`, {
    ...payload,       // { data, fn_index, event_data, ... }
    session_hash
})
// SSE_DATA_URL = "queue/join"
```

**服务端提交路由** [routes.py:1357-1399](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1357-L1399)：

```python
@router.post("/queue/join")
async def queue_join(body, request, username):
    body = PredictBodyInternal(**body.model_dump(), request=request)
    success, event_id, state = await blocks._queue.push(body, request, username)
    return {"event_id": event_id}
```

**入队处理** [queueing.py:279-468](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L279-L468)：

1. 验证 fn_index
2. 检查队列是否已满
3. 执行验证器（如有）
4. 检查缓存命中（如有 cache 装饰器）
5. 创建 Event 对象，生成 event_id（uuid4）
6. 初始化会话消息队列 `pending_messages_per_session[session_hash]`
7. 事件加入 `event_queue_per_concurrency_id` 等待调度
8. 广播队列位置估计（`EstimationMessage`）

### 3.5 sse_v1 与 sse_v2/v2.1/v3 的差异：增量 Diff 输出

sse_v2+ 在 v1 基础上增加了一个优化：生成器函数的中间结果只发送增量 diff，减少数据传输量。

**Diff 格式** [stream.ts:121-178](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L121-L178)：

```typescript
// diff 是一组编辑操作
[action, path, value][]

// action: "replace" | "append" | "add" | "delete"
// path: [key1, index1, ...] 嵌套定位路径
// value: 新值

// 示例：给 output[0].text 追加内容
[["append", [0, "text"], " more content"]]

// 示例：替换数组第3项
[["replace", [2], {"name": "new", "value": 42}]]
```

**客户端累积** [stream.ts:101-119](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L101-L119)：

```typescript
function apply_diff_stream(pending_diff_streams, event_id, data) {
    if (!pending_diff_streams[event_id]) {
        // 首次：保存完整数据
        pending_diff_streams[event_id] = data.data;
    } else {
        // 后续：应用 diff 到已有数据
        data.data.forEach((value, i) => {
            let new_data = apply_diff(pending_diff_streams[event_id][i], value);
            pending_diff_streams[event_id][i] = new_data;
            data.data[i] = new_data;  // 替换为完整数据供上层使用
        });
    }
}
```

**适用条件** [submit.ts:551-557](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L551-L557)：

```typescript
if (
    data &&
    dependency.connection !== "stream" &&  // 非 stream 连接类型
    ["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)
) {
    apply_diff_stream(pending_diff_streams, event_id!, data);
}
```

### 3.6 sse_v1 vs sse_v2 vs sse_v3：SSE 关闭策略与 Diff 差异

三个版本共用同一大分支（[submit.ts:395-631](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L395-L631)），差异仅在 3 处条件判断，全部集中在 `sse_v2+`（即 sse_v2, sse_v2.1, sse_v3）与 sse_v1 之间。**sse_v2、sse_v2.1、sse_v3 三者在代码中行为完全一致，无独立分支。**

#### 差异 1：生成器增量 Diff（sse_v1 vs sse_v2+）

[submit.ts:551-557](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L551-L557)：
```typescript
if (
    data &&
    dependency.connection !== "stream" &&
    ["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)  // ← sse_v1 不在此列表
) {
    apply_diff_stream(pending_diff_streams, event_id!, data);
}
```

- **sse_v1**：生成器每次 `yield` 时服务端都发送完整的当前输出
- **sse_v2+**：首次发送完整数据，后续只发送 diff 增量，客户端用 `apply_diff_stream()` 累积为完整数据

#### 差异 2：回调异常时是否关 SSE（sse_v1 vs sse_v2+）

[submit.ts:611-615](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L611-L615)（位于 callback 的 `catch` 块内）：
```typescript
if (["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)) {  // ← sse_v1 不在此列表
    close_stream(stream_status, that.abort_controller);  // 关 SSE
    stream_status.open = false;
    close();
}
// sse_v1 异常时不会执行上述代码，只在 catch 外 fire error 事件后自然 close()
```

- **sse_v1**：回调内抛异常 → 只 `close()` 本请求迭代器，**SSE 连接保持**（其他并发请求可能继续收消息）
- **sse_v2+**：回调内抛异常 → 立即调用 `close_stream()` 关闭整条 SSE 连接 + `close()` 迭代器

#### 差异 3：process_completed 时清理 unclosed_events（所有版本相同）

[stream.ts:55-62](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L55-L62)：
```typescript
if (
    _data.msg === "process_completed" &&
    ["sse", "sse_v1", "sse_v2", "sse_v2.1", "sse_v3"].includes(config.protocol)  // ← 所有版本
) {
    unclosed_events.delete(event_id);
}
```

所有 sse_* 版本（含旧版 sse）在收到 `process_completed` 时都会从 `unclosed_events` Set 中删除该 event_id。

#### 注释与代码不一致之处

[submit.ts:402](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L402) 的注释写道：
```
// v3 only closes the stream when the backend sends the close stream message.
```

但**代码中不存在任何只针对 v3 的单独判断**。上述所有条件判断要么包含 `sse_v2 + sse_v2.1 + sse_v3` 三者，要么包含全部版本。此注释与代码事实不符，应理解为：v3 设计上意图让服务端完全主导流关闭，但代码实现中（为简化？）sse_v2 和 v3 的行为相同。

#### 正常完成时：所有 sse_v1+ 版本 SSE 关闭策略一致

服务端每次 `process_completed` 后检查会话内是否还有未完成事件 [routes.py:1525-1559](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1525-L1559)。若全部完成则发送 `CloseStreamMessage`，客户端在 [stream.ts:43-45](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L43-L45) 无条件关闭 SSE。**此逻辑不区分 sse_v1/v2/v3，所有版本行为一致。**

---

## 4. 结果接收：会话级长连接与消息分发

### 4.1 会话级 SSE 连接的建立

**open_stream()** [stream.ts:5-89](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L5-L89) 是 sse_v1+ 共用的会话级长连接入口：

```typescript
export async function open_stream(this: Client): Promise<void> {
    stream_status.open = true;
    // 只带 session_hash，不带 fn_index
    let url = new URL(`${config.root}/${SSE_URL}?session_hash=${this.session_hash}`);
    stream = this.stream(url);
    // 注册统一的 onmessage 处理器
    stream.onmessage = function (event) { /* 分发逻辑 */ };
}
```

**连接时机** [submit.ts:626-628](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L626-L628)：

```typescript
// 注册回调后检查，如果流未打开则建立
if (!stream_status.open) {
    await this.open_stream();
}
```

**连接复用的关键数据**（都挂载在 Client 实例上）：

| 成员变量 | 类型 | 作用 |
|---------|------|------|
| `stream_status` | `{ open: boolean }` | 标记 SSE 流是否已打开 |
| `event_callbacks` | `Record<event_id, callback>` | 事件ID 到回调函数的映射 |
| `pending_stream_messages` | `Record<event_id, msg[]>` | 早到消息的缓存 |
| `unclosed_events` | `Set<event_id>` | 未完成的事件集合 |
| `abort_controller` | `AbortController` | 用于中止 fetch 请求 |

### 4.2 消息分发机制

**核心问题**：同一条 SSE 流上会传来多个事件的消息，怎么知道哪条消息属于哪个请求？

**答案**：每条消息都带 `event_id` 字段，客户端用 `event_callbacks` 表分发。

**分发逻辑** [stream.ts:41-77](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L41-L77)：

```typescript
stream.onmessage = async function (event: MessageEvent) {
    let _data = JSON.parse(event.data);
    
    // 1. close_stream：直接关闭连接，不交给回调
    if (_data.msg === "close_stream") {
        close_stream(stream_status, that.abort_controller);
        return;
    }
    
    const event_id = _data.event_id;
    
    if (!event_id) {
        // 2. 无 event_id 的消息：广播给所有回调（如 heartbeat）
        await Promise.all(
            Object.keys(event_callbacks).map(eid => event_callbacks[eid](_data))
        );
    } else if (event_callbacks[event_id]) {
        // 3. 有回调：分发到对应 callback
        let fn = event_callbacks[event_id];
        setTimeout(fn, 0, _data);  // 浏览器环境避免阻塞 UI
    } else {
        // 4. 无回调：缓存（回调还没注册，消息先到了）
        if (!pending_stream_messages[event_id]) {
            pending_stream_messages[event_id] = [];
        }
        pending_stream_messages[event_id].push(_data);
    }
};
```

### 4.3 竞态处理：消息早于回调

**为什么会有竞态？**

提交任务是 POST 请求，建立 SSE 连接是 GET 请求。如果 POST 很快返回 event_id，但 SSE 连接还没建好（或者回调还没注册），服务端的消息就可能先到了 SSE 流上。

**怎么解决？** 用 `pending_stream_messages` 做缓存。

**回调注册时补消费** [submit.ts:619-625](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L619-L625)：

```typescript
// 注册回调前，先检查有没有早到的消息
if (event_id in pending_stream_messages) {
    pending_stream_messages[event_id].forEach((msg) => callback(msg));
    delete pending_stream_messages[event_id];
}
// 注册回调
event_callbacks[event_id] = callback;
unclosed_events.add(event_id);
```

**时序图解**：

```
客户端                                服务端
   |                                     |
   |-- POST /queue/join -->|             |
   |                      |              |
   |                [服务端处理入队]     |
   |                      |              |
   |<-- {event_id: "a"} --|              |  (POST 响应返回)
   |                                     |
   |  [注册 callback["a"] = fn]          |
   |  [发现流未开，开始建连接]            |
   |                                     |
   |-- GET /queue/data?session_hash=Y ->|
   |                                     |
   |   [此时服务端可能已经发了几条消息]    |
   |                                     |
   |<-- msg1 (event_id: "a") -----------|
   |<-- msg2 (event_id: "a") -----------|
   |     这两条消息进 pending_stream_messages["a"]
   |                                     |
   |  [SSE 连接建好，onmessage 开始工作]   |
   |  [同时检查 pending，发现有缓存]       |
   |  [立即用 msg1、msg2 调用 callback]   |
   |                                     |
   |<-- msg3 (event_id: "a") -----------|  后续消息直接走 callback
```

### 4.4 单请求回调的内部处理

每个 `submit()` 调用都会创建一个专属的 callback 函数 [submit.ts:489-617](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L489-L617)：

```typescript
let callback = async function (_data: object): Promise<void> {
    const { type, status, data, original_msg } = handle_message(_data, last_status[fn_index]);
    
    if (type == "heartbeat") return;  // 心跳直接忽略
    
    if (type === "update" && status && !complete) {
        fire_event({ type: "status", ...status });  // 状态更新
    } else if (type === "complete") {
        complete = status;  // 记住完成状态，等 data 到了再 fire
    } else if (type === "generating" || type === "streaming") {
        fire_event({ type: "status", stage: status.stage, ... });
        if (sse_v2+) {
            apply_diff_stream(pending_diff_streams, event_id, data);  // 应用 diff
        }
    }
    
    if (data) {
        fire_event({ type: "data", data: handle_payload(...) });  // 数据事件
        if (complete) {
            fire_event({ type: "status", stage: "complete", ... });
            close();  // 本请求的迭代器结束（不关 SSE 连接！）
        }
    }
    
    if (status?.stage === "complete" || status?.stage === "error") {
        delete event_callbacks[event_id];     // 清理回调
        delete pending_diff_streams[event_id]; // 清理 diff 状态
        close();  // 本请求的迭代器结束
    }
};
```

> **关键区分**：`close()` 只结束本请求的 AsyncIterator（设 `done = true`），**不关闭 SSE 连接**。SSE 连接由 `close_stream` 消息控制，或者 sse_v2+ 异常时主动关闭。

### 4.5 消息类型一览

**服务端消息类型** [server_messages.py](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/server_messages.py)：

| 消息 msg 字段 | 触发时机 | 对应客户端 handle_message type |
|-------------|---------|------------------------------|
| `estimation` | 入队后，定期更新队列位置 | `update` (stage: pending) |
| `process_starts` | 任务开始执行 | `update` (stage: pending) |
| `process_generating` | 生成器中间结果 | `generating` + 可选 data |
| `process_streaming` | 流式输入输出 | `streaming` + 可选 data |
| `process_completed` | 执行完成 | `complete` + data |
| `progress` | 进度更新 | `update` |
| `log` | 日志消息 | `log` |
| `heartbeat` | 定期保活 | `heartbeat` (忽略) |
| `close_stream` | 服务端要求关闭流 | 直接关闭，不进回调 |
| `unexpected_error` | 未预期异常 | `unexpected_error` |

**仅客户端识别但服务端不再发送的消息** [api_info.ts:256-259](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/helpers/api_info.ts#L256-L259)：

| 消息 msg 字段 | handle_message type | 说明 |
|-------------|---------------------|------|
| `send_data` | `data` | 旧版 sse 协议使用，当前服务端不发送 |
| `send_hash` | `hash` | 旧版 sse 协议使用，当前服务端不发送 |
| `queue_full` | `update` (error) | 现由 503 状态码替代 |

**客户端事件类型** [types.ts:358-435](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/types.ts#L358-L435)：

| 事件 type | 说明 | 触发条件 |
|----------|------|---------|
| `data` | 输出数据 | process_generating / process_completed 带 output 时 |
| `status` | 状态变化 | estimation / process_starts / complete / error 等 |
| `log` | 日志消息 | log 消息 |
| `render` | 动态渲染 | 服务端返回 render_config 时 |

**Status.stage 枚举**：
- `pending`: 排队中 / 刚开始处理
- `generating`: 生成器输出中
- `streaming`: 流式输出中
- `complete`: 完成
- `error`: 错误

---

## 5. 服务端消息推送机制

### 5.1 消息如何从服务端发到 SSE

**服务端数据流**：

```
工作线程 process_events()
   └─> Queue.send_message(event, message)
          └─> message.event_id = event._id
          └─> pending_messages_per_session[session_hash].put_nowait(message)
                      │
                      ▼
          queue_data_helper() SSE 循环
              └─> 从 AsyncQueue 取消息
              └─> 格式化为 SSE data 帧
              └─> yield 给客户端
```

**send_message()** [queueing.py:240-249](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L240-L249)：

```python
def send_message(self, event, event_message):
    if not event.alive:
        return
    event_message.event_id = event._id  # 打上 event_id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)  # 放入会话消息队列
```

**SSE 推送循环** [routes.py:1490-1572](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1490-L1572)：

```python
async def sse_stream(request):
    heartbeat_task = asyncio.create_task(heartbeat())
    while True:
        if await request.is_disconnected():
            await blocks._queue.clean_events(session_hash=session_hash)
            return
        
        # 从队列取消息，超时 10 秒
        message = await asyncio.wait_for(
            pending_messages_per_session[session_hash].get(), timeout=10
        )
        
        if message:
            yield process_msg(message)
            
            # process_completed 后检查是否所有事件都完成了
            if isinstance(message, ProcessCompletedMessage) and message.event_id:
                pending_event_ids_session[session_hash].remove(message.event_id)
                if len(pending_event_ids_session[session_hash]) == 0:
                    yield process_msg(CloseStreamMessage())
                    return
```

### 5.2 心跳机制

**服务端心跳** [routes.py:1481-1488](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/routes.py#L1481-L1488)：

```python
async def heartbeat():
    while blocks.is_running:
        await asyncio.sleep(heartbeat_rate)  # 默认 15 秒
        queue = blocks._queue.pending_messages_per_session.get(session_hash)
        if queue:
            await queue.put(HeartbeatMessage())
```

**客户端处理** [submit.ts:496-498](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L496-L498)：

```typescript
if (type == "heartbeat") {
    return;  // 直接忽略，不向上层传递
}
```

> 心跳消息没有 `event_id`，在 [stream.ts:48-53](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L48-L53) 中被广播给所有回调，但每个回调都直接 return 忽略。

### 5.3 广播 vs 单播

| 消息类型 | 是否带 event_id | 分发方式 |
|---------|----------------|---------|
| estimation | ✅ 是 | 按 event_id 单播 |
| process_starts | ✅ 是 | 按 event_id 单播 |
| process_generating | ✅ 是 | 按 event_id 单播 |
| process_completed | ✅ 是 | 按 event_id 单播 |
| progress | ✅ 是 | 按 event_id 单播 |
| log | ✅ 是 | 按 event_id 单播 |
| heartbeat | ❌ 否 | 广播给所有回调（被忽略） |
| close_stream | ❌ 否 | 全局处理，关闭连接 |
| unexpected_error | 视情况 | 可能无 event_id |

---

## 6. 完整时序图

### 6.1 当前有效协议（sse_v1+）完整流程

```
客户端 (Client)                          服务端 (Server)
     |                                         |
     |  第1个请求 submit()                     |
     |  POST /queue/join { session_hash, fn_index, data }
     |---------------------------------------->|
     |                                         |
     |  { event_id: "evt_a" }                  |
     |<----------------------------------------|
     |                                         |
     |  注册 event_callbacks["evt_a"] = cb_a   |
     |  发现 stream_status.open = false        |
     |  GET /queue/data?session_hash=Y         |
     |---------------------------------------->|  (建立 SSE 长连接)
     |                                         |
     |  msg: estimation (event_id: "evt_a")   |
     |<----------------------------------------|
     |    → cb_a 处理，fire status event       |
     |                                         |
     |  第2个请求 submit() 并发发起             |
     |  POST /queue/join { session_hash, fn_index, data }
     |---------------------------------------->|
     |                                         |
     |  { event_id: "evt_b" }                  |
     |<----------------------------------------|
     |                                         |
     |  注册 event_callbacks["evt_b"] = cb_b   |
     |  流已打开，不用新建                      |
     |                                         |
     |  msg: process_starts (event_id: "evt_a")|
     |<----------------------------------------|
     |    → cb_a 处理                           |
     |                                         |
     |  msg: estimation (event_id: "evt_b")   |
     |<----------------------------------------|
     |    → cb_b 处理                           |
     |                                         |
     |  msg: process_completed (event_id: "evt_a")
     |<----------------------------------------|
     |    → cb_a 处理，fire data + status      |
     |    → delete event_callbacks["evt_a"]    |
     |    → evt_a 的 iterator close()          |
     |    → SSE 连接仍然保持                    |
     |                                         |
     |  msg: process_completed (event_id: "evt_b")
     |<----------------------------------------|
     |    → cb_b 处理，fire data + status      |
     |    → delete event_callbacks["evt_b"]    |
     |    → evt_b 的 iterator close()          |
     |                                         |
     |  服务端检测：pending_event_ids 为空      |
     |  msg: close_stream                      |
     |<----------------------------------------|
     |    → stream_status.open = false          |
     |    → SSE 连接关闭                        |
```

### 6.2 取消任务流程

```
客户端                                服务端
   |                                     |
   |  1. POST /cancel                    |
   |     { event_id, session_hash, fn_index }
   |------------------------------------>|
   |                                     |
   |  2. 服务端处理：                     |
   |     - 从队列移除（如果还在排队）     |
   |     - 或取消正在运行的任务           |
   |                                     |
   |  3. 发送 ProcessCompletedMessage    |
   |     success=True, output={}         |
   |<------------------------------------|
   |     (走正常 SSE 分发路径)            |
   |                                     |
   |  4. POST /reset                     |
   |     { event_id }                    |
   |------------------------------------>|
   |     重置迭代器状态                   |
```

---

## 7. 关键端点汇总

| 端点 | 方法 | 用途 | 适用协议 |
|------|------|------|---------|
| `/config` | GET | 获取应用配置（含 protocol 字段） | 所有 |
| `/info` | GET | 获取 API 信息（端点参数） | 所有 |
| `/upload` | POST | 上传文件 | 所有 |
| `/run/{endpoint}` | POST | 非队列模式直接执行 | 非队列 |
| `/queue/join` | POST | 队列模式提交任务，返回 event_id | sse_v1+（当前唯一有效） |
| `/queue/data` | GET | SSE 长连接接收结果（仅接收 session_hash） | sse_v1+ |
| `/call/{api_name}` | POST | 简化版队列提交（自动映射参数名） | sse_v1+ |
| `/call/{api_name}/{event_id}` | GET | 简化版 SSE 接收 | sse_v1+ |
| `/cancel` | POST | 取消任务 | 所有队列模式 |
| `/reset` | POST | 重置迭代器状态 | 所有队列模式 |
| `/heartbeat/{session_hash}` | GET | 心跳保活（state 相关） | 所有 |
| `/component_server` | POST | 组件服务端方法调用 | 所有 |

---

## 8. 错误处理

### 8.1 客户端错误场景

| 场景 | 状态码 | 处理方式 |
|------|-------|---------|
| 队列已满 | 503 | 发送 status stage="error"，message="Queue is full" |
| 验证失败 | 422 | 发送 status stage="error"，code="validation_error" |
| 连接断开 | - | 发送 status stage="error"，broken=true |
| 会话未找到 | 404 | 发送 status stage="error"，session_not_found=true |
| 服务端异常 | 500 | 发送 status stage="error" |

### 8.2 服务端错误处理

- **业务异常**（`gr.Error`）：包装为 `ProcessCompletedMessage(success=False)`，在 SSE 流中正常返回
- **未预期异常**：发送 `UnexpectedErrorMessage`
- **队列停止**：POST /queue/join 返回 503 `Queue is stopped`
- **客户端断开**：SSE 循环检测到断开，清理会话事件

### 8.3 sse_v2+ 异常时的连接关闭

[submit.ts:611-615](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L611-L615)：

```typescript
// 回调内异常时，sse_v2+ 主动关闭 SSE 连接
if (["sse_v2", "sse_v2.1", "sse_v3"].includes(protocol)) {
    close_stream(stream_status, that.abort_controller);
    stream_status.open = false;
    close();
}
```

sse_v1 不会主动关闭 SSE 连接，只关闭本请求的迭代器。

---

## 9. 核心代码参考

### 9.1 协议分支入口

[submit.ts:77-79](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L77-L79)
```typescript
let protocol = config.protocol ?? "ws";
if (protocol === "ws") {
    throw new Error("WebSocket protocol is not supported in this version");
}
```

### 9.2 非队列模式提交

[submit.ts:188-265](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L188-L265)
```typescript
if (skip_queue(fn_index, config)) {
    post_data(`${config.root}${api_prefix}/run${_endpoint}`, { ...payload, session_hash })
        .then(([output, status_code]) => {
            if (status_code == 200) {
                fire_event({ type: "data", ... });
                fire_event({ type: "status", stage: "complete", ... });
            }
        });
}
```

### 9.3 sse_v1+ 提交 + 回调注册

[submit.ts:429-437](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L429-L437)
```typescript
post_data(`${config.root}${api_prefix}/${SSE_DATA_URL}?${url_params}`, {
    ...payload, session_hash
}).then(async ([response, status]) => {
    event_id = response.event_id;
    // ... 注册 callback，建 SSE 流
});
```

### 9.4 会话级 SSE 消息分发

[stream.ts:41-77](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/stream.ts#L41-L77)
```typescript
stream.onmessage = async function (event) {
    let _data = JSON.parse(event.data);
    if (_data.msg === "close_stream") {
        close_stream(stream_status, that.abort_controller);
        return;
    }
    const event_id = _data.event_id;
    if (!event_id) {
        // 广播
    } else if (event_callbacks[event_id]) {
        event_callbacks[event_id](_data);
    } else {
        pending_stream_messages[event_id].push(_data);
    }
};
```

### 9.5 回调注册与竞态处理

[submit.ts:619-628](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/client/js/src/utils/submit.ts#L619-L628)
```typescript
if (event_id in pending_stream_messages) {
    pending_stream_messages[event_id].forEach((msg) => callback(msg));
    delete pending_stream_messages[event_id];
}
event_callbacks[event_id] = callback;
unclosed_events.add(event_id);
if (!stream_status.open) {
    await this.open_stream();
}
```

### 9.6 服务端消息推送

[queueing.py:240-249](file:///d:/fz/0601/solo-dogfeeding/code/247-gradio/gradio/queueing.py#L240-L249)
```python
def send_message(self, event, event_message):
    if not event.alive:
        return
    event_message.event_id = event._id
    messages = self.pending_messages_per_session[event.session_hash]
    messages.put_nowait(event_message)
```
