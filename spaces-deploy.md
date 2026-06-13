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

### 2.4 页面嵌入的三大入口及参数传递

Gradio 提供三种运行模式，参数来源各不同：

#### 入口 A：SSR 模式（SvelteKit 应用）

文件：[js/app/src/routes/[...catchall]/+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L434-L472)

Props 由上层 Layout 或路由传入，最终传递给 Embed 和 Blocks：

```svelte
<Embed
    display={container && is_embed}   <!-- 嵌入 + container 同时为 true 才启用外框 -->
    {is_embed}                         <!-- 来自 Props，默认 false -->
    info={false}                       <!-- SSR 入口下 info 固定为 false（信息栏不通过 Embed 显示） -->
    {space}                            <!-- 来自 app.config.space_id -->
    ...
/>
<Blocks
    fill_height={!is_embed && config.fill_height}  <!-- 嵌入时禁用整页填充 -->
    footer_links={is_embed ? [] : config.footer_links}  <!-- 嵌入时隐藏 footer_links -->
    ...
/>
```

**SSR 入口关键参数：**

| Props | 默认值 | 来源 | 作用 |
|-------|--------|------|------|
| `is_embed` | `false` | 调用方传入 | 控制嵌入/整页模式切换 |
| `container` | 必填 | 调用方传入 | 与 `is_embed` 组合控制 `display` |
| `control_page_title` | `true` | 调用方传入 | 是否允许 JS 修改 document.title |
| `initial_height` | 必填 | 调用方传入 | 未加载完成前的占位高度 |
| `space` | 必填 | `app.config.space_id` | 用于 Space 头部导航加载 |

#### 入口 B：SPA 模式（Custom Element）

