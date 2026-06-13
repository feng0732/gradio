# Spaces 部署适配代码路径分析

## 一、环境探测

### 1.1 核心探测逻辑

Spaces 环境探测主要通过环境变量检测实现，核心代码位于 [gradio/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py#L563-L570)：

```python
def get_space() -> str | None:
    if os.getenv("SYSTEM") == "spaces":
        return os.getenv("SPACE_ID")
    return None

def is_zero_gpu_space() -> bool:
    return os.getenv("SPACES_ZERO_GPU") == "true"
```

### 1.2 环境变量说明

| 环境变量 | 预期值 | 含义 |
|---------|--------|------|
| `SYSTEM` | `spaces` | 标识运行在 Hugging Face Spaces 环境中 |
| `SPACE_ID` | `username/space_name` | Space 的唯一标识符 |
| `SPACES_ZERO_GPU` | `true` | 标识为 ZeroGPU 环境 |
| `SPACE_HOST` | 主机名 | Space 访问域名（支持逗号分隔多域名） |

### 1.3 调用链

1. **Blocks 初始化** - [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/blocks.py#L1140-L1142)
   ```python
   self.api_open = utils.get_space() is None  # Space 环境下 API 不开放
   self.space_id = utils.get_space()          # 保存 space_id 到实例
   ```

2. **PWA 自动启用** - [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/blocks.py#L2822)
   ```python
   self.pwa = utils.get_space() is not None if pwa is None else pwa
   ```

3. **Share 禁用** - [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/blocks.py#L3115-L3117)
   ```python
   if self.share and self.space_id:
       warnings.warn("Setting share=True is not supported on Hugging Face Spaces")
       self.share = False
   ```

4. **配置下发** - [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/blocks.py#L2396-L2400)
   ```python
   config = {
       "space_id": self.space_id,
       "is_colab": utils.colab_check(),
       "is_space": self.space_id is not None,  # 前端判断字段
       ...
   }
   ```

5. **部署命令防护** - [gradio/cli/commands/deploy_space.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/cli/commands/deploy_space.py#L273-L274)
   ```python
   if os.getenv("SYSTEM") == "spaces":
       return  # 在 Space 内运行时阻止重复部署
   ```

### 1.4 前端接收

前端在配置中接收 `space_id` 和 `is_space` 字段：
- [js/app/src/routes/[...catchall]/+page.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.ts#L65-L82)
- [js/app/src/routes/[...catchall]/+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L286)
  ```javascript
  window.__gradio_space__ = config.space_id;  // 全局挂载
  ```

---

## 二、页面嵌入 (iframe)

### 2.1 嵌入检测逻辑

嵌入检测主要通过以下机制：

1. **Props 传递** - `is_embed` 参数由上层传入 [+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L102-L119)

2. **窗口上下文检测** - [+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L369)
   ```javascript
   if (space_id && !is_embed && window.self === window.top) {
       // 在顶层窗口且非嵌入模式时，加载 Space 头部导航
       const header = await init(space_id);
   }
   ```

### 2.2 嵌入模式下的行为差异

1. **主题作用域** - [+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L159-L160)
   ```javascript
   const dark_class_element = is_embed ? target.parentElement! : document.body;
   const bg_element = is_embed ? target : target.parentElement!;
   ```

2. **容器样式** - [Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Embed.svelte#L99-L107)
   ```svelte
   <div
       class:embed-container={display}
       class:with-info={info}
       data-iframe-height  <!-- 标记供 iframe 自动调整高度 -->
   >
   ```

3. **信息栏显示** - [Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Embed.svelte#L134-L154)
   ```svelte
   {#if display && space && info}
       <div class="info">
           <span><a href="https://huggingface.co/spaces/{space}">{space}</a></span>
           <span>Built with <a href="https://gradio.app">Gradio</a>.</span>
           <span>Hosted on <a href="https://huggingface.co/spaces">Spaces</a></span>
       </div>
   {/if}
   ```

### 2.3 嵌入相关组件

核心嵌入组件：[js/core/src/Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Embed.svelte)

关键 Props：
- `is_embed`: boolean - 是否为嵌入模式
- `display`: boolean - 是否显示嵌入容器样式
- `info`: boolean - 是否显示底部信息栏
- `space`: string | null - Space ID
- `initial_height`: string - 初始高度
- `fill_width`: boolean - 是否填满宽度

---

## 三、心跳处理

### 3.1 心跳启用条件

心跳连接并非始终启用，由 [connect_heartbeat](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py#L1636-L1684) 函数判断：

```python
def connect_heartbeat(config: BlocksConfigDict, blocks, fns=None) -> bool:
    any_state = any(isinstance(block, State) for block in blocks)
    any_unload = any(target[1] == "unload" for dep in config["dependencies"])
    any_stream = any(target[1] == "stream" for dep in config["dependencies"])
    any_per_session_cache = any(fn.cache._per_session for fn in fns)
    
    return any_state or any_unload or any_stream or any_per_session_cache
```

**启用条件**（满足任一）：
- 存在 `State` 组件
- 存在 `unload` 事件处理
- 存在 `stream` 流式输出
- 存在会话级缓存

### 3.2 心跳频率配置

[get_heartbeat_rate](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py#L1601-L1633)：

```python
def get_heartbeat_rate() -> float:
    interval = os.getenv("GRADIO_HEARTBEAT_INTERVAL")
    if interval:
        rate = float(interval)
        if rate > 0:
            return rate
    return 0.25 if os.getenv("GRADIO_IS_E2E_TEST") else 15  # 默认 15 秒
```

### 3.3 服务端实现

核心路由：[/heartbeat/{session_hash}](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py#L1195-L1254)

```python
@router.get("/heartbeat/{session_hash}")
def heartbeat(session_hash: str, request: fastapi.Request, ...):
    heartbeat_rate = utils.get_heartbeat_rate()
    
    async def iterator():
        stop_stream_task = asyncio.create_task(app.stop_event.wait())
        while True:
            try:
                yield "data: ALIVE\n\n"
                wait_task = asyncio.create_task(asyncio.sleep(heartbeat_rate))
                done, _ = await asyncio.wait(
                    [wait_task, stop_stream_task],
                    return_when=asyncio.FIRST_COMPLETED,
                )
                if stop_stream_task in done:
                    raise asyncio.CancelledError()
            except asyncio.CancelledError:
                # 客户端断开连接时的清理逻辑
                # 1. 执行 unload 事件
                # 2. 标记 session 关闭（1小时后删除）
                # 3. 清理会话缓存
                app.state_holder.session_data[session_hash].is_closed = True
                caching.clear_session_caches(session_hash)
```

**关键特性**：
- 使用 SSE (Server-Sent Events) 长连接
- 定期发送 `ALIVE` 消息保活
- 连接断开时触发清理逻辑
- 支持服务器停止时优雅关闭

### 3.4 客户端实现

[client.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/client/js/src/client.ts#L238-L273)：

```javascript
async _resolve_heartbeat(_config: Config): Promise<void> {
    if (this.config && this.config.connect_heartbeat) {
        // Space 环境且有 token 时获取 JWT
        if (this.config.space_id && this.options.token) {
            this.jwt = await get_jwt(this.config.space_id, this.options.token);
        }
        
        const heartbeat_url = new URL(
            `${this.config.root}${this.api_prefix}/${HEARTBEAT_URL}/${this.session_hash}`
        );
        
        if (this.jwt) {
            heartbeat_url.searchParams.set("__sign", this.jwt);
        }
        
        if (!this.heartbeat_event) {
            this.heartbeat_event = this.stream(heartbeat_url);
        }
    }
}
```

### 3.5 消息类型定义

[server_messages.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/server_messages.py#L65-L66)：

```python
class HeartbeatMessage(BaseMessage):
    msg: Literal[ServerMessage.heartbeat] = ServerMessage.heartbeat
```

---

## 四、分支决策总览

```
启动流程
├─ 环境探测 (get_space)
│   ├─ SYSTEM == "spaces" ?
│   │   ├─ 是：设置 space_id，禁用 share，启用 PWA
│   │   └─ 否：常规模式
│   └─ SPACES_ZERO_GPU == "true" ?
│       └─ 是：ZeroGPU 特殊处理
│
├─ 页面嵌入
│   ├─ is_embed == true ?
│   │   ├─ 是：使用嵌入容器样式，不显示 Space 头部
│   │   └─ 否：完整模式，window.top == self 时加载 Space 头部
│   └─ info == true ?
│       └─ 是：显示底部 "Hosted on Spaces" 信息栏
│
└─ 心跳处理
    ├─ connect_heartbeat 判断
    │   ├─ State 组件 ?
    │   ├─ unload 事件 ?
    │   ├─ stream 输出 ?
    │   └─ 会话级缓存 ?
    ├─ 任一满足：建立 SSE 长连接
    │   ├─ Space + token：附加 __sign JWT 参数
    │   └─ 默认间隔 15 秒发送 ALIVE
    └─ 断开时清理：unload 事件 → 标记关闭 → 清理缓存
```

## 五、关键代码文件索引

| 模块 | 文件路径 | 核心行号 |
|------|---------|---------|
| 环境探测 | [gradio/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py) | L563-L570 |
| Space ID 注入 | [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/blocks.py) | L1140-L1142, L2396, L2822 |
| 嵌入组件 | [js/core/src/Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Embed.svelte) | L99-L157 |
| 嵌入逻辑 | [+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte) | L365-L377 |
| 心跳判断 | [gradio/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py) | L1601-L1684 |
| 心跳服务端 | [gradio/routes.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py) | L1195-L1254 |
| 心跳客户端 | [client/js/src/client.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/client/js/src/client.ts) | L238-L273 |
| Space 状态检测 | [client/js/src/helpers/spaces.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/client/js/src/helpers/spaces.ts) | L9-L151 |
| 部署命令 | [gradio/cli/commands/deploy_space.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/cli/commands/deploy_space.py) | L252-L321 |
