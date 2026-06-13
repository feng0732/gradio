# Auth & OAuth 访问控制链路

本文档梳理 Gradio 框架中 Auth（用户名密码登录）和 OAuth（Hugging Face 登录）的完整访问控制链路，重点讲清 **页面级登录墙**、**受限接口校验**、**登录态注入** 这三层关系，以及 OAuth 的 Cookie 会话存储和过期处理的准确机制。

---

## 目录

- [整体三层防护架构](#整体三层防护架构)
- [一、Basic Auth 流程（用户名/密码登录）](#一basic-auth-流程用户名密码登录)
  - [1.1 启动配置](#11-启动配置)
  - [1.2 登录回调](#12-登录回调)
  - [1.3 Cookie 与会话存储](#13-cookie-与会话存储)
  - [1.4 第一层：页面级登录墙](#14-第一层页面级登录墙)
  - [1.5 第二层：受限接口校验（login_check 依赖）](#15-第二层受限接口校验login_check-依赖)
  - [1.6 第三层：登录态注入（get_current_user → gr.Request.username）](#16-第三层登录态注入get_current_user--grrequestusername)
  - [1.7 登出流程](#17-登出流程)
- [二、OAuth 流程（Hugging Face 登录）](#二oauth-流程hugging-face-登录)
  - [2.1 启动配置](#21-启动配置)
  - [2.2 登录重定向](#22-登录重定向)
  - [2.3 OAuth 回调](#23-oauth-回调)
  - [2.4 SessionMiddleware 的 Cookie 会话存储与过期处理](#24-sessionmiddleware-的-cookie-会话存储与过期处理)
  - [2.5 OAuth 的三层防护差异](#25-oauth-的三层防护差异)
  - [2.6 登录态注入：OAuthProfile / OAuthToken](#26-登录态注入oauthprofile--oauthtoken)
  - [2.7 登出流程](#27-登出流程)
- [三、两种认证方式对比](#三两种认证方式对比)
- [四、关键代码索引](#四关键代码索引)

---

## 整体三层防护架构

Gradio 的访问控制采用 **三层递进式防护**，每一层依赖前一层的结果：

```
┌────────────────────────────────────────────────────────────┐
│                    第一层：页面级登录墙                        │
│  GET /  →  main() 路由判断                                    │
│  ├─ 无需认证 / 已登录 → 返回完整 config（渲染应用页面）          │
│  └─ 未登录 → 返回 auth_required: true（渲染登录表单）          │
└─────────────────────┬──────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────┐
│                   第二层：受限接口校验                         │
│  Depends(login_check) 注入到所有 API 路由                     │
│  ├─ /config, /info, /openapi.json, /run/{api}               │
│  ├─ /call/{api}, /queue/join, /proxy=..., /file=...         │
│  └─ 未登录 → HTTP 401 Unauthorized                           │
└─────────────────────┬──────────────────────────────────────┘
                      │
┌─────────────────────▼──────────────────────────────────────┐
│                   第三层：登录态注入                           │
│  用户函数调用时，将身份信息注入函数参数                          │
│  ├─ Basic Auth: gr.Request.username（通过 Depends 获取）     │
│  └─ OAuth: OAuthProfile / OAuthToken（通过 special_args）    │
└────────────────────────────────────────────────────────────┘
```

**关键关系**：
- 第一层决定用户能否看到页面（HTML 级别）
- 第二层防止未登录直接调用 API（接口级别）
- 第三层让用户函数获取当前登录身份（业务级别）
- 前两层使用 FastAPI `Depends` 依赖注入实现复用
- 第三层在用户函数执行前的 `special_args` / `compile_gr_request` 阶段完成

---

## 一、Basic Auth 流程（用户名/密码登录）

### 1.1 启动配置

**入口**：[blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py) 的 `launch()` 方法

```python
def launch(
    self,
    auth: Callable[[str, str], bool] | tuple[str, str] | list[tuple[str, str]] | None = None,
    auth_message: str | None = None,
    auth_dependency: Callable[[fastapi.Request], str | None] | None = None,
    ...
):
    # auth 和 auth_dependency 互斥
    if auth is not None and auth_dependency is not None:
        raise ValueError("You cannot provide both `auth` and `auth_dependency` in launch().")
    
    # 标准化 auth 格式
    if auth and not callable(auth) and not isinstance(auth[0], tuple) and not isinstance(auth[0], list):
        self.auth = [auth]  # 单个 tuple 转为 list
    else:
        self.auth = auth
    self.auth_message = auth_message
```

`auth` 参数支持三种形式：
1. `tuple[str, str]` - 单组用户名密码
2. `list[tuple[str, str]]` - 多组用户名密码
3. `Callable[[str, str], bool]` - 自定义验证函数（同步或异步）

**App 配置**：[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py) `App.configure_app()`

```python
def configure_app(self, blocks: gradio.Blocks) -> None:
    auth = blocks.auth
    if auth is not None:
        if not callable(auth):
            self.auth = {account[0]: account[1] for account in auth}  # list[tuple] → dict
        else:
            self.auth = auth  # callable 保持原样
    else:
        self.auth = None
```

### 1.2 登录回调

**路由**：`POST /login`

位于 [routes.py:465-507](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L465-L507)

```python
@app.post("/login")
async def login(request: fastapi.Request, form_data: OAuth2PasswordRequestForm = Depends()):
    username, password = form_data.username.strip(), form_data.password
    
    # 认证逻辑：dict 查表 或 callable 自定义验证
    if (
        not callable(app.auth)
        and username in app.auth
        and compare_passwords_securely(password, app.auth[username])
    ) or (
        callable(app.auth)
        and (
            await app.auth(username, password)
            if inspect.iscoroutinefunction(app.auth)
            else app.auth(username, password)
        )
    ):
        # 登录成功：生成随机 token
        token = secrets.token_urlsafe(16)
        app.tokens[token] = username  # 服务端内存存储 token → username
        
        response = JSONResponse(content={"success": True})
        # 设置安全 Cookie（HTTPS 环境使用）
        response.set_cookie(
            key=f"access-token-{app.cookie_id}",
            value=token,
            httponly=True,
            samesite="none",
            secure=True,
        )
        # 设置非安全 Cookie（非 HTTPS 环境下降级使用）
        response.set_cookie(
            key=f"access-token-unsecure-{app.cookie_id}",
            value=token,
            httponly=True,
        )
        return response
    else:
        raise HTTPException(status_code=400, detail="Incorrect credentials.")
```

**密码安全比较**：[route_utils.py:864-865](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/route_utils.py#L864-L865)

使用 `hmac.compare_digest` 恒定时间比较，防止时序攻击：

```python
def compare_passwords_securely(input_password: str, correct_password: str) -> bool:
    return hmac.compare_digest(input_password.encode(), correct_password.encode())
```

**前端登录表单**：[Login.svelte](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/core/src/Login.svelte)

```javascript
const submit = async (): Promise<void> => {
    const formData = new FormData();
    formData.append("username", username);
    formData.append("password", password);
    
    const login_url = new URL("login", root).href;
    let response = await fetch(login_url, {
        method: "POST",
        body: formData
    });
    if (response.status === 400) {
        incorrect_credentials = true;  // 显示"用户名或密码错误"
    } else if (response.status == 200) {
        location.reload();  // Cookie 已写入，刷新页面进入应用
    }
};
```

### 1.3 Cookie 与会话存储

**Cookie 结构**：

| Cookie 名称 | HttpOnly | SameSite | Secure | 用途 |
|------------|----------|----------|--------|------|
| `access-token-{cookie_id}` | ✅ | `none` | ✅ | HTTPS 环境下的主会话 Cookie |
| `access-token-unsecure-{cookie_id}` | ✅ | 默认 | ❌ | 非 HTTPS 环境下降级使用 |

- `cookie_id`：App 启动时生成的随机值 `secrets.token_urlsafe(32)`，每个实例唯一
- 服务端维护 `app.tokens` 字典：`{random_token: username}`
- **无过期时间**：服务端内存存储，进程重启后所有会话失效

**获取当前用户（所有三层的核心函数）**：[routes.py:398-406](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L398-L406)

```python
@router.get("/user")
@router.get("/user/")
def get_current_user(request: fastapi.Request) -> str | None:
    # 优先级 1: auth_dependency（外部认证系统）
    if app.auth_dependency is not None:
        return app.auth_dependency(request)
    # 优先级 2: 安全 Cookie
    token = request.cookies.get(f"access-token-{app.cookie_id}")
    # 优先级 3: 非安全 Cookie（降级）
    token = token or request.cookies.get(f"access-token-unsecure-{app.cookie_id}")
    # 服务端查表获取 username
    return app.tokens.get(token)
```

这是三层防护的共享基础：第一层页面判断、第二层接口校验、第三层注入都会调用 `get_current_user`。

### 1.4 第一层：页面级登录墙

**实现位置**：[routes.py:603-666](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L603-L666)

```python
@app.head("/", response_class=HTMLResponse)
@app.get("/", response_class=HTMLResponse)
def main(
    request: fastapi.Request,
    user: str = Depends(get_current_user),  # 先解析身份
    page: str = "",
    deep_link: str = "",
):
    blocks = app.get_blocks()
    
    if (app.auth is None and app.auth_dependency is None) or user is not None:
        # 分支 A: 无需认证 OR 已登录 → 返回完整应用配置
        config = utils.safe_deepcopy(blocks.config)
        config["username"] = user  # 将 username 注入到前端 config
        # 过滤当前页的 components/dependencies/layout
        config["components"] = [c for c in config["components"] if c["id"] in config["page"][page]["components"]]
        config["dependencies"] = [d for d in config.get("dependencies", []) if d["id"] in config["page"][page]["dependencies"]]
        config["layout"] = config["page"][page]["layout"]
        config["current_page"] = page
        # ...
    elif app.auth_dependency:
        # 分支 B: 使用 auth_dependency 但未认证 → 返回 401（由外部系统处理登录）
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail={"error": "Not authenticated", "auth_message": blocks.auth_message},
        )
    else:
        # 分支 C: 使用 basic auth 但未登录 → 返回最小化登录墙配置
        config = {
            "auth_required": True,           # 前端识别此字段渲染登录表单
            "auth_message": blocks.auth_message,  # 自定义提示信息
            "space_id": blocks.space_id,
            "root": root,
            "page": {"": {"layout": {}}},
            "pages": [""],
            "components": [],                # 空组件列表
            "dependencies": [],              # 空事件依赖
            "current_page": "",
        }
```

**前端渲染逻辑**：
- SPA 模式：[Index.svelte:449-452](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/spa/src/Index.svelte#L449-L452)

```javascript
function load_demo(): void {
    if (config.auth_required) get_login();   // 渲染登录表单组件
    else get_blocks();                        // 渲染应用组件
}
```

- SSR 模式（SvelteKit）：
  - 服务端预检查：[+page.server.ts:25-39](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/app/src/routes/[...catchall]/+page.server.ts#L25-L39) 发起 `GET /config` 请求，若返回 401 则标记 `auth_required = true`
  - 通用加载：[+page.ts:50-93](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/app/src/routes/[...catchall]/+page.ts#L50-L93) 若 `auth_required` 则跳过 `Client.connect`，直接返回登录墙配置
  - 页面渲染：[+page.svelte:449](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/app/src/routes/[...catchall]/+page.svelte#L449) 根据 `config.auth_required` 条件渲染 `<Login>` 或 `<Blocks>`

### 1.5 第二层：受限接口校验（login_check 依赖）

**login_check 依赖函数**：[routes.py:408-419](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L408-L419)

```python
@router.get("/login_check")
@router.get("/login_check/")
def login_check(user: str = Depends(get_current_user)):
    # (无需认证) OR (已登录) → 通过，返回 None
    if (app.auth is None and app.auth_dependency is None) or user is not None:
        return
    # 未登录 → 抛出 401，FastAPI 自动终止请求
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail={"error": "Not authenticated", "auth_message": blocks.auth_message},
    )
```

**受 `login_check` 保护的 API 路由列表**（均通过 FastAPI `dependencies=[Depends(login_check)]` 注入）：

| 路由 | 文件位置 | 说明 |
|------|---------|------|
| `GET /config` | [routes.py:954-955](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L954-L955) | 获取应用配置 |
| `GET /info` | [routes.py:715-716](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L715-L716) | 获取 API 信息 |
| `GET /openapi.json` | [routes.py:749](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L749) | OpenAPI 文档 |
| `GET /dev/reload` | [routes.py:432](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L432) | 热重载 SSE |
| `POST /run/{api_name}` | [routes.py:1269-1270](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1269-L1270) | 直接调用 API（不排队） |
| `POST /api/{api_name}` | [routes.py:1271-1272](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1271-L1272) | 同上（兼容旧路径） |
| `POST /call/v2/{api_name}` | [routes.py:1318-1319](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1318-L1319) | 调用 API（V2 格式） |
| `POST /call/{api_name}` | [routes.py:1342-1343](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1342-L1343) | 调用 API（简单格式） |
| `POST /queue/join` | [routes.py:1357](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1357) | 加入执行队列 |
| `GET/HEAD /proxy={url}` | [routes.py:1055-1056](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1055-L1056) | 反向代理 |
| `GET/HEAD /file={path}` | [routes.py:1082-1083](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1082-L1083) | 文件服务 |
| `GET /file/{path}` | [routes.py:1185](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L1185) | 旧版文件接口 |

**设计要点**：`login_check` 不返回用户身份，只负责"放行或拦截"。实际用户身份由各路由单独通过 `Depends(get_current_user)` 获取，用于第三层注入。

### 1.6 第三层：登录态注入（get_current_user → gr.Request.username）

注入链路贯穿 **HTTP API → 队列 → 函数执行** 三个阶段：

#### 阶段 1：HTTP 路由层获取 username

所有需要执行用户函数的 API 路由都通过 `Depends(get_current_user)` 获取 username：

```python
# 示例：/run/{api_name} 路由 [routes.py:1273-1278]
async def predict(
    api_name: str,
    body: PredictBody,
    request: fastapi.Request,
    username: str = Depends(get_current_user),  # ← 此处获取
): ...

# 示例：/queue/join 路由 [routes.py:1357-1362]
async def queue_join(
    body: PredictBody,
    request: fastapi.Request,
    username: str = Depends(get_current_user),  # ← 此处获取
): ...
```

#### 阶段 2：封装为 gr.Request 对象

**compile_gr_request**：[route_utils.py:287-317](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/route_utils.py#L287-L317)

```python
def compile_gr_request(body, fn, username, request):
    gr_request = Request(
        username=username,        # ← username 注入到 gr.Request
        request=body.request or request,
        session_hash=body.session_hash,
    )
    return gr_request
```

**gr.Request 类定义**：[route_utils.py:141-183](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/route_utils.py#L141-L183)

```python
class Request:
    def __init__(
        self,
        request: fastapi.Request | None = None,
        username: str | None = None,   # ← 登录用户名（Basic Auth）
        session_hash: str | None = None,
        **kwargs,
    ):
        self.request = request
        self.username = username        # 存储在实例属性中
        self.session_hash = session_hash
```

#### 阶段 3：队列传递 username

如果请求走队列，username 会随 Event 对象一起入队：

- 入队：[queueing.py:279-280](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/queueing.py#L279-L280) `push(body, request, username)`
- 存储：[queueing.py:340-345](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/queueing.py#L340-L345) `Event(session_hash, validator_fn, request, username)`
- 出队执行：[queueing.py:812](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/queueing.py#L812) `username = events[0].username`

#### 阶段 4：函数执行前注入 special_args

在 `call_function` 中，`gr.Request` 对象通过 `special_args` 注入到用户函数参数：

[blocks.py:1636-1642](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py#L1636-L1642)

```python
processed_input, progress_index, _, _ = special_args(
    fn_to_analyze,
    processed_input,
    request,       # ← gr.Request 对象（含 username 属性）
    event_data,
    component_props=component_props,
)
```

在 `special_args` 中匹配类型注解：

[helpers.py:962-964](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/helpers.py#L962-L964)

```python
elif type_hint in (routes.Request, Optional[routes.Request]):
    if inputs is not None:
        inputs.insert(i, request)  # 将 gr.Request 注入函数参数
```

至此，用户可以在函数中通过 `request.username` 获取当前登录用户名。

### 1.7 登出流程

**路由**：`GET /logout`

位于 [routes.py:520-544](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L520-L544)

```python
@app.get("/logout")
def logout(
    request: fastapi.Request,
    user: str = Depends(get_current_user),
    all_session: bool = True,  # 默认删除该用户所有会话
):
    response = RedirectResponse(url=root, status_code=302)
    # 删除浏览器 Cookie
    response.delete_cookie(key=f"access-token-{app.cookie_id}", path="/")
    response.delete_cookie(key=f"access-token-unsecure-{app.cookie_id}", path="/")
    
    if all_session:
        # 删除该用户所有 token（所有设备同时登出）
        for token in list(app.tokens.keys()):
            if app.tokens[token] == user:
                del app.tokens[token]
    else:
        # 仅删除当前会话 token
        current_token = request.cookies.get(f"access-token-{app.cookie_id}")
        if current_token in app.tokens:
            del app.tokens[current_token]
    return response
```

---

## 二、OAuth 流程（Hugging Face 登录）

### 2.1 启动配置

**触发条件**：Blocks 中存在 `gr.LoginButton` 组件

检测逻辑：[blocks.py:1414-1419](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py#L1414-L1419)

```python
@property
def expects_oauth(self):
    """Return whether the app expects user to authenticate via OAuth."""
    return any(
        isinstance(block, components.LoginButton) for block in self.blocks.values()
    )
```

**OAuth 路由挂载**：[routes.py:513-517](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L513-L517)

```python
if app.blocks is not None and app.blocks.expects_oauth:
    attach_oauth(app)  # 挂载 OAuth 路由 + SessionMiddleware（覆盖默认 logout）
else:
    # 挂载普通 logout 路由（basic auth 用）
    @app.get("/logout")
    def logout(...): ...
```

**SessionMiddleware 配置**：[oauth.py:25-54](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L25-L54)

```python
def attach_oauth(app: fastapi.FastAPI):
    # 根据环境分支
    if get_space() is not None:
        _add_oauth_routes(app)        # Space 环境：真实 OAuth
    else:
        _add_mocked_oauth_routes(app)  # 本地开发：Mock OAuth
    
    # SessionMiddleware 签名密钥 = sha256(OAUTH_CLIENT_SECRET + "-v4")
    # "-v4" 是版本号，升级会话格式时递增可使旧 Cookie 全部失效
    session_secret = (OAUTH_CLIENT_SECRET or "") + "-v4"
    app.add_middleware(
        SessionMiddleware,
        secret_key=hashlib.sha256(session_secret.encode()).hexdigest(),
        same_site="none",
        https_only=True,
    )
```

**必需的环境变量**（Space 环境）：
- `OAUTH_CLIENT_ID` - OAuth 客户端 ID
- `OAUTH_CLIENT_SECRET` - OAuth 客户端密钥（同时作为 Session 签名密钥的种子）
- `OAUTH_SCOPES` - OAuth 权限范围（如 `"openid profile"`）
- `OPENID_PROVIDER_URL` - OpenID Provider URL，通常为 `https://huggingface.co`

### 2.2 登录重定向

**LoginButton 组件**：[login_button.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/components/login_button.py)

`activate()` 方法做两件事：
1. 绑定 JS 点击事件 → 跳转到 `/login/huggingface` 或 `/logout`
2. 绑定页面 load 事件 → `_check_login_status` 动态更新按钮文案

```python
def activate(self):
    _js = _js_handle_redirect.replace("BUTTON_DEFAULT_VALUE", json.dumps(self.value))
    self.click(fn=None, inputs=[self], outputs=None, js=_js)
    self.attach_load_event(self._check_login_status, None)
```

**前端点击重定向 JS**：

```javascript
// login_button.py _js_handle_redirect
(buttonValue) => {
    uri = buttonValue === BUTTON_DEFAULT_VALUE 
        ? '/login/huggingface?_target_url=/REDIRECT_URL'   // 未登录 → 去授权
        : '/logout?_target_url=/REDIRECT_URL';              // 已登录 → 去登出
    window.location.assign(uri + window.location.search);
}
```

**登录重定向路由**：`GET /login/huggingface`

[oauth.py:94-99](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L94-L99)

```python
@app.get("/login/huggingface")
async def oauth_login(request: fastapi.Request):
    redirect_uri = _generate_redirect_uri(request)
    # authlib 负责：生成 state/nonce，存入 session，重定向到 HF 授权页
    return await oauth.huggingface.authorize_redirect(request, redirect_uri)
```

**重定向 URI 生成**：[oauth.py:197-221](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L197-L221)

```python
def _generate_redirect_uri(request: fastapi.Request) -> str:
    # 目标页面：_target_url 查询参数 或 当前带参路径
    target = request.query_params.get("_target_url") or ("/?" + urllib.parse.urlencode(request.query_params))
    
    # Space 环境：必须回调到 hf.space 域名（自定义域名时取 SPACE_HOST 逗号分隔的第一个）
    if space_host := os.getenv("SPACE_HOST"):
        space_host = space_host.split(",")[0]
        return f"https://{space_host}/login/callback?_target_url={target}"
    
    # 本地：用 request.url_for 构造
    return str(request.url_for("oauth_redirect_callback").include_query_params(_target_url=target))
```

### 2.3 OAuth 回调

**回调路由**：`GET /login/callback`

[oauth.py:101-147](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L101-L147)

```python
@app.get("/login/callback")
async def oauth_redirect_callback(request: fastapi.Request) -> RedirectResponse:
    try:
        # authlib：验证 state，用授权码换 access_token + id_token
        oauth_info = await oauth.huggingface.authorize_access_token(request)
    except MismatchingStateError:
        # ===== State 不匹配处理（Cookie 损坏 / 第三方 Cookie 被阻止）=====
        # 1. 清理残留的 OAuth state 键
        for key in list(request.session.keys()):
            if key.startswith("_state_huggingface"):
                request.session.pop(key)
        
        nb_redirects = int(request.query_params.get("_nb_redirects", 0))
        target_url = request.query_params.get("_target_url")
        query_params = {"_nb_redirects": nb_redirects + 1}
        if target_url:
            query_params["_target_url"] = target_url
        login_uri = f"/login/huggingface?{urllib.parse.urlencode(query_params)}"
        
        # 超过 MAX_REDIRECTS=2 次，判定为 iframe 中第三方 Cookie 被阻止
        if nb_redirects > MAX_REDIRECTS:
            host = os.environ.get("SPACE_HOST")
            host_url = "https://" + host.rstrip("/")
            return RedirectResponse(host_url + login_uri)  # 跳转到顶层非 iframe 视图
        
        return RedirectResponse(login_uri)  # 重试登录
    
    # ===== 登录成功：token 信息写入 session =====
    request.session["oauth_info"] = oauth_info
    return _redirect_to_target(request)
```

**安全重定向（防开放重定向 CVE-2026-28415）**：[oauth.py:224-240](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L224-L240)

```python
def _redirect_to_target(request, default_target="/"):
    target = request.query_params.get("_target_url", default_target)
    parsed = urllib.parse.urlparse(target)
    # 剥离 scheme/host，只保留路径；lstrip 掉 4+ 个斜杠防止 "////evil.com" 绕过
    safe_target = "/" + (parsed.path or "").lstrip("/\\")
    if parsed.query:
        safe_target += "?" + parsed.query
    if parsed.fragment:
        safe_target += "#" + parsed.fragment
    return RedirectResponse(safe_target)
```

**本地 Mock OAuth 流程**：[oauth.py:156-194](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L156-L194)

本地开发时不走真实 OAuth：
- `/login/huggingface` 直接 302 到 `/login/callback?_target_url=...`
- `/login/callback` 调用 `_get_mocked_oauth_info()`（从 `huggingface_hub` 读取本机登录的账号），写入 session 后重定向

### 2.4 SessionMiddleware 的 Cookie 会话存储与过期处理

这是 OAuth 与 Basic Auth 最核心的区别。

#### SessionMiddleware 工作原理

由 `starlette.middleware.sessions.SessionMiddleware` 提供，基于签名 Cookie 的客户端会话：

```
请求到达时：
  1. 从 Cookie 中读取 "session" 值
  2. 用 secret_key 验证签名（itsdangerous.URLSafeTimedSerializer）
  3. 反序列化为 dict → 挂载到 request.session
  
响应返回时：
  1. 检查 request.session 是否被修改
  2. 序列化为 JSON + 签名
  3. 通过 Set-Cookie 写回浏览器
```

**Cookie 配置**：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| Cookie 名称 | `session` | starlette 默认 |
| 签名算法 | HMAC-SHA256 (itsdangerous) | 防篡改 |
| 签名密钥 | `sha256(OAUTH_CLIENT_SECRET + "-v4")` | 每个 Space 唯一，升级版本号可批量失效 |
| SameSite | `none` | 允许 iframe 跨站携带 |
| Secure | `true` | 仅 HTTPS 传输 |
| HttpOnly | `true`（starlette 默认） | JS 不可读 |
| 存储位置 | 客户端 Cookie | **服务端无状态** |

#### oauth_info 的数据结构

成功登录后写入 `request.session["oauth_info"]` 的完整结构：

```python
{
    "access_token": "hf_xxxxxxxxxxxx",       # API 调用凭据
    "token_type": "bearer",
    "expires_in": 3600,                       # 相对有效期（秒）
    "id_token": "eyJhbGciOi...",             # OpenID JWT
    "scope": "openid profile",                # 授权范围
    "expires_at": 1691676444,                 # 绝对过期时间戳（seconds since epoch）
    "userinfo": {                             # 用户信息
        "sub": "11111111111111111111111",     # 用户唯一 ID
        "name": "Abubakar Abid",              # 全名
        "preferred_username": "abidlabs",     # 用户名
        "profile": "https://huggingface.co/abidlabs",
        "picture": "https://.../avatar.png",
        "website": "",
        "aud": "00000000-0000-0000-0000-000000000000",
        "auth_time": 1691672844,
        "nonce": "aaaaaaaaaaaaaaaaaaa",
        "iat": 1691672844,
        "exp": 1691676444,
        "iss": "https://huggingface.co"
    }
}
```

#### 双层过期机制

OAuth 会话有 **两层过期判断**，分别由不同组件负责：

| 层级 | 检查位置 | 判断字段 | 失效动作 |
|------|---------|---------|---------|
| **第 1 层**：Cookie 签名级过期 | SessionMiddleware (starlette) | itsdangerous 的 `max_age`（默认永久） | 签名无效 → `request.session` 为空 dict |
| **第 2 层**：Token 业务级过期 | `_get_valid_oauth_info_from_session()` | `oauth_info["expires_at"]` < `time.time()` | 从 session 中删除 `oauth_info` 键，并返回 `None` |

**业务级过期检查函数**：[oauth.py:243-255](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L243-L255)

```python
def _get_valid_oauth_info_from_session(session):
    oauth_info = session.get("oauth_info")
    if oauth_info is None:
        return None
    
    expires_at = oauth_info.get("expires_at")
    if expires_at is not None and expires_at < time.time():
        # 过期了：主动从 session 中清除（下一次响应会把空 session 写回 Cookie）
        session.pop("oauth_info", None)
        return None
    
    return oauth_info
```

**检查时机**：
1. LoginButton 页面加载时 `_check_login_status()` → 刷新按钮显示（登录/登出）
2. 每次用户函数执行时 `special_args()` 注入 `OAuthProfile`/`OAuthToken` → 过期则视为未登录

> 注意：SessionMiddleware 本身 **不设置** Cookie 的 `max-age`，默认是"会话 Cookie"（关闭浏览器失效）。Token 的业务过期由 Gradio 应用层主动判断，不是靠浏览器自动过期。

### 2.5 OAuth 的三层防护差异

OAuth 与 Basic Auth 最大的区别在于：**OAuth 不是页面级登录墙**，页面始终可访问，只有函数内主动判断身份。

| 防护层级 | Basic Auth | OAuth |
|---------|-----------|-------|
| 第一层 页面级登录墙 | `auth_required: true` → 渲染登录表单，应用不可见 | 无（始终返回完整 config，页面可见） |
| 第二层 接口校验 | `login_check` → 未登录返回 401 | `login_check` 对 OAuth **不生效**（因为 `app.auth is None` 且无 `auth_dependency`） |
| 第三层 登录态注入 | `gr.Request.username` | `gr.OAuthProfile` / `gr.OAuthToken` |

**login_check 对 OAuth 为何不生效**：

```python
# login_check 实现 [routes.py:408-419]
def login_check(user: str = Depends(get_current_user)):
    # OAuth 模式下：app.auth is None，app.auth_dependency is None
    # 因此条件 (app.auth is None and app.auth_dependency is None) 为 True
    # 直接 return，永远不会抛 401
    if (app.auth is None and app.auth_dependency is None) or user is not None:
        return
    raise HTTPException(401, ...)
```

这意味着：
- OAuth 应用中，所有 API（`/run/{api}`、`/config`、`/file=...` 等）**无需登录即可调用**
- 受保护只能在用户函数内通过 `OAuthProfile` / `OAuthToken` 参数类型判断
  - 函数参数为非 Optional（如 `profile: gr.OAuthProfile`）：未登录时自动抛出 `Error("This action requires a logged in user.")`
  - 函数参数为 Optional（如 `profile: Optional[gr.OAuthProfile]`）：未登录时值为 `None`，由业务逻辑自行处理

### 2.6 登录态注入：OAuthProfile / OAuthToken

注入发生在 `special_args()` 函数中，根据用户函数的参数类型注解自动匹配：

**注入流程**：[helpers.py:970-1028](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/helpers.py#L970-L1028)

```python
# 1. 从 gr.Request 获取 session（兼容两种调用方式）
session = (
    getattr(request, "session", {})                     # HTTP 调用：gr.Request.request 是 fastapi.Request
    or getattr(getattr(request, "request", None), "session", {})  # WebSocket/队列调用
)

# 2. 校验 token 是否过期
oauth_info = oauth._get_valid_oauth_info_from_session(session)

# 3. 根据类型注解注入 OAuthProfile
if type_hint in (Optional[oauth.OAuthProfile], oauth.OAuthProfile):
    oauth_profile = oauth_info["userinfo"] if oauth_info is not None else None
    if oauth_profile is not None:
        oauth_profile = oauth.OAuthProfile(oauth_profile)
    elif type_hint == oauth.OAuthProfile:  # 非 Optional → 未登录抛错
        raise Error("This action requires a logged in user. Please sign in and retry.")
    inputs.insert(i, oauth_profile)

# 4. 或注入 OAuthToken
elif type_hint in (Optional[oauth.OAuthToken], oauth.OAuthToken):
    oauth_token = (
        oauth.OAuthToken(
            token=oauth_info["access_token"],
            scope=oauth_info["scope"],
            expires_at=oauth_info["expires_at"],
        ) if oauth_info is not None else None
    )
    if oauth_token is None and type_hint == oauth.OAuthToken:  # 非 Optional → 未登录抛错
        raise Error("This action requires a logged in user. Please sign in and retry.")
    inputs.insert(i, oauth_token)
```

**OAuthProfile 数据类**：[oauth.py:258-299](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L258-L299)

```python
@dataclass
class OAuthProfile(typing.Dict):  # 继承 Dict 保持向后兼容（旧代码可当 dict 用）
    name: str           # userinfo.name → 用户全名
    username: str         # userinfo.preferred_username → HF 用户名
    profile: str        # userinfo.profile → 个人主页 URL
    picture: str       # userinfo.picture → 头像 URL
```

**OAuthToken 数据类**：[oauth.py:302-336](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L302-L336)

```python
@dataclass
class OAuthToken:
    token: str          # access_token → 可用于调用 HF API
    scope: str         # 授权范围
    expires_at: int    # 过期时间戳（seconds since epoch）
```

**LoginButton 页面加载时的状态检查**：[login_button.py:100-117](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/components/login_button.py#L100-L117)

```python
def _check_login_status(self, request: Request) -> LoginButton:
    session = getattr(request, "session", None) or getattr(request.request, "session", None)
    
    if session is None:
        return LoginButton(self.value, interactive=True)  # 未登录 → "Sign in with Hugging Face"
    
    oauth_info = oauth._get_valid_oauth_info_from_session(session)
    if oauth_info is None:
        return LoginButton(self.value, interactive=True)  # 未登录/过期 → 同上
    
    username = oauth_info["userinfo"]["preferred_username"]
    return LoginButton(self.logout_value.format(username), interactive=True)  # 已登录 → "Logout (abidlabs)"
```

### 2.7 登出流程

**路由**：`GET /logout`

[oauth.py:149-153](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L149-L153)

```python
@app.get("/logout")
async def oauth_logout(request: fastapi.Request) -> RedirectResponse:
    request.session.pop("oauth_info", None)  # 仅删除 session 中的 oauth_info 键
    return _redirect_to_target(request)     # 安全重定向回目标页
```

与 Basic Auth 不同：
- 只能登出当前会话（无法登出所有设备，因为服务端无会话存储）
- 只删除 `oauth_info` 键，`session` Cookie 本身仍然存在但内容为空

---

## 三、两种认证方式对比

| 特性 | Basic Auth | OAuth |
|------|-----------|-------|
| **配置方式** | `launch(auth=...)` | 在 Blocks 中放 `gr.LoginButton()` |
| **登录形式** | 页面级登录墙（未登录看不到应用） | 组件级登录按钮（页面始终可见，函数内判断身份） |
| **会话存储** | 服务端内存字典 `app.tokens: {token: username}` | 客户端签名 Cookie（SessionMiddleware），服务端无状态 |
| **Cookie 名** | `access-token-{cookie_id}` + 非安全降级版 | `session`（starlette 默认） |
| **Cookie 签名密钥** | 无（token 是随机字符串，服务端查表） | `sha256(OAUTH_CLIENT_SECRET + "-v4")` |
| **过期机制** | 无（进程重启所有会话失效） | 双层：Cookie 签名级（默认会话 Cookie）+ Token 业务级（`expires_at`） |
| **第一层（页面）** | 未登录 → `auth_required: true` → 渲染登录表单 | 未登录 → 照常渲染应用，LoginButton 显示"Sign in" |
| **第二层（API）** | `login_check` → 未登录 401 | `login_check` 不生效（`app.auth is None`），所有 API 开放 |
| **第三层（注入）** | `gr.Request.username` | `gr.OAuthProfile` / `gr.OAuthToken` |
| **未登录处理** | 前端拦截 + 后端 401 | 非 Optional 参数自动报错；Optional 参数返回 None |
| **登出范围** | 可登出全部会话（默认）或仅当前会话 | 仅当前会话（服务端无状态） |
| **密码校验** | `hmac.compare_digest` 恒定时间比较 | OAuth 协议处理 |
| **适用场景** | 内部系统、简单鉴权 | Hugging Face Space 应用、HF 生态集成 |

---

## 四、关键代码索引

### 认证配置
- [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py) - `launch()` 方法（L2602-L2800）、`expects_oauth` 属性（L1414-L1419）、`auth` 属性（L1144）
- [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py) - `App.configure_app()`（L265-L280）

### Basic Auth
- **登录回调**：[routes.py:465-507](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L465-L507) - `POST /login`
- **密码比较**：[route_utils.py:864-865](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/route_utils.py#L864-L865) - `compare_passwords_securely`
- **get_current_user**：[routes.py:398-406](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L398-L406)
- **第一层 页面级登录墙**：[routes.py:603-666](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L603-L666) - `main()`
- **第二层 login_check**：[routes.py:408-419](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L408-L419)
- **第三层 gr.Request**：[route_utils.py:141-220](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/route_utils.py#L141-L220) - `Request` 类、`compile_gr_request`
- **登出**：[routes.py:520-544](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L520-L544)
- **前端登录表单**：[Login.svelte](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/core/src/Login.svelte)
- **前端 SSR 登录墙检测**：[+page.server.ts](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/app/src/routes/[...catchall]/+page.server.ts)、[+page.ts](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/app/src/routes/[...catchall]/+page.ts)

### OAuth
- **attach_oauth / SessionMiddleware**：[oauth.py:25-54](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L25-L54)
- **真实 OAuth 路由**：[oauth.py:57-153](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L57-L153)
- **Mock OAuth 路由**：[oauth.py:156-194](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L156-L194)
- **重定向 URI / 安全重定向**：[oauth.py:197-240](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L197-L240)
- **会话有效性检查**：[oauth.py:243-255](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L243-L255) - `_get_valid_oauth_info_from_session`
- **OAuthProfile / OAuthToken**：[oauth.py:258-336](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L258-L336)
- **LoginButton 组件**：[login_button.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/components/login_button.py)
- **登录态注入（special_args）**：[helpers.py:918-1042](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/helpers.py#L918-L1042)
- **call_function 注入触发点**：[blocks.py:1582-1680](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py#L1582-L1680)
- **process_api 调用链**：[blocks.py:2174-2352](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py#L2174-L2352)
- **队列中 username 传递**：[queueing.py:279-397](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/queueing.py#L279-L397)

### 使用示例
- [hello_login/run.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/demo/hello_login/run.py) - Basic Auth 使用示例