文件：[js/spa/src/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/spa/src/main.ts#L44-L195)

通过 `<gradio-app>` Web Component 的 HTML 属性传入，再映射到 Index.svelte Props：

HTML 用法：
```html
<gradio-app
    space="username/space-name"
    src="https://xxx.hf.space"
    embed="true"
    info="true"
    container="true"
    initial_height="500px"
    autoscroll="true"
    eager="false"
    theme_mode="dark"
    control_page_title="false"
></gradio-app>
```

属性到 Props 的映射（[main.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/spa/src/main.ts#L60-L126)）：

| HTML 属性 | 默认值 | Props | 转换逻辑 |
|-----------|--------|-------|---------|
| `space` | `null` | `space` | 直接传入（trim） |
| `src` | `null` | `src` | 直接传入（trim），优先级低于 `space` |
| `embed` | `"true"` | `is_embed` | 等于 `"false"` 时为 `false`，否则 `true`（**默认嵌入**） |
| `container` | `"true"` | `container` | 同上 |
| `info` | `true` | `info` | 同上（**默认显示信息栏**） |
| `initial_height` | `"300px"` | `initial_height` | 直接传入 |
| `autoscroll` | `null` | `autoscroll` | `"true"` 时为 `true` |
| `eager` | `null` | `eager` | `"true"` 时为 `true` |
| `theme_mode` | `null` | `theme_mode` | 直接传入（`"dark"` / `"light"` / `"system"`） |
| `control_page_title` | `null` | `control_page_title` | `"true"` 时为 `true` |

> **注意**：SPA 入口默认 `is_embed=true`、`info=true`、`container=true`，和 SSR 入口恰好相反。

SPA 内部再传递给 Embed 组件（[Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/spa/src/Index.svelte#L100-L117)）：
```
is_embed → 原样传给 Embed 的 is_embed
container + is_embed → Embed 的 display（与 SSR 相同规则）
info → Embed 的 info（与 SSR 不同，SPA 默认 true）
```

#### 入口 C：URL 查询参数（两种入口通用）

| 参数 | 作用 | 代码位置 |
|------|------|---------|
| `?__theme=dark` / `?__theme=light` | 强制主题覆盖，优先级高于 `theme_mode` Props | [+page.svelte#L126-L128](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L126-L128) / [Index.svelte#L236-L238](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/spa/src/Index.svelte#L236-L238) |
| `?deep_link=xxx` | 恢复共享的组件状态 | [+page.ts#L36](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.ts#L36) |
| `?view=api` | 直接打开 API 文档（需 `footer_links` 含 "api"） | [Blocks.svelte#L269](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Blocks.svelte#L269) |
| `?view=settings` | 直接打开设置面板 | [Blocks.svelte#L271](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Blocks.svelte#L271) |

### 2.5 底部信息栏显示时机（两种 Footer 对比）

Gradio 存在**两种独立**的底部信息栏，分属不同组件、互不影响：

#### 类型 A：Embed 信息栏（Hosted on Spaces 标识）

- 所属组件：[js/core/src/Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Embed.svelte#L134-L154)
- 显示内容：Space 链接 + "Built with Gradio" + "Hosted on Spaces"
- 显示条件：`display && space && info`（三者同时为 true）

| 条件 | 含义 | 说明 |
|------|------|------|
| `display` | `container && is_embed` | 必须同时启用嵌入模式和容器外框 |
| `space` | `config.space_id` | 必须是在 Space 环境（`SYSTEM=spaces`）|
| `info` | Props 传入 | SSR 入口固定 `false`，SPA 入口默认 `true` |

**何时显示：**

| 入口 | is_embed | container | space_id | info | 结果 |
|------|----------|-----------|----------|------|------|
| SSR 常规访问（hf.space 直接打开） | false | true | 有 | false | ❌ 不显示 |
| SSR + embed=true 外部嵌入 | true | true | 有 | false | ❌ 不显示（info 固定 false）|
| SPA（<gradio-app> Web Component）默认 | true | true | 有 | true | ✅ 显示 |
| SPA + info=false | true | true | 有 | false | ❌ 不显示 |
| SPA 嵌入非 Space 应用 | true | true | 无 | true | ❌ 不显示（space 为空）|

#### 类型 B：Blocks 底部链接（API / Gradio / Settings）

- 所属组件：[js/core/src/Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Blocks.svelte#L482-L551)
- 显示内容：API 文档按钮 + "Built with Gradio" + Settings 按钮
- 显示条件：`footer_links.length > 0`（由 `launch(footer_links=...)` 或 `mount_gradio_app(footer_links=...)` 控制）

**服务端默认值：** [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py#L2511-L2512)
```python
if footer_links is None:
    footer_links = ["api", "gradio", "settings"]
```

可选值：
- `"api"` → 显示 API 文档入口
- `"gradio"` → 显示 "Built with Gradio" 链接
- `"settings"` → 显示设置按钮
- `dict` → 自定义链接 `{"label": "...", "url": "..."}`

**嵌入时的强制覆盖：** [+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte#L469)
```javascript
footer_links={is_embed ? [] : config.footer_links}
// 嵌入模式下 footer_links 强制清空 = 类型 B 不显示
```

**类型 B 显示时机：**

| 场景 | footer_links 实际值 | 结果 |
|------|---------------------|------|
| 直接访问（默认） | `["api", "gradio", "settings"]` | ✅ 完整显示 |
| 直接访问 + `footer_links=[]` | `[]` | ❌ 完全隐藏 |
| 直接访问 + `footer_links=["gradio"]` | `["gradio"]` | ⚡ 只显示 Built with Gradio |
| SSR 嵌入（is_embed=true） | `[]`（被强制覆盖） | ❌ 完全隐藏 |
| SPA 嵌入（Web Component） | 取决于 config（不强制覆盖） | ⚡ 若 Space 未显式隐藏则显示 |

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

## 四、登录跳转分支（OAuth）

当 Space 启用了 `hf_oauth: true`（在 README 元数据中），Gradio 会通过 `attach_oauth()` 附加 OAuth 路由。核心代码位于 [gradio/oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py)。

### 4.1 OAuth 路由分支：Space vs 本地

[attach_oauth()](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py#L25-L54) 中首先根据环境选择分支：

```python
def attach_oauth(app: fastapi.FastAPI):
    if get_space() is not None:      # ★ 分支判断
        _add_oauth_routes(app)       # Space 环境：真实 OAuth
    else:
        _add_mocked_oauth_routes(app) # 本地开发：Mock 用户

    # Session Middleware 配置（两个分支共用）
    app.add_middleware(
        SessionMiddleware,
        secret_key=hashlib.sha256(session_secret.encode()).hexdigest(),
        same_site="none",            # 跨站 iframe 必须允许
        https_only=True,             # 强制 HTTPS
    )
```

| 分支 | 函数 | 用户来源 | 说明 |
|------|------|---------|------|
| Space 环境 | `_add_oauth_routes()` | 真实 Hugging Face OpenID | 要求设置 4 个环境变量：`OAUTH_CLIENT_ID`、`OAUTH_CLIENT_SECRET`、`OAUTH_SCOPES`、`OPENID_PROVIDER_URL`（由 Spaces 平台自动注入）|
| 本地开发 | `_add_mocked_oauth_routes()` | 本机 `huggingface-cli login` 的用户 | 生成 8 小时有效的 Mock token，方便本地调试 |

### 4.2 登录流程：SPACE_HOST 与自定义域名

用户点击 `gr.LoginButton()` → 跳转到 `/login/huggingface` → [oauth_login()](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py#L94-L99)

在生成回调 URL 的 `_generate_redirect_uri()` [L197-L221](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py#L197-L221) 中有一个重要分支：

```python
def _generate_redirect_uri(request: fastapi.Request) -> str:
    # ... 构造 target（登录后跳转的应用内路径）...

    if space_host := os.getenv("SPACE_HOST"):       # ★ 分支判断
        space_host = space_host.split(",")[0]       # 取第一个（自定义域名时会有逗号分隔）
        redirect_uri = (
            f"https://{space_host}/login/callback?"
            + urllib.parse.urlencode({"_target_url": target})
        )
        return redirect_uri                          # 强制使用官方 hf.space 域名
    else:
        # 本地开发：用当前请求的 host 构造
        return request.url_for("oauth_redirect_callback")...
```

**为什么强制使用 SPACE_HOST？**

> Hugging Face OAuth App 的回调 URL 必须是预先注册的 `*.hf.space` 域名。当用户通过**自定义域名**（如 `demo.mydomain.com`，Spaces 支持配置）访问时，OAuth 回调必须回到 `xxx.hf.space/login/callback` 才能通过鉴权，然后再由服务端 `_redirect_to_target()` 携带 session cookie 跳回应用路径。

### 4.3 Cookie 失效（iframe 场景）与 MAX_REDIRECTS 逃生门

当应用被嵌入到第三方站点的 iframe 中时，浏览器的**第三方 Cookie 拦截策略**（如 Safari ITP、Chrome 的限制）会阻止 OAuth 的 session cookie 写入，导致状态比对失败。

此场景由 [oauth_redirect_callback()](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py#L101-L147) 中的 `MismatchingStateError` 捕获：

```python
except MismatchingStateError:
    # 清理损坏的 session state
    for key in list(request.session.keys()):
        if key.startswith("_state_huggingface"):
            request.session.pop(key)

    nb_redirects = int(request.query_params.get("_nb_redirects", 0))
    target_url = request.query_params.get("_target_url")
    query_params = {"_nb_redirects": nb_redirects + 1}
    if target_url:
        query_params["_target_url"] = target_url
    login_uri = f"/login/huggingface?{urllib.parse.urlencode(query_params)}"

    # ★★★ 核心分支：超过 MAX_REDIRECTS（=2）判定为 iframe 中 Cookie 失效
    if nb_redirects > MAX_REDIRECTS:                    # 第 3 次重试时触发
        host = os.environ.get("SPACE_HOST")
        host_url = "https://" + host.rstrip("/")
        return RedirectResponse(host_url + login_uri)   # ★ 跳出 iframe，
                                                        #   顶层窗口打开 Space 域名

    return RedirectResponse(login_uri)  # 否则重试登录（最多 2 次）
```

**完整的 Cookie 失效逃生流程：**

```
iframe 内点击登录
│
├─ 第 1 次 → HF OAuth → callback → state 不匹配（Cookie 没写入）
│           _nb_redirects=1 → 重试 /login/huggingface
│
├─ 第 2 次 → 再次 state 不匹配
│           _nb_redirects=2 → 再重试
│
└─ 第 3 次 → 仍 state 不匹配
            _nb_redirects=3 > MAX_REDIRECTS(2)
            → 重定向到 https://{SPACE_HOST}/login/huggingface（顶层窗口打开）
            → 用户在顶层窗口完成 OAuth（Cookie 可正常写入）
            → 最终跳回应用
```

### 4.4 回调后的安全重定向与 Open Redirect 防护

登录成功后调用 `_redirect_to_target()` [L224-L240](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py#L224-L240)：

```python
def _redirect_to_target(request, default_target="/") -> RedirectResponse:
    target = request.query_params.get("_target_url", default_target)
    parsed = urllib.parse.urlparse(target)
    # ★ 安全处理：剥离 scheme 和 host，只保留 path+query+fragment
    # 防止传入 _target_url=//evil.com 这种 scheme-relative URL 发起钓鱼跳转
    safe_target = "/" + (parsed.path or "").lstrip("/\\")
    if parsed.query:
        safe_target += "?" + parsed.query
    if parsed.fragment:
        safe_target += "#" + parsed.fragment
    return RedirectResponse(safe_target)
```

### 4.5 基础认证（非 OAuth）与登录跳转

当使用 `launch(auth=...)` 的用户名密码认证时（不启用 OAuth），跳转逻辑位于 [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py#L465-L477)：

```python
@app.post("/login")
async def login(request, form_data):
    if app.auth is None:
        root = route_utils.get_root_url(request, route_path="/login", ...)
        return RedirectResponse(url=root, status_code=302)  # 认证被禁用 → 回首页
    # ... 校验用户名密码，成功则写入 cookie ...
```

未认证用户访问需要认证的路由时会返回 401，前端收到后显示登录页面并携带 `auth_message`。**该分支无 iframe 特殊处理，完全依赖 Cookie。**

---

## 五、分支决策总览（完整）

```
启动流程
├─ 环境探测 (get_space)
│   ├─ SYSTEM == "spaces" ?
│   │   ├─ 是：设置 space_id，禁用 share，启用 PWA，api_open=false
│   │   └─ 否：常规模式
│   └─ SPACES_ZERO_GPU == "true" ?
│       └─ 是：ZeroGPU 特殊处理
│
├─ 页面入口选择
│   ├─ 入口 A：SSR SvelteKit (+page.svelte)
│   │   ├─ is_embed=false, container=true, info=false（默认值）
│   │   ├─ footer_links 嵌入时被强制清空 → 类型B不显示
│   │   └─ space_id + 非嵌入 + 顶层窗口 → 加载 Space Header
│   │
│   ├─ 入口 B：SPA Custom Element (<gradio-app>)
│   │   ├─ embed=true, container=true, info=true（默认值，与 SSR 相反）
│   │   ├─ footer_links 不强制覆盖 → 按服务端配置显示
│   │   └─ display && space && info → 显示类型A（Hosted on Spaces）信息栏
│   │
│   └─ URL 查询参数
│       ├─ ?__theme=dark|light → 覆盖主题
│       ├─ ?deep_link=xxx → 恢复共享状态
│       └─ ?view=api|settings → 打开对应面板
│
├─ 底部 Footer 显示
│   ├─ 类型A（Embed.svelte info栏）：display && space && info
│   │   ├─ SSR 入口：始终不显示（info=false）
│   │   └─ SPA 入口：默认显示（info=true），需 info=false 才隐藏
│   │
│   └─ 类型B（Blocks footer_links）：footer_links.length > 0
│       ├─ 默认：["api", "gradio", "settings"]
│       ├─ SSR 嵌入：强制清空 → 不显示
│       └─ SPA 嵌入：原样显示
│
├─ 心跳处理
│   ├─ connect_heartbeat 判断（任一满足即启用）
│   │   ├─ State 组件 ?
│   │   ├─ unload 事件 ?
│   │   ├─ stream 输出 ?
│   │   └─ 会话级缓存 ?
│   ├─ 任一满足：建立 SSE 长连接
│   │   ├─ Space + token：附加 __sign JWT 参数
│   │   └─ 默认间隔 15 秒发送 ALIVE
│   └─ 断开时清理：unload 事件 → 标记关闭 → 清理缓存
│
└─ OAuth 登录跳转
    ├─ 环境分支
    │   ├─ Space 环境：真实 OAuth（_add_oauth_routes）
    │   └─ 本地开发：Mock OAuth（_add_mocked_oauth_routes）
    │
    ├─ 回调 URL 分支
    │   ├─ SPACE_HOST 存在：强制使用 https://{SPACE_HOST}/login/callback
    │   │   └─ 自定义域名场景：先回到 hf.space，再 302 回应用路径
    │   └─ 本地：用当前请求 host 构造
    │
    ├─ Cookie 失效（iframe）处理
    │   ├─ MismatchingStateError 捕获
    │   ├─ _nb_redirects ≤ MAX_REDIRECTS(2)：重试登录
    │   └─ _nb_redirects > 2：顶层窗口跳转 SPACE_HOST 完成登录
    │
    └─ 回调后重定向
        └─ _redirect_to_target() 做安全剥离，防止 Open Redirect
```

---

## 六、关键代码文件索引

| 模块 | 文件路径 | 核心行号 |
|------|---------|---------|
| 环境探测 | [gradio/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py) | L563-L570 |
| Space ID 注入 | [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/blocks.py) | L1140-L1142, L2396, L2822 |
| 嵌入组件 Props | [js/core/src/Embed.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Embed.svelte) | L99-L157 |
| SSR 入口参数 & Footer 覆盖 | [+page.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/app/src/routes/[...catchall]/+page.svelte) | L102-L119, L434-L472 |
| SPA Custom Element 属性映射 | [js/spa/src/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/spa/src/main.ts) | L44-L195 |
| SPA Index.svelte Props 定义 | [js/spa/src/Index.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/spa/src/Index.svelte) | L98-L117 |
| Blocks 底部 Footer 渲染 | [js/core/src/Blocks.svelte](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/js/core/src/Blocks.svelte) | L482-L551 |
| footer_links 默认值 | [gradio/routes.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py) | L2511-L2512 |
| 心跳判断 & 频率 | [gradio/utils.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/utils.py) | L1601-L1684 |
| 心跳服务端 | [gradio/routes.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py) | L1195-L1254 |
| 心跳客户端 JWT | [client/js/src/client.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/client/js/src/client.ts) | L238-L273 |
| OAuth 路由 & iframe Cookie 逃生门 | [gradio/oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/oauth.py) | L25-L240 |
| 基础登录路由 | [gradio/routes.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/routes.py) | L408-L477 |
| Space 状态检测 | [client/js/src/helpers/spaces.ts](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/client/js/src/helpers/spaces.ts) | L9-L151 |
| 部署命令 | [gradio/cli/commands/deploy_space.py](file:///d:/fz/0601/solo-dogfeeding/code/255-gradio/gradio/cli/commands/deploy_space.py) | L252-L321 |
