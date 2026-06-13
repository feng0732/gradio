# Gradio 公网隧道（Share Tunnel）代码分析

## 概述

Gradio 的分享链接功能基于 **FRP（Fast Reverse Proxy）** 实现，通过在本地启动 FRP 客户端（frpc），与远程 FRP 服务器建立隧道连接，从而将本地服务暴露到公网。整个流程涉及三个核心阶段：**隧道启动**、**握手连接**、**地址生成**。

## 核心文件

| 文件 | 作用 |
|------|------|
| [gradio/tunneling.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py) | 隧道核心逻辑，封装了 `Tunnel` 类，负责 frpc 下载、启动、地址读取 |
| [gradio/networking.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py) | 网络辅助层，`setup_tunnel()` 函数协调隧道建立 |
| [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py) | 应用入口，`launch()` 方法触发隧道创建并生成 `share_token` |

---

## 一、隧道启动流程

### 1.1 启动入口：`Blocks.launch()`

当用户调用 `demo.launch(share=True)` 时，在 [blocks.py#L3119-L3132](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3119-L3132) 中触发隧道建立：

```python
if self.share:
    share_url = networking.setup_tunnel(
        local_host=self.server_name,
        local_port=self.server_port,
        share_token=self.share_token,
        share_server_address=self.share_server_address,
        share_server_tls_certificate=self.share_server_tls_certificate,
    )
```

### 1.2 Share Token 生成

`share_token` 是隧道的唯一标识，在 Blocks 初始化时生成，见 [blocks.py#L143](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L143-L143)：

```python
self.share_token = secrets.token_urlsafe(32)
```

- 使用 Python 标准库 `secrets.token_urlsafe(32)` 生成
- 32 字节随机数，经 URL-safe Base64 编码后约 43 个字符
- 作为 FRP 代理名称，用于服务端识别不同的隧道连接

### 1.3 隧道配置获取：`networking.setup_tunnel()`

[networking.py#L23-L67](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py#L23-L67) 是隧道建立的协调函数，流程如下：

**步骤 1：确定远程服务器地址**

- 如果用户未指定自定义 `share_server_address`，则向 Gradio API 服务器请求可用的 FRP 节点：

```python
response = httpx.get(GRADIO_API_SERVER, timeout=30)
payload = response.json()[0]
remote_host, remote_port = payload["host"], int(payload["port"])
certificate = payload["root_ca"]
```

- API 地址：`https://api.gradio.app/v3/tunnel-request`
- 返回内容包含：服务器主机、端口、根 CA 证书

**步骤 2：写入 TLS 证书**

将服务端返回的根 CA 证书写入本地文件，用于后续 TLS 连接验证：

```python
Path(CERTIFICATE_PATH).parent.mkdir(parents=True, exist_ok=True)
with open(CERTIFICATE_PATH, "w") as f:
    f.write(certificate)
```

**步骤 3：创建并启动 Tunnel 对象**

```python
tunnel = Tunnel(
    remote_host, remote_port, local_host, local_port,
    share_token, share_server_tls_certificate
)
address = tunnel.start_tunnel()
```

### 1.4 frpc 二进制下载

`tunneling.py` 中的 `Tunnel.download_binary()` 静态方法负责下载 FRP 客户端：

**下载地址构造**（[tunneling.py#L26-L28](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L26-L28)）：

```python
BINARY_REMOTE_NAME = f"frpc_{platform.system().lower()}_{machine.lower()}"
BINARY_URL = f"https://cdn-media.huggingface.co/frpc-gradio-{VERSION}/{BINARY_REMOTE_NAME}{EXTENSION}"
```

- 根据操作系统和 CPU 架构选择对应二进制
- 版本号：`0.3`
- 存储位置：`{HF_HOME}/gradio/frpc/frpc_{系统}_{架构}_v{版本}`

**完整性校验**（[tunneling.py#L102-L110](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L102-L110)）：

下载完成后，使用 SHA-256 校验和验证文件完整性，防止篡改：

```python
if BINARY_URL in CHECKSUMS:
    sha = hashlib.sha256()
    with open(BINARY_PATH, "rb") as f:
        for chunk in iter(lambda: f.read(CHUNK_SIZE * sha.block_size), b""):
            sha.update(chunk)
    if calculated_hash != CHECKSUMS[BINARY_URL]:
        raise ChecksumMismatchError()
```

---

## 二、握手机制

### 2.1 FRP 客户端启动参数

隧道启动的核心在 `Tunnel._start_tunnel()` 方法（[tunneling.py#L123-L154](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L123-L154)），通过子进程启动 frpc：

```python
command = [
    binary,
    "http",                  # HTTP 代理模式
    "-n", self.share_token,  # 代理名称（即 share_token）
    "-l", str(self.local_port),   # 本地端口
    "-i", self.local_host,        # 本地主机
    "--uc",                  # 使用自定义子域名
    "--sd", "random",        # 子域名策略：随机
    "--ue",                  # 使用加密
    "--server_addr", f"{self.remote_host}:{self.remote_port}",  # 服务器地址
    "--disable_log_color",   # 禁用日志颜色（便于解析）
]
```

### 2.2 参数详解

| 参数 | 含义 | 作用 |
|------|------|------|
| `http` | 代理模式 | 建立 HTTP 类型的反向代理 |
| `-n` | 代理名称 | 使用 `share_token` 作为唯一标识，服务端据此区分不同隧道 |
| `-l` / `-i` | 本地地址 | 指定要暴露的本地服务地址和端口 |
| `--uc` | 自定义子域名 | 启用子域名配置 |
| `--sd random` | 子域名策略 | 由服务端分配随机子域名 |
| `--ue` | 加密传输 | 启用数据加密 |
| `--server_addr` | 服务器地址 | FRP 服务器的地址和端口 |

### 2.3 TLS 握手

如果提供了 TLS 证书（默认服务器都会提供），则追加 TLS 相关参数：

```python
if self.share_server_tls_certificate is not None:
    command.extend([
        "--tls_enable",
        "--tls_trusted_ca_file", self.share_server_tls_certificate,
    ])
```

- `--tls_enable`：启用 TLS 加密连接
- `--tls_trusted_ca_file`：指定可信 CA 证书文件，用于验证服务端身份

### 2.4 握手过程

1. **TCP 连接建立**：frpc 连接到 `remote_host:remote_port`
2. **TLS 握手**：使用指定的 CA 证书验证服务端，建立加密通道
3. **登录验证**：frpc 发送登录请求，携带 `share_token` 作为身份标识
4. **代理注册**：注册 HTTP 代理，请求随机子域名
5. **隧道就绪**：服务端确认，返回公网访问地址

---

## 三、地址生成

### 3.1 从 frpc 输出中提取地址

地址生成通过解析 frpc 进程的标准输出实现，见 `Tunnel._read_url_from_tunnel_stream()` 方法（[tunneling.py#L156-L193](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L156-L193)）。

**核心逻辑**：

```python
while url == "":
    line = self.proc.stdout.readline()
    line = line.decode("utf-8")
    
    if "start proxy success" in line:
        result = re.search("start proxy success: (.+)\n", line)
        url = result.group(1)
    elif "login to server failed" in line:
        _raise_tunnel_error()
```

### 3.2 地址格式

当 frpc 成功与服务端建立代理后，会输出类似日志：

```
start proxy success: abc123.gradio.live
```

通过正则表达式 `start proxy success: (.+)\n` 提取出公网域名。

### 3.3 协议调整

在 `Blocks.launch()` 中（[blocks.py#L3129-L3132](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3129-L3132)），会根据配置调整 URL 协议：

```python
parsed_url = urlparse(share_url)
self.share_url = urlunparse(
    (self.share_server_protocol,) + parsed_url[1:]
)
```

- 默认服务器：使用 `https` 协议
- 自定义服务器：默认使用 `http` 协议（可通过 `share_server_protocol` 覆盖）

### 3.4 超时机制

隧道建立有 30 秒超时限制（[tunneling.py#L54](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L54-L54)）：

```python
TUNNEL_TIMEOUT_SECONDS = 30
```

如果在超时时间内未读取到成功信息，则抛出异常并输出 frpc 日志。

---

## 四、完整流程时序图

```
用户代码            Blocks        networking.py       tunneling.py       frpc 进程      FRP 服务器
   |                  |               |                   |                |              |
   | launch(share=True)|               |                   |                |              |
   |----------------->|               |                   |                |              |
   |                  | 生成 share_token                   |                |              |
   |                  | (secrets.token_urlsafe(32))        |                |              |
   |                  |               |                   |                |              |
   |                  | setup_tunnel()|                   |                |              |
   |                  |-------------->|                   |                |              |
   |                  |               | 请求隧道服务器信息 |                |              |
   |                  |               | (api.gradio.app)  |                |              |
   |                  |               |------------------->|                |              |
   |                  |               |<-------------------|                |              |
   |                  |               |  返回 host/port/ca|                |              |
   |                  |               |                   |                |              |
   |                  |               | 创建 Tunnel 对象  |                |              |
   |                  |               |------------------>|                |              |
   |                  |               |                   |                |              |
   |                  |               | start_tunnel()    |                |              |
   |                  |               |------------------>|                |              |
   |                  |               |                   | 下载 frpc      |              |
   |                  |               |                   | (首次运行)     |              |
   |                  |               |                   | 校验 SHA256    |              |
   |                  |               |                   |                |              |
   |                  |               |                   | 启动子进程     |              |
   |                  |               |                   |--------------->|              |
   |                  |               |                   |                |  TCP 连接    |
   |                  |               |                   |                |------------->|
   |                  |               |                   |                |  TLS 握手    |
   |                  |               |                   |                |<============>|
   |                  |               |                   |                |  登录(带token)|
   |                  |               |                   |                |------------->|
   |                  |               |                   |                |  注册代理     |
   |                  |               |                   |                |------------->|
   |                  |               |                   |                |  分配子域名   |
   |                  |               |                   |                |<-------------|
   |                  |               |                   |  读取输出      |              |
   |                  |               |                   |<---------------|              |
   |                  |               |                   |  "start proxy success: xxx.gradio.live"
   |                  |               |                   |  正则提取 URL  |              |
   |                  |               |<------------------|                |              |
   |                  |<--------------|                   |                |              |
   |<-----------------|  返回 share_url                   |                |              |
```

---

## 五、关键设计要点

### 5.1 安全性

- **TLS 加密**：默认使用 TLS 加密隧道连接，防止中间人攻击
- **CA 证书验证**：通过服务端下发的根证书验证服务端身份
- **随机 Token**：`share_token` 长度足够（32字节随机数），难以猜测
- **SHA-256 校验**：frpc 二进制文件下载后进行完整性校验

### 5.2 可靠性

- **超时保护**：30 秒超时，避免无限等待
- **进程管理**：使用 `atexit` 注册退出钩子，程序退出时自动关闭隧道
- **全局追踪**：`CURRENT_TUNNELS` 列表追踪所有活跃隧道

### 5.3 可扩展性

- **自定义服务器**：支持通过 `share_server_address` 指定自建 FRP 服务器
- **自定义协议**：支持 `share_server_protocol` 配置 http/https
- **自定义证书**：支持 `share_server_tls_certificate` 指定 TLS 证书

---

## 六、相关环境变量

| 变量名 | 作用 |
|--------|------|
| `GRADIO_SHARE_SERVER_ADDRESS` | 自定义 FRP 分享服务器地址 |
| `HF_HOME` | frpc 二进制文件存储根目录 |
