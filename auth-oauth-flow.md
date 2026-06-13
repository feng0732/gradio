# Auth & OAuth 访问控制链路

本文档梳理 Gradio 框架中 Auth（用户名密码登录）和 OAuth（Hugging Face 登录）的完整访问控制链路，包括登录回调、Cookie 会话管理和受限页面判断逻辑。

---

## 目录

- [整体架构](#整体架构)
- [一、Basic Auth 流程](#一basic-auth-流程用户名密码登录)
  - [1.1 启动配置](#11-启动配置)
  - [1.2 登录回调](#12-登录回调)
  - [1.3 Cookie 与会话](#13-cookie-与会话)
  - [1.4 受限页面判断](#14-受限页面判断)
  - [1.5 登出流程](#15-登出流程)
- [二、OAuth 流程](#二oauth-流程hugging-face-登录)
  - [2.1 启动配置](#21-启动配置)
  - [2.2 登录重定向](#22-登录重定向)
  - [2.3 OAuth 回调](#23-oauth-回调)
  - [2.4 Cookie 与会话](#24-cookie-与会话)
  - [2.5 受限页面与用户信息注入](#25-受限页面与用户信息注入)
  - [2.6 登出流程](#26-登出流程)
- [三、两种认证方式对比](#三两种认证方式对比)
- [四、关键代码索引](#四关键代码索引)

---

## 整体架构

Gradio 支持两种独立的认证机制，**互斥使用**（不能同时启用）：

| 维度 | Basic Auth | OAuth |
|------|-----------|-------|
| 触发方式 | `launch(auth=...) | gr.LoginButton 组件 |
| 登录形式 | 页面级登录墙 | 组件级登录按钮 |
| 会话存储 | 服务端 tokens 字典 + Cookie | SessionMiddleware Cookie (starlette) |
| Cookie 名 | `access-token-{cookie_id}` | `session` (starlette 管理 |
| 用户标识 | username 字符串 | OAuthProfile / OAuthToken 对象 |
| 适用场景 | 简单内部应用 | Hugging Face Space 应用 |

两种方式都通过 FastAPI 的 `Depends(login_check)` 或 `Depends(get_current_user)` 依赖注入实现路由保护。

---

## 一、Basic Auth 流程（用户名/密码登录）

### 1.1 启动配置

**入口**：[blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py) 的 `launch()` 方法

```python
# blocks.py launch() 方法
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
    
    # 标准化 auth 格式：统一为 list[tuple] 或 callable
    if auth and not callable(auth) and not isinstance(auth[0], tuple) and not isinstance(auth[0], list):
        self.auth = [auth]  # 单个 tuple 转为 list
    else:
        self.auth = auth
```

`auth` 参数支持三种形式：
1. `tuple[str, str]` - 单组用户名密码
2. `list[tuple[str, str]]` - 多组用户名密码
3. `Callable[[str, str], bool]` - 自定义验证函数（支持协程）

**App 配置**：[routes.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py) `App.configure_app()`

```python
# routes.py App.configure_app()
def configure_app(self, blocks: gradio.Blocks) -> None:
    auth = blocks.auth
    if auth is not None:
        if not callable(auth):
            self.auth = {account[0]: account[1] for account in auth}  # 转为 dict
        else:
            self.auth = auth
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
    
    # 验证逻辑
    if (
        not callable(app.auth)
        and username in app.auth
        and compare_passwords_securely(password, app.auth[username])
    ) or (
        callable(app.auth)
        and (
            await app.auth(username, password)  # 支持 async
            if inspect.iscoroutinefunction(app.auth)
            else app.auth(username, password)
        )
    ):
        # 登录成功：生成 token
        token = secrets.token_urlsafe(16)
        app.tokens[token] = username  # 服务端存储 token -> username 映射
        
        response = JSONResponse(content={"success": True})
        # 设置安全 Cookie
        response.set_cookie(
            key=f"access-token-{app.cookie_id}",
            value=token,
            httponly=True,
            samesite="none",
            secure=True,
        )
        # 同时设置非安全 Cookie（降级方案）
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

使用 `hmac.compare_digest` 进行恒定时间比较，防止时序攻击：

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
        incorrect_credentials = true;  // 显示错误
    } else if (response.status == 200) {
        location.reload();  // 刷新页面，进入应用
    }
};
```

### 1.3 Cookie 与会话

**Cookie 结构**：

| Cookie 名称 | 特性 | 用途 |
|------------|------|------|
| `access-token-{cookie_id}` | httponly, samesite=none, secure | HTTPS 环境下的主会话 Cookie |
| `access-token-unsecure-{cookie_id}` | httponly | 非 HTTPS 环境下降级使用 |

- `cookie_id` 是 App 启动时生成的随机值：`secrets.token_urlsafe(32)`
- 服务端维护 `app.tokens` 字典：`{token: username}`
- **无过期时间**（服务端内存存储，进程重启失效）

**获取当前用户**：[routes.py:398-406](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L398-L406)

```python
@router.get("/user")
def get_current_user(request: fastapi.Request) -> str | None:
    if app.auth_dependency is not None:
        return app.auth_dependency(request)
    token = request.cookies.get(
        f"access-token-{app.cookie_id}"
    ) or request.cookies.get(f"access-token-unsecure-{app.cookie_id}")
    return app.tokens.get(token)
```

优先级：`auth_dependency` > 安全 Cookie > 非安全 Cookie

### 1.4 受限页面判断

**登录检查依赖**：[routes.py:408-419](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L408-L419)

```python
@router.get("/login_check")
def login_check(user: str = Depends(get_current_user)):
    if (app.auth is None and app.auth_dependency is None) or user is not None:
        return  # 已登录或无需认证
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail={
            "error": "Not authenticated",
            "auth_message": blocks.auth_message,
        },
    )
```

**主页面判断逻辑**：[routes.py:603-666](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L603-L666)

```python
@app.get("/")
def main(request: fastapi.Request, user: str = Depends(get_current_user), ...):
    if (app.auth is None and app.auth_dependency is None) or user is not None:
        # 情况1：无需认证 或 已登录 -> 返回完整配置
        config = utils.safe_deepcopy(blocks.config)
        config["username"] = user
        # ... 完整页面配置
    elif app.auth_dependency:
        # 情况2：使用 auth_dependency 但未认证 -> 401
        raise HTTPException(status_code=401, ...)
    else:
        # 情况3：使用 basic auth 但未登录 -> 返回登录页配置
        config = {
            "auth_required": True,
            "auth_message": blocks.auth_message,
            # ... 最小化配置，前端渲染登录表单
        }
```

**需要 `login_check` 保护的 API 路由：
- `/config` - 配置接口
- `/info` - API 信息
- `/openapi.json` - OpenAPI 文档
- `/dev/reload` - 热重载
- `/proxy=...` - 反向代理
- `/file=...` - 文件服务
- `/file/{path}` - 旧版文件接口

### 1.5 登出流程

**路由**：`GET /logout`

位于 [routes.py:520-544](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py#L520-L544)

```python
@app.get("/logout")
def logout(request: fastapi.Request, user: str = Depends(get_current_user), all_session: bool = True):
    response = RedirectResponse(url=root, status_code=302)
    # 删除 Cookie
    response.delete_cookie(key=f"access-token-{app.cookie_id}", path="/")
    response.delete_cookie(key=f"access-token-unsecure-{app.cookie_id}", path="/")
    
    if all_session:
        # 删除该用户所有会话
        for token in list(app.tokens.keys()):
            if app.tokens[token] == user:
                del app.tokens[token]
    elif request.cookies.get(f"access-token-{app.cookie_id}") in app.tokens:
        # 仅删除当前会话
        del app.tokens[request.cookies.get(f"access-token-{app.cookie_id}")]
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
    attach_oauth(app)  # 挂载 OAuth 路由 + SessionMiddleware
else:
    # 挂载普通 logout 路由（basic auth 用）
    @app.get("/logout")
    def logout(...): ...
```

**SessionMiddleware 配置**：[oauth.py:25-54](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L25-L54)

```python
def attach_oauth(app: fastapi.FastAPI):
    # 根据是否在 Space 环境决定使用真实 OAuth 还是 Mock OAuth
    if get_space() is not None:
        _add_oauth_routes(app)       # 真实 OAuth
    else:
        _add_mocked_oauth_routes(app)  # 本地 Mock
    
    # Session Middleware - 用于存储 OAuth 会话
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
- `OAUTH_CLIENT_SECRET` - OAuth 客户端密钥
- `OAUTH_SCOPES` - OAuth 权限范围
- `OPENID_PROVIDER_URL` - OpenID Provider URL（通常为 `https://huggingface.co`）

### 2.2 登录重定向

**登录按钮**：[login_button.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/components/login_button.py)

`LoginButton` 组件在 `activate()` 方法中：
1. 绑定 `click` 事件的 JS 处理函数
2. 绑定页面加载事件 `_check_login_status` 检查登录状态

**前端重定向逻辑**：

```javascript
// login_button.py 中的 _js_handle_redirect
(buttonValue) => {
    uri = buttonValue === BUTTON_DEFAULT_VALUE 
        ? '/login/huggingface?_target_url=/REDIRECT_URL' 
        : '/logout?_target_url=/REDIRECT_URL';
    window.location.assign(uri + window.location.search);
}
```

**登录重定向路由**：`GET /login/huggingface`

位于 [oauth.py:94-99](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L94-L99)

```python
@app.get("/login/huggingface")
async def oauth_login(request: fastapi.Request):
    redirect_uri = _generate_redirect_uri(request)
    return await oauth.huggingface.authorize_redirect(request, redirect_uri)
```

**重定向 URI 生成**：[oauth.py:197-221](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L197-L221)

```python
def _generate_redirect_uri(request: fastapi.Request) -> str:
    # 确定登录后跳转目标
    if "_target_url" in request.query_params:
        target = request.query_params["_target_url"]
    else:
        target = "/?" + urllib.parse.urlencode(request.query_params)
    
    # Space 环境：使用 SPACE_HOST 构建回调 URL（确保回调到 hf.space 域名
    if space_host := os.getenv("SPACE_HOST"):
        space_host = space_host.split(",")[0]  # 自定义域名时取第一个
        redirect_uri = f"https://{space_host}/login/callback?_target_url={target}"
        return redirect_uri
    
    # 本地环境：使用 request.url_for
    redirect_uri = request.url_for("oauth_redirect_callback").include_query_params(
        _target_url=target
    )
    return str(redirect_uri)
```

### 2.3 OAuth 回调

**回调路由**：`GET /login/callback`

位于 [oauth.py:101-147](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L101-L147)

```python
@app.get("/login/callback")
async def oauth_redirect_callback(request: fastapi.Request) -> RedirectResponse:
    try:
        # 用授权码换取 access token
        oauth_info = await oauth.huggingface.authorize_access_token(request)
    except MismatchingStateError:
        # State 不匹配：通常是 Cookie 损坏
        # 清理相关 session 状态
        for key in list(request.session.keys()):
            if key.startswith("_state_huggingface"):
                request.session.pop(key)
        
        nb_redirects = int(request.query_params.get("_nb_redirects", 0))
        target_url = request.query_params.get("_target_url")
        
        query_params = {"_nb_redirects": nb_redirects + 1}
        if target_url:
            query_params["_target_url"] = target_url
        
        login_uri = f"/login/huggingface?{urllib.parse.urlencode(query_params)}"
        
        # 超过最大重定向次数：可能是第三方 Cookie 被阻止
        if nb_redirects > MAX_REDIRECTS:  # MAX_REDIRECTS = 2
            host = os.environ.get("SPACE_HOST")
            host_url = "https://" + host.rstrip("/")
            return RedirectResponse(host_url + login_uri)  # 跳转到非 iframe 视图
        
        return RedirectResponse(login_uri)  # 重试登录
    
    # 登录成功：存储用户信息到 session
    request.session["oauth_info"] = oauth_info
    return _redirect_to_target(request)
```

**安全重定向（防止开放重定向漏洞）：[oauth.py:224-240](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L224-L240)

```python
def _redirect_to_target(request: fastapi.Request, default_target: str = "/") -> RedirectResponse:
    target = request.query_params.get("_target_url", default_target)
    parsed = urllib.parse.urlparse(target)
    # 只保留路径部分，剥离 scheme/host，防止开放重定向
    safe_target = "/" + (parsed.path or "").lstrip("/\\")
    if parsed.query:
        safe_target += "?" + parsed.query
    if parsed.fragment:
        safe_target += "#" + parsed.fragment
    return RedirectResponse(safe_target)
```

**本地 Mock OAuth**：[oauth.py:156-194](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L156-L194)

本地开发时，使用本地登录的 HF 账号模拟 OAuth：
- `/login/huggingface` 直接重定向到 `/login/callback`
- `/login/callback` 将 Mock 用户信息写入 session
- 使用 `_get_mocked_oauth_info()` 从 `huggingface_hub` 获取当前登录用户

### 2.4 Cookie 与会话

**SessionMiddleware**：

由 `starlette.middleware.sessions.SessionMiddleware` 提供

| 特性 | 说明 |
|------|------|
| Cookie 名 | `session`（starlette 默认） |
| 签名密钥 | `sha256(OAUTH_CLIENT_SECRET + "-v4")` |
| SameSite | `none` |
| Secure | `true`（HTTPS only） |
| 存储位置 | 客户端 Cookie（签名加密 |

**会话有效性检查**：[oauth.py:243-255](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L243-L255)

```python
def _get_valid_oauth_info_from_session(session):
    oauth_info = session.get("oauth_info")
    if oauth_info is None:
        return None
    
    expires_at = oauth_info.get("expires_at")
    if expires_at is not None and expires_at < time.time():
        session.pop("oauth_info", None)  # 过期则清除
        return None
    
    return oauth_info
```

`oauth_info 结构：
```python
{
    "access_token": "...",
    "token_type": "bearer",
    "expires_in": 3600,
    "id_token": "...",
    "scope": "openid profile",
    "expires_at": 1691676444,  # 过期时间戳
    "userinfo": {
        "sub": "...",
        "name": "User Name",
        "preferred_username": "username",
        "profile": "https://huggingface.co/username",
        "picture": "...",
        # ... 其他 OpenID 字段
    }
}
```

### 2.5 受限页面与用户信息注入

**OAuth 与 Basic Auth 的重要区别**：
- OAuth 不是页面级的登录墙
- 而是组件级的登录按钮
- 页面本身始终可访问，但函数可以通过类型注解获取用户信息

**用户信息注入**：[helpers.py:970-1028](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/helpers.py#L970-L1028)

在 `special_args` 函数中，根据函数参数类型自动注入：

```python
# 从 request 中获取 session
session = (
    getattr(request, "session", {})
    or getattr(getattr(request, "request", None), "session", {})
)

oauth_info = oauth._get_valid_oauth_info_from_session(session)

# 注入 OAuthProfile
if type_hint in (Optional[oauth.OAuthProfile], oauth.OAuthProfile):
    oauth_profile = oauth_info["userinfo"] if oauth_info is not None else None
    if oauth_profile is not None:
        oauth_profile = oauth.OAuthProfile(oauth_profile)
    elif type_hint == oauth.OAuthProfile:
        raise Error("This action requires a logged in user...")
    inputs.insert(i, oauth_profile)

# 注入 OAuthToken
elif type_hint in (Optional[oauth.OAuthToken], oauth.OAuthToken):
    oauth_token = oauth.OAuthToken(
        token=oauth_info["access_token"],
        scope=oauth_info["scope"],
        expires_at=oauth_info["expires_at"],
    ) if oauth_info is not None else None
    if oauth_token is None and type_hint == oauth.OAuthToken:
        raise Error("This action requires a logged in user...")
    inputs.insert(i, oauth_token)
```

**OAuthProfile 类**：[oauth.py:258-299](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L258-L299)

```python
@dataclass
class OAuthProfile(typing.Dict):  # 继承 Dict 用于向后兼容
    name: str           # 用户全名
    username: str         # 用户名
    profile: str        # 个人主页 URL
    picture: str       # 头像 URL
```

**OAuthToken 类**：[oauth.py:302-336](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L302-L336)

```python
@dataclass
class OAuthToken:
    token: str          # access token
    scope: str         # 权限范围
    expires_at: int    # 过期时间戳
```

**LoginButton 状态检查**：[login_button.py:100-117](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/components/login_button.py#L100-L117)

页面加载时自动检查登录状态：

```python
def _check_login_status(self, request: Request) -> LoginButton:
    session = getattr(request, "session", None) or getattr(request.request, "session", None)
    
    if session is None:
        return LoginButton(self.value, interactive=True)  # 未登录
    
    oauth_info = oauth._get_valid_oauth_info_from_session(session)
    if oauth_info is None:
        return LoginButton(self.value, interactive=True)  # 未登录
    
    # 已登录：显示登出按钮
    username = oauth_info["userinfo"]["preferred_username"]
    return LoginButton(self.logout_value.format(username), interactive=True)
```

### 2.6 登出流程

**路由**：`GET /logout`

位于 [oauth.py:149-153](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py#L149-L153)

```python
@app.get("/logout")
async def oauth_logout(request: fastapi.Request) -> RedirectResponse:
    request.session.pop("oauth_info", None)  # 清除 session 中的用户信息
    return _redirect_to_target(request)   # 重定向回目标页
```

---

## 三、两种认证方式对比

| 特性 | Basic Auth | OAuth |
|------|-----------|-------|
| 配置方式 | `launch(auth=...)` | `gr.LoginButton()` 组件 |
| 登录形式 | 页面级登录墙 | 组件级登录按钮 |
| 会话存储 | 服务端内存 `tokens` 字典 | 客户端签名 Cookie（SessionMiddleware） |
| Cookie 名 | `access-token-{cookie_id}` | `session` |
| 用户标识 | username 字符串 | OAuthProfile / OAuthToken 对象 |
| 页面访问控制 | 未登录显示登录页 | 始终可访问，函数内判断 |
| 过期机制 | 无（进程重启失效 | 有（expires_at） |
| 适用场景 | 内部系统、简单鉴权 | Hugging Face Space 应用 |
| 登出范围 | 可登出所有会话 | 仅当前会话 |
| 密码比较 | hmac.compare_digest | OAuth 协议处理 |
| 与 gr.Request | `request.username` | `request.session` |

---

## 四、关键代码索引

### 核心文件：

- **认证配置**
  - [blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/blocks.py) - `launch()` 方法、`expects_oauth` 属性、`auth` 属性

- **Basic Auth**
  - [routes.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/routes.py) - `/login`、`/logout`、`/login_check`、`get_current_user`、`main` 页面
  - [route_utils.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/route_utils.py) - `compare_passwords_securely`
  - [Login.svelte](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/js/core/src/Login.svelte) - 前端登录表单

- **OAuth**
  - [oauth.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/oauth.py) - `attach_oauth`、`_add_oauth_routes`、`_add_mocked_oauth_routes`、`_get_valid_oauth_info_from_session`、`OAuthProfile`、`OAuthToken`
  - [login_button.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/components/login_button.py) - `LoginButton` 组件
  - [helpers.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/gradio/helpers.py) - `special_args` 中的用户信息注入

- **示例
  - [hello_login/run.py](file:///d:/fz/0601/solo-dogfeeding/code/254-gradio/demo/hello_login/run.py) - Basic Auth 使用示例
