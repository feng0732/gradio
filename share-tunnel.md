# Gradio 公网隧道（Share Tunnel）代码深度分析

## 概述

Gradio 的分享链接功能基于 FRP（Fast Reverse Proxy）实现，通过在本地启动 frpc 子进程，与远程服务器建立隧道，将本地服务暴露到公网。

本文严格区分两类信息：
- **代码事实**：从本仓库 Python 代码中可以直接观察和确认的行为
- **FRP 语义推断**：根据 frpc 命令行参数名称和开源 FRP 项目语义所做的合理推测，**本仓库代码不直接可见**

---

## 核心文件

| 文件 | 作用 |
|------|------|
| [gradio/tunneling.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py) | 隧道核心逻辑：`Tunnel` 类，负责 frpc 下载、子进程启动、stdout 解析 |
| [gradio/networking.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py) | 网络辅助层：`setup_tunnel()` 函数协调服务器发现和隧道创建 |
| [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py) | 应用入口：`launch()` 触发隧道创建、生成 `share_token`、改写最终 URL |

---

## 一、代理标识（share_token）的代码事实

### 1.1 生成

> **代码事实**，见 [blocks.py#L143](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L143-L143)

```python
self.share_token = secrets.token_urlsafe(32)
```

- 生成时机：`Blocks.__init__` 时
- 算法：Python 标准库 `secrets.token_urlsafe(32)`，密码学安全随机数
- 32 字节 → URL-safe Base64 编码后约 43 个字符

### 1.2 传递给 frpc

> **代码事实**，见 [tunneling.py#L128-L129](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L128-L129)

```python
"-n",
self.share_token,
```

`share_token` 作为 frpc 命令的 `-n` 参数值传入。Python 代码**只知道**这是传给 frpc 的一个命令行参数，名为 `-n`。

> **FRP 语义推断**：`-n` 在 FRP 中代表 proxy name（代理名称），服务端用它区分不同隧道的代理注册。但本仓库 Python 代码中没有任何代码读取、验证或依赖 `-n` 的 FRP 语义，只是原样传递。

### 1.3 share_token 与子域名的关系

> **代码事实**：Python 代码中没有任何逻辑将 `share_token` 映射到子域名。两者的生成路径完全独立：`share_token` 在本地生成，子域名地址从 frpc stdout 中读取。

> **FRP 语义推断**：根据 FRP 项目设计，`--sd random` 参数请求服务端随机分配子域名，该子域名与 `-n` 指定的代理名称是独立的两个概念。

---

## 二、frpc 子进程启动的代码事实

### 2.1 完整命令构造

> **代码事实**，见 [tunneling.py#L125-L141](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L125-L141)

```python
command = [
    binary,                                                    # frpc 可执行文件路径
    "http",                                                    # 子命令
    "-n", self.share_token,                                    # 参数 -n
    "-l", str(self.local_port),                                # 参数 -l
    "-i", self.local_host,                                     # 参数 -i
    "--uc",                                                    # 开关 --uc
    "--sd", "random",                                          # 参数 --sd，值为 random
    "--ue",                                                    # 开关 --ue
    "--server_addr", f"{self.remote_host}:{self.remote_port}", # 参数 --server_addr
    "--disable_log_color",                                     # 开关 --disable_log_color
]
```

以上是代码中**可直接确认的全部事实**：构造了一个命令行参数列表并启动子进程。

### 2.2 各参数的代码可见职责

下表严格区分代码事实与 FRP 语义推断：

| 参数 | 代码可见事实 | FRP 语义推断（本仓库不可见） |
|------|-------------|--------------------------|
| `http` | 作为 frpc 的第一个子命令参数传入 | FRP 的代理类型，表示 HTTP 反向代理模式 |
| `-n <token>` | 将 `share_token` 作为 `-n` 的值传入 | FRP 中代表 proxy name（代理名称） |
| `-l <port>` | 将 `local_port` 转字符串后作为 `-l` 的值传入 | FRP 中代表 local port（本地监听端口） |
| `-i <host>` | 将 `local_host` 作为 `-i` 的值传入 | FRP 中代表 local IP（本地绑定地址） |
| `--uc` | 作为开关传入，无值 | FRP 中代表 use custom subdomain（启用自定义子域名） |
| `--sd random` | `--sd` 参数的值固定为 `"random"` | FRP 中代表 subdomain 策略，random 请求随机子域名 |
| `--ue` | 作为开关传入，无值 | FRP 中代表 use encryption（启用加密） |
| `--server_addr <addr>` | 拼接 `remote_host:remote_port` 作为值传入 | FRP 中代表 FRP 服务端地址 |
| `--disable_log_color` | 作为开关传入，无值 | 禁用日志彩色输出（代码中有正则解析 stdout 的需求，这是可推断的原因） |

### 2.3 TLS 相关参数

> **代码事实**，见 [tunneling.py#L142-L149](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L142-L149)

```python
if self.share_server_tls_certificate is not None:
    command.extend([
        "--tls_enable",
        "--tls_trusted_ca_file",
        self.share_server_tls_certificate,
    ])
```

**代码可见行为**：当 `share_server_tls_certificate` 不为 None 时，追加 `--tls_enable` 开关和 `--tls_trusted_ca_file` 参数。Python 代码**不直接观察** TLS 握手过程。

> **FRP 语义推断**：`--tls_enable` 启用 frpc 到 frps 的 TLS 连接；`--tls_trusted_ca_file` 指定受信 CA 证书文件用于验证服务端。

### 2.4 子进程启动与生命周期

> **代码事实**，见 [tunneling.py#L150-L154](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L150-L154)

```python
self.proc = subprocess.Popen(
    command, stdout=subprocess.PIPE, stderr=subprocess.PIPE
)
atexit.register(self.kill)
```

- `stdout` 被管道重定向，用于后续逐行读取
- `stderr` 被管道重定向但代码**从未读取** stderr
- `atexit.register(self.kill)` 确保进程退出时终止 frpc

> **代码事实**，见 [tunneling.py#L124](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L124-L124) 和 [blocks.py#L3369-L3370](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3369-L3370)

```python
CURRENT_TUNNELS.append(self)  # 注册到全局列表
# ...
for tunnel in CURRENT_TUNNELS:
    tunnel.kill()  # KeyboardInterrupt 时逐一终止
```

---

## 三、frpc 输出解析的代码事实

### 3.1 输出读取机制

> **代码事实**，见 [tunneling.py#L156-L193](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L156-L193)

Python 代码**唯一可观察 frpc 内部状态的方式**是读取子进程的 stdout。代码在一个 while 循环中逐行读取，关注两个字符串模式：

```python
if "start proxy success" in line:
    result = re.search("start proxy success: (.+)\n", line)
    if result is None:
        _raise_tunnel_error()
    else:
        url = result.group(1)
elif "login to server failed" in line:
    _raise_tunnel_error()
```

### 3.2 代码可见的 frpc 输出阶段

**这两个匹配模式是 Python 代码对 frpc 进程状态的全部认知**：

| stdout 中的字符串 | 代码的处理行为 | 推断的 frpc 阶段 |
|------------------|--------------|----------------|
| `"login to server failed"` | 立即抛出异常，视为致命错误 | frpc 与服务端之间的登录/认证阶段失败 |
| `"start proxy success: <url>"` | 正则提取 `: ` 后面的内容作为 URL，返回 | frpc 代理建立成功，返回公网访问地址 |

> **关键区分**：
> - `"login to server failed"` 的存在证实 frpc 有一个"登录"阶段，但**代码不可见**登录的具体协议、报文格式和认证机制
> - `"start proxy success"` 的存在证实 frpc 有一个"代理建立"阶段，且成功时返回一个 URL，但**代码不可见**代理注册的具体过程

### 3.3 日志收集与错误处理

> **代码事实**

```python
log = []
# ...
log.append(line.strip())  # 每一行 stdout 都被收集
```

所有 frpc 输出行都被收集到 `log` 列表中。当超时或遇到错误时，完整日志会输出到 stderr：

```python
def _raise_tunnel_error():
    log_text = "\n".join(log)
    print(log_text, file=sys.stderr)
    raise ValueError(f"{TUNNEL_ERROR_MESSAGE}\n{log_text}")
```

### 3.4 超时机制

> **代码事实**，见 [tunneling.py#L54](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L54-L54) 和 [tunneling.py#L169-L170](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L169-L170)

```python
TUNNEL_TIMEOUT_SECONDS = 30
# ...
if time.time() - start_timestamp >= TUNNEL_TIMEOUT_SECONDS:
    _raise_tunnel_error()
```

30 秒内未匹配到 `"start proxy success"` 或 `"login to server failed"` 则超时。

---

## 四、服务端返回地址的代码事实

### 4.1 地址提取

> **代码事实**，见 [tunneling.py#L184-L189](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L184-L189)

```python
if "start proxy success" in line:
    result = re.search("start proxy success: (.+)\n", line)
    url = result.group(1)
```

**代码可见行为**：从 frpc 输出中，取 `"start proxy success: "` 之后到行尾的内容作为 `url`。

### 4.2 返回地址的格式推断

> **以下为基于代码行为逻辑的推断，不是直接可见事实**

Python 代码随后对 `url` 执行 `urlparse()` 和 `urlunparse()`（见 [blocks.py#L3129-L3132](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3129-L3132)）：

```python
parsed_url = urlparse(share_url)
self.share_url = urlunparse(
    (self.share_server_protocol,) + parsed_url[1:]
)
```

如果 frpc 返回的是纯域名（如 `abc123.gradio.live`），`urlparse` 会将其解析为 `path` 而非 `netloc`，导致 `urlunparse` 改写后生成错误的 URL（如 `https:abc123.gradio.live`，缺少 `//`）。

经实际验证：

```
urlparse("http://abc123.gradio.live")
  → scheme='http', netloc='abc123.gradio.live', path=''
  → urlunparse(('https',)+parsed[1:]) = 'https://abc123.gradio.live'  ✓

urlparse("abc123.gradio.live")
  → scheme='', netloc='', path='abc123.gradio.live'
  → urlunparse(('https',)+parsed[1:]) = 'https:abc123.gradio.live'  ✗
```

**推断结论**：frpc 返回的地址**必须**是带 `http://` 前缀的完整 URL（如 `http://abc123.gradio.live`），否则后续改写逻辑无法正确工作。

> **注意**：这一推断的依据是代码中 `urlparse`+`urlunparse` 的组合行为。代码本身没有对 `share_url` 格式做任何显式校验或注释说明。

### 4.3 子域名从何而来

> **代码事实**：Python 代码中**没有任何逻辑**生成或选择子域名。frpc 命令中 `--sd random` 是传给 frpc 的参数，子域名的生成发生在 frpc 进程与服务端的交互中，对 Python 代码来说是**黑箱**。

> **FRP 语义推断**：`--sd random` 请求服务端随机分配子域名，`--uc` 启用自定义子域名功能。但子域名的具体生成算法、分配策略完全在 frpc/frps 的 Go 代码中，本仓库不可见。

---

## 五、本地 URL 改写的代码事实

### 5.1 协议选择逻辑

> **代码事实**，见 [blocks.py#L2966-L2968](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L2966-L2968)

```python
self.share_server_protocol = share_server_protocol or (
    "http" if share_server_address is not None else "https"
)
```

这是纯粹的 Python 三元逻辑：

| 条件 | `share_server_protocol` 值 |
|------|--------------------------|
| 用户显式传了 `share_server_protocol` 参数 | 使用用户指定值 |
| `share_server_address is not None`（即使用了自定义服务器） | `"http"` |
| `share_server_address is None`（即使用官方服务器） | `"https"` |

### 5.2 URL 改写过程

> **代码事实**，见 [blocks.py#L3129-L3132](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3129-L3132)

```python
parsed_url = urlparse(share_url)
self.share_url = urlunparse(
    (self.share_server_protocol,) + parsed_url[1:]
)
```

**代码可见行为**：
1. 将 `setup_tunnel()` 返回的字符串用 `urlparse` 分解为 6 元组 `(scheme, netloc, path, params, query, fragment)`
2. 保留后 5 个部分不变，仅将 `scheme` 替换为 `share_server_protocol`
3. 用 `urlunparse` 重新组合

**效果**：将 frpc 返回的 URL 的协议部分替换为 `share_server_protocol`，其他部分原样保留。

### 5.3 官方服务器场景的完整示例

> **基于代码逻辑的推演**

```
setup_tunnel() 返回:  "http://abc123.gradio.live"
                     ↓ urlparse
scheme='http', netloc='abc123.gradio.live', path='', params='', query='', fragment=''
                     ↓ urlunparse 以 'https' 替换 scheme
最终 share_url:      "https://abc123.gradio.live"
```

### 5.4 自定义服务器场景的完整示例

```
setup_tunnel() 返回:  "http://xyz.local:8080"
                     ↓ urlparse
scheme='http', netloc='xyz.local:8080', path='', params='', query='', fragment=''
                     ↓ urlunparse 以 'http' 替换 scheme（share_server_address != None → 默认 http）
最终 share_url:      "http://xyz.local:8080"
```

### 5.5 为什么需要改写

> **FRP 语义推断**：frpc 返回的地址始终以 `http://` 开头，因为 frp 内部只处理 HTTP 流量。官方服务器在网关层有 TLS 终结，对外应使用 `https`，因此需要在 Python 层改写协议。自定义服务器可能没有 TLS，所以默认保持 `http`。

---

## 六、服务器发现的代码事实

### 6.1 默认路径：请求 Gradio API

> **代码事实**，见 [networking.py#L35-L53](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py#L35-L53)

```python
if share_server_address is None:
    response = httpx.get(GRADIO_API_SERVER, timeout=30)
    payload = response.json()[0]
    remote_host, remote_port = payload["host"], int(payload["port"])
    certificate = payload["root_ca"]
    # ...
    with open(CERTIFICATE_PATH, "w") as f:
        f.write(certificate)
    share_server_tls_certificate = CERTIFICATE_PATH
```

**代码可见行为**：
1. 向 `https://api.gradio.app/v3/tunnel-request` 发 GET 请求
2. 从 JSON 响应的 `[0]` 中取 `host`、`port`、`root_ca` 三个字段
3. 将 `root_ca` 写入本地 `.gradio/certificate.pem`
4. 将证书路径赋给 `share_server_tls_certificate`

### 6.2 自定义路径：用户指定服务器

> **代码事实**，见 [networking.py#L55-L57](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py#L55-L57)

```python
else:
    remote_host, remote_port = share_server_address.split(":")
    remote_port = int(remote_port)
```

**代码可见行为**：按 `:` 分割用户提供的地址字符串为 host 和 port。此时**不**请求 API、**不**获取证书、`share_server_tls_certificate` 保持为用户传入的原始值（通常为 None）。

---

## 七、frpc 二进制下载的代码事实

### 7.1 下载与校验

> **代码事实**，见 [tunneling.py#L83-L110](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L83-L110)

- 下载地址根据 `platform.system()` 和 `platform.machine()` 动态拼接
- 下载源：`https://cdn-media.huggingface.co/frpc-gradio-0.3/frpc_{os}_{arch}[.exe]`
- 存储位置：`{HF_HOME}/gradio/frpc/frpc_{os}_{arch}_v0.3`
- 已存在的文件**不重新下载**（`if not Path(BINARY_PATH).exists()`）
- 下载后做 SHA-256 校验，与硬编码的 `CHECKSUMS` 字典比对
- 校验失败抛出 `ChecksumMismatchError`

---

## 八、三者关系的数据流总结

```
┌──────────────────────────────────────────────────────────────────────┐
│                       代码事实（本仓库可见）                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  share_token                                                         │
│  生成: secrets.token_urlsafe(32)  [blocks.py:143]                   │
│  传递: frpc -n <share_token>      [tunneling.py:128-129]            │
│                                                                      │
│         ↓ frpc 子进程内部（黑箱，代码不可见）                        │
│                                                                      │
│  frpc stdout 输出                                                    │
│  可见模式1: "login to server failed"   [tunneling.py:190]           │
│  可见模式2: "start proxy success: http://xxx.gradio.live"            │
│             [tunneling.py:184-189]                                   │
│                                                                      │
│         ↓ Python 代码处理                                            │
│                                                                      │
│  地址提取: 正则 "start proxy success: (.+)\n" → url                 │
│            [tunneling.py:185-189]                                    │
│                                                                      │
│  协议改写: urlparse + urlunparse                                    │
│            scheme 替换为 share_server_protocol                       │
│            [blocks.py:3129-3132]                                    │
│                                                                      │
│  最终 URL: https://xxx.gradio.live（官方服务器）                     │
│           http://xxx:port（自定义服务器）                             │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│                       FRP 语义推断（本仓库不可见）                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  frpc 内部行为:                                                      │
│  - TCP 连接到 server_addr                                            │
│  - TLS 握手（如果 --tls_enable）                                     │
│  - 登录认证（"login to server failed" 证实此阶段存在）               │
│  - 代理注册（-n 指定名称，--uc/--sd random 请求子域名）              │
│  - 服务端分配子域名并返回完整 URL                                    │
│                                                                      │
│  上述步骤的具体协议和报文格式在 Go 代码中，Python 代码不可见         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 九、完整流程时序图（区分代码事实与推断）

```
用户/Python代码                          frpc子进程                   外部服务
    │                                       │                           │
    │ Blocks.__init__()                     │                           │
    │ share_token = secrets.token_urlsafe() │                           │
    │───────────────────────────            │                           │
    │                                       │                           │
    │ launch(share=True)                    │                           │
    │ 确定 share_server_protocol            │                           │
    │───────────────────────────            │                           │
    │                                       │                           │
    │ networking.setup_tunnel()             │                           │
    │───────────────────────────            │                           │
    │ httpx.get(api.gradio.app)  ──────────────────────────────────>   │
    │ <─── {host, port, root_ca}  ──────────────────────────────────   │
    │ 写入证书文件                          │                           │
    │───────────────────────────            │                           │
    │ Tunnel.start_tunnel()                 │                           │
    │ download_binary()                     │                           │
    │ httpx.get(cdn-media.huggingface) ────────────────────────────>   │
    │ SHA256 校验                           │                           │
    │───────────────────────────            │                           │
    │ subprocess.Popen([frpc, ...])  ──────>│                           │
    │                                       │                           │
    │  ╔════════════════════════════════════╗                           │
    │  ║ 以下为 frpc 黑箱行为（代码不可见）  ║                           │
    │  ║                                  ║                           │
    │  ║  frpc ──TCP/TLS──> FRP服务器     ║                           │
    │  ║  frpc ──登录────> FRP服务器      ║                           │
    │  ║  frpc ──注册代理─> FRP服务器      ║                           │
    │  ║  frpc <──分配地址── FRP服务器     ║                           │
    │  ╚════════════════════════════════════╝                           │
    │                                       │                           │
    │  ← stdout: "login to server failed"  │  (若登录失败)             │
    │    → 抛出异常                         │                           │
    │                                       │                           │
    │  ← stdout: "start proxy success:      │                           │
    │             http://xxx.gradio.live"   │                           │
    │    → 正则提取 url                     │                           │
    │───────────────────────────            │                           │
    │ urlparse + urlunparse 改写协议        │                           │
    │ 最终 share_url 生成                   │                           │
    │───────────────────────────            │                           │
```

图中实线框为代码可见行为，虚线框 `║` 内为 frpc 黑箱行为。

---

## 十、关键边界总结

| 行为 | 代码是否可见 | 证据来源 |
|------|-------------|---------|
| `share_token` 生成算法 | ✅ 可见 | [blocks.py#L143](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L143-L143) |
| `share_token` 传给 frpc 的 `-n` 参数 | ✅ 可见 | [tunneling.py#L128-L129](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L128-L129) |
| frpc 有"登录"阶段 | ✅ 可推断 | stdout 中检查 `"login to server failed"` [tunneling.py#L190](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L190-L190) |
| 登录的具体协议和认证方式 | ❌ 不可见 | frpc 内部 Go 代码 |
| frpc 有"代理建立"阶段 | ✅ 可推断 | stdout 中检查 `"start proxy success"` [tunneling.py#L184](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L184-L184) |
| 代理注册的具体过程 | ❌ 不可见 | frpc 内部 Go 代码 |
| `-n` 在 FRP 中的语义为 proxy name | ❌ 不可见 | FRP 开源项目约定 |
| `--uc` 启用自定义子域名 | ❌ 不可见 | FRP 开源项目约定 |
| `--sd random` 请求随机子域名 | ❌ 不可见 | FRP 开源项目约定 |
| `--ue` 启用加密 | ❌ 不可见 | FRP 开源项目约定 |
| 服务端返回地址格式为 `http://...` | ⚠️ 间接推断 | `urlparse`+`urlunparse` 改写逻辑要求此格式 |
| 子域名由服务端生成 | ⚠️ 间接推断 | Python 代码不生成子域名，只能从 frpc 输出获取 |
| URL 协议改写逻辑 | ✅ 可见 | [blocks.py#L3129-L3132](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3129-L3132) |
| `share_server_protocol` 默认值逻辑 | ✅ 可见 | [blocks.py#L2966-L2968](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L2966-L2968) |
| API 服务器返回 host/port/root_ca | ✅ 可见 | [networking.py#L37-L40](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py#L37-L40) |

图例：✅ 直接可见 | ⚠️ 间接推断（基于代码逻辑的必要前提） | ❌ 完全不可见
