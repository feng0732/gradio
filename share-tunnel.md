# Gradio 公网隧道（Share Tunnel）代码深度分析

## 概述

Gradio 的分享链接功能基于 **FRP（Fast Reverse Proxy）** 实现，通过在本地启动 FRP 客户端（frpc），与远程 FRP 服务器建立隧道连接，从而将本地服务暴露到公网。整个流程涉及三个核心阶段：**隧道启动**、**握手连接**、**地址生成**。

本文重点分析三个关键概念的关系：
1. **代理标识**（`share_token`）：隧道的唯一身份标识
2. **服务端分配地址**：FRP 服务端返回的公网访问地址
3. **最终链接协议**：根据配置确定的最终 URL 协议（http/https）

---

## 核心文件

| 文件 | 作用 |
|------|------|
| [gradio/tunneling.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py) | 隧道核心逻辑，封装了 `Tunnel` 类，负责 frpc 下载、启动、地址读取 |
| [gradio/networking.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/networking.py) | 网络辅助层，`setup_tunnel()` 函数协调隧道建立 |
| [gradio/blocks.py](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py) | 应用入口，`launch()` 方法触发隧道创建，生成 `share_token` 并处理最终 URL |

---

## 一、代理标识（share_token）的生命周期

### 1.1 生成时机与算法

`share_token` 是隧道的唯一身份标识，在 Blocks 初始化时生成，见 [blocks.py#L143](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L143-L143)：

```python
self.share_token = secrets.token_urlsafe(32)
```

- **生成时机**：用户创建 `gr.Blocks()` 对象时
- **算法**：Python 标准库 `secrets.token_urlsafe(32)`
- **长度**：32 字节随机数 → Base64 URL 编码后约 43 个字符
- **随机性**：密码学安全的随机数，不可预测

### 1.2 在握手中的作用

`share_token` 通过 `-n` 参数传递给 frpc，作为代理名称（proxy name）：

```python
command = [
    binary,
    "http",                  # HTTP 代理模式
    "-n", self.share_token,  # 代理名称 = share_token
    ...
]
```

**核心作用**：
1. **隧道身份标识**：FRP 服务端用它来区分不同的隧道连接
2. **代理注册标识**：注册 HTTP 代理时的唯一名称
3. **安全隔离**：足够长的随机字符串防止猜测和冲突

### 1.3 与子域名的关系

重要：`share_token` **不等于** 最终的子域名。子域名由 FRP 服务端根据 `--sd random` 参数随机生成，与 `share_token` 没有直接的数学关联。

---

## 二、握手阶段：各参数的职责

### 2.1 frpc 启动命令详解

隧道启动的核心在 `Tunnel._start_tunnel()` 方法（[tunneling.py#L123-L154](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L123-L154)），通过子进程启动 frpc：

```python
command = [
    binary,
    "http",                  # [1] 代理类型
    "-n", self.share_token,  # [2] 代理名称 = share_token
    "-l", str(self.local_port),   # [3] 本地端口
    "-i", self.local_host,        # [4] 本地主机
    "--uc",                  # [5] 启用自定义子域名
    "--sd", "random",        # [6] 子域名策略：随机
    "--ue",                  # [7] 启用数据加密
    "--server_addr", f"{self.remote_host}:{self.remote_port}",  # [8] 服务器地址
    "--disable_log_color",   # [9] 禁用日志颜色（便于解析）
]
```

### 2.2 参数职责明细

| 参数 | 全称 | 职责 | 与其他概念的关系 |
|------|------|------|----------------|
| `http` | - | 代理类型，指定为 HTTP 反向代理模式 | 决定了服务端如何处理流量 |
| `-n` | proxy name | 代理名称，使用 `share_token` 作为唯一标识 | **代理标识**的传递载体 |
| `-l` | local port | 本地服务端口 | 指向要暴露的 Gradio 本地服务 |
| `-i` | local host | 本地服务主机地址 | 指向要暴露的 Gradio 本地服务 |
| `--uc` | use custom subdomain | 启用自定义子域名功能 | 告诉服务端需要分配子域名 |
| `--sd` | subdomain | 子域名策略，`random` 表示请求随机分配 | 决定**服务端分配地址**的生成方式 |
| `--ue` | use encryption | 启用 frp 协议层的数据加密 | 保障隧道内传输安全 |
| `--server_addr` | server address | FRP 服务端地址和端口 | 握手的目标服务器 |
| `--disable_log_color` | - | 禁用彩色日志输出 | 便于后续正则解析地址 |

### 2.3 TLS 握手参数

如果提供了 TLS 证书（默认官方服务器都会提供），则追加以下参数：

```python
if self.share_server_tls_certificate is not None:
    command.extend([
        "--tls_enable",                     # 启用 TLS 加密连接
        "--tls_trusted_ca_file",            # 指定 CA 证书文件路径
        self.share_server_tls_certificate,  # 从 API 服务器获取的根证书
    ])
```

**注意**：`--ue` 和 `--tls_enable` 是两层不同的加密：
- `--ue`：FRP 协议层面的加密
- `--tls_enable`：传输层的 TLS 加密

### 2.4 握手完整流程

握手过程包括以下步骤：

```
本地 frpc                         FRP 服务端
   |                                 |
   | 1. TCP 连接                     |
   |------------------------------->|
   |                                 |
   | 2. TLS 握手（如果启用）         |
   |<========= 加密通道 ==========>|
   |                                 |
   | 3. 登录请求                     |
   |   (携带 share_token)            |
   |------------------------------->|
   |                                 |
   | 4. 登录响应                     |
   |<-------------------------------|
   |                                 |
   | 5. 代理注册请求                 |
   |   类型: http                    |
   |   名称: share_token             |
   |   子域名: random                |
   |------------------------------->|
   |                                 |
   | 6. 代理注册响应                 |
   |   分配子域名: abc123            |
   |   访问地址: http://abc123.gradio.live
   |<-------------------------------|
   |                                 |
   | 7. 输出日志                     |
   |   "start proxy success: http://abc123.gradio.live"
   |<--- 本地 Python 进程解析此输出
```

---

## 三、服务端分配地址的实际格式

### 3.1 地址来源

服务端分配的地址通过 frpc 进程的标准输出返回，由 `_read_url_from_tunnel_stream()` 方法解析（[tunneling.py#L156-L193](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/tunneling.py#L156-L193)）。

**解析逻辑**：

```python
if "start proxy success" in line:
    result = re.search("start proxy success: (.+)\n", line)
    url = result.group(1)
```

### 3.2 实际格式验证

通过 `urlparse` + `urlunparse` 的组合使用方式，可以推断出服务端返回的实际格式。让我们验证两种可能性：

```python
from urllib.parse import urlparse, urlunparse

# 情况 1：返回带协议的完整 URL
share_url = "http://abc123.gradio.live"
parsed = urlparse(share_url)
# scheme='http', netloc='abc123.gradio.live', path=''
new_url = urlunparse(('https',) + parsed[1:])
# 结果: 'https://abc123.gradio.live'  ✓ 正确

# 情况 2：返回纯域名
share_url = "abc123.gradio.live"
parsed = urlparse(share_url)
# scheme='', netloc='', path='abc123.gradio.live'
new_url = urlunparse(('https',) + parsed[1:])
# 结果: 'https:abc123.gradio.live'  ✗ 错误（缺少 //）
```

**结论**：服务端返回的是**带协议的完整 URL**，格式为 `http://<随机子域名>.gradio.live`。

### 3.3 子域名的生成

子域名由 FRP 服务端生成，与 `--sd random` 参数相关：
- 客户端请求 `--sd random` 表示"请给我一个随机子域名"
- 服务端生成一个随机字符串（如 `abc123`）作为子域名
- 最终地址格式：`http://abc123.gradio.live`
- 子域名与 `share_token` 没有直接关联

---

## 四、本地 URL 改写与协议转换

### 4.1 协议选择逻辑

最终 URL 的协议由 `share_server_protocol` 决定，其初始化逻辑在 [blocks.py#L2966-L2968](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L2966-L2968)：

```python
self.share_server_protocol = share_server_protocol or (
    "http" if share_server_address is not None else "https"
)
```

**规则**：

| 场景 | share_server_protocol 默认值 | 原因 |
|------|----------------------------|------|
| 使用官方服务器（无自定义地址） | `https` | 官方服务器配置了 TLS 证书，提供安全连接 |
| 使用自定义服务器 | `http` | 自定义服务器可能未配置 TLS，默认使用明文 |
| 用户显式指定参数 | 用户指定值 | 优先级最高，覆盖默认值 |

### 4.2 URL 改写过程

在 [blocks.py#L3129-L3132](file:///d:/fz/0601/solo-dogfeeding/code/244-gradio/gradio/blocks.py#L3129-L3132) 中完成最终 URL 的构造：

```python
share_url = networking.setup_tunnel(...)  # 返回 "http://abc123.gradio.live"
parsed_url = urlparse(share_url)
self.share_url = urlunparse(
    (self.share_server_protocol,) + parsed_url[1:]
)
```

**详细步骤**：

1. **获取服务端地址**：`setup_tunnel()` 返回 `http://abc123.gradio.live`
2. **URL 解析**：`urlparse()` 将 URL 分解为 6 个组成部分：
   ```
   scheme   = 'http'
   netloc   = 'abc123.gradio.live'
   path     = ''
   params   = ''
   query    = ''
   fragment = ''
   ```
3. **替换协议**：用 `share_server_protocol` 替换原来的 `scheme`
4. **重新组合**：`urlunparse()` 将各部分重新组合成完整 URL

**示例**：

| 场景 | 服务端返回 | share_server_protocol | 最终 URL |
|------|-----------|----------------------|----------|
| 官方服务器 | `http://abc123.gradio.live` | `https` | `https://abc123.gradio.live` |
| 自定义服务器（默认） | `http://xyz.local:8080` | `http` | `http://xyz.local:8080` |
| 自定义服务器（指定 https） | `http://xyz.local:8080` | `https` | `https://xyz.local:8080` |

### 4.3 为什么需要改写协议？

frpc 返回的地址使用 `http` 协议（因为 frp 内部通信使用 http），但：
1. 官方服务器在网关层提供了 TLS 终结，所以对外应该使用 `https`
2. 自定义服务器可能有也可能没有 TLS，所以给用户选择权
3. 协议改写只修改 URL 的 scheme 部分，不影响其他部分

---

## 五、三者关系的完整视图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      概念关系与数据流                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 代理标识 (share_token)                                          │
│     ┌─────────────────────────────────────────┐                    │
│     │ 生成: secrets.token_urlsafe(32)         │                    │
│     │ 位置: Blocks.__init__                   │                    │
│     │ 作用: 隧道唯一身份标识                   │                    │
│     └───────────────┬─────────────────────────┘                    │
│                     │                                              │
│                     ▼                                              │
│     ┌─────────────────────────────────────────┐                    │
│     │ frpc -n <share_token> --sd random       │  ◄── 握手阶段      │
│     └───────────────┬─────────────────────────┘                    │
│                     │                                              │
│  2. 服务端分配地址                                                 │
│     ┌───────────────▼─────────────────────────┐                    │
│     │ FRP 服务端生成随机子域名                 │                    │
│     │ 返回: http://abc123.gradio.live          │                    │
│     └───────────────┬─────────────────────────┘                    │
│                     │                                              │
│                     ▼                                              │
│     ┌─────────────────────────────────────────┐                    │
│     │ parsed_url = urlparse(share_url)        │  ◄── 解析阶段      │
│     │  scheme: 'http'                         │                    │
│     │  netloc: 'abc123.gradio.live'           │                    │
│     └───────────────┬─────────────────────────┘                    │
│                     │                                              │
│  3. 最终链接协议                                                   │
│     ┌───────────────▼─────────────────────────┐                    │
│     │ share_server_protocol                   │                    │
│     │  官方服务器: 'https'                    │                    │
│     │  自定义服务器: 'http'                   │  ◄── 协议选择      │
│     └───────────────┬─────────────────────────┘                    │
│                     │                                              │
│                     ▼                                              │
│     ┌─────────────────────────────────────────┐                    │
│     │ urlunparse(                             │  ◄── 最终生成      │
│     │   (share_server_protocol,)              │                    │
│     │   + parsed_url[1:]                      │                    │
│     │ )                                       │                    │
│     └───────────────┬─────────────────────────┘                    │
│                     │                                              │
│                     ▼                                              │
│     ┌─────────────────────────────────────────┐                    │
│     │ 最终 share_url: https://abc123.gradio.live │                 │
│     └─────────────────────────────────────────┘                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、完整流程时序图

```
用户代码            Blocks        networking.py       tunneling.py       frpc 进程      FRP 服务器
   |                  |               |                   |                |              |
   | Blocks()         |               |                   |                |              |
   |----------------->|               |                   |                |              |
   |                  | 生成 share_token                   |                |              |
   |                  | secrets.token_urlsafe(32)          |                |              |
   |                  |               |                   |                |              |
   | launch(share=True)|               |                   |                |              |
   |----------------->|               |                   |                |              |
   |                  | 确定 share_server_protocol        |                |              |
   |                  | (官方->https, 自定义->http)       |                |              |
   |                  |               |                   |                |              |
   |                  | setup_tunnel()|                   |                |              |
   |                  |-------------->|                   |                |              |
   |                  |               | 请求隧道服务器信息 |                |              |
   |                  |               | api.gradio.app    |                |              |
   |                  |               |------------------->|                |              |
   |                  |               |<-------------------|                |              |
   |                  |               | host/port/ca      |                |              |
   |                  |               |                   |                |              |
   |                  |               | 创建 Tunnel 对象  |                |              |
   |                  |               |------------------>|                |              |
   |                  |               |                   |                |              |
   |                  |               | start_tunnel()    |                |              |
   |                  |               |------------------>|                |              |
   |                  |               |                   | 下载 frpc      |              |
   |                  |               |                   | SHA256 校验    |              |
   |                  |               |                   |                |              |
   |                  |               |                   | 启动子进程     |              |
   |                  |               |                   | [http,         |              |
   |                  |               |                   |  -n token,     |              |
   |                  |               |                   |  --uc,         |              |
   |                  |               |                   |  --sd random,  |              |
   |                  |               |                   |  --ue, ...]    |              |
   |                  |               |                   |--------------->|              |
   |                  |               |                   |                |  TCP 连接    |
   |                  |               |                   |                |------------->|
   |                  |               |                   |                |  TLS 握手    |
   |                  |               |                   |                |<============>|
   |                  |               |                   |                |  登录(token) |
   |                  |               |                   |                |------------->|
   |                  |               |                   |                |  注册代理    |
   |                  |               |                   |                | (random 子域)|
   |                  |               |                   |                |------------->|
   |                  |               |                   |                |  分配子域名  |
   |                  |               |                   |                |<-------------|
   |                  |               |                   |  读取 stdout   |              |
   |                  |               |                   |<---------------|              |
   |                  |               |                   |  "start proxy  |              |
   |                  |               |                   |   success:     |              |
   |                  |               |                   |   http://abc123|              |
   |                  |               |                   |   .gradio.live"|              |
   |                  |               |                   |  正则提取      |              |
   |                  |               |                   |  url = "http://|              |
   |                  |               |                   |  abc123.gradio.|              |
   |                  |               |                   |  live"         |              |
   |                  |               |<------------------|                |              |
   |                  |<--------------| 返回原始 URL       |                |              |
   |                  |               |                   |                |              |
   |                  |  urlparse("http://abc123...")      |                |              |
   |                  |  scheme='http', netloc='abc...'    |                |              |
   |                  |               |                   |                |              |
   |                  |  urlunparse(                       |                |              |
   |                  |    ('https',) + parsed[1:]         |                |              |
   |                  |  )                                 |                |              |
   |                  |               |                   |                |              |
   |                  |  最终 URL:                         |                |              |
   |                  |  https://abc123.gradio.live        |                |              |
   |<-----------------| 返回 share_url                     |                |              |
```

---

## 七、关键设计要点

### 7.1 安全性

- **TLS 加密**：官方服务器默认使用 TLS 加密隧道连接，防止中间人攻击
- **CA 证书验证**：通过服务端下发的根证书验证服务端身份
- **随机 Token**：`share_token` 使用密码学安全随机数，长度足够（32字节），难以猜测
- **SHA-256 校验**：frpc 二进制文件下载后进行完整性校验，防止篡改
- **双层加密**：`--ue`（FRP 协议加密）+ `--tls_enable`（传输层加密）

### 7.2 协议设计的权衡

| 决策 | 原因 | 潜在问题 |
|------|------|----------|
| frpc 返回 `http://` | frp 内部通信使用 http | 需要在 Python 层改写协议 |
| 官方服务器改写为 `https` | 网关层有 TLS 终结 | 依赖基础设施配置 |
| 自定义服务器默认为 `http` | 不确定自定义服务器是否有 TLS | 可能导致不安全连接（用户需显式指定 https） |

### 7.3 可扩展性

- **自定义服务器**：支持通过 `share_server_address` 指定自建 FRP 服务器
- **自定义协议**：支持 `share_server_protocol` 显式指定 http/https
- **自定义证书**：支持 `share_server_tls_certificate` 指定 TLS 证书

---

## 八、相关环境变量

| 变量名 | 作用 |
|--------|------|
| `GRADIO_SHARE_SERVER_ADDRESS` | 自定义 FRP 分享服务器地址（格式: host:port） |
| `HF_HOME` | frpc 二进制文件存储根目录，默认为 `~/.cache/huggingface` |

---

## 九、常见问题

**Q: `share_token` 和子域名有什么关系？**
A: 没有直接关系。`share_token` 是隧道的内部标识，子域名由 FRP 服务端随机生成。

**Q: 为什么不直接让 frpc 返回 `https://` 地址？**
A: frp 是通用的反向代理工具，不感知上层的 TLS 配置。官方服务器的 TLS 终结在网关层，frp 本身只处理 http 流量。

**Q: 自定义服务器时如何使用 https？**
A: 需要同时满足两个条件：(1) 自定义服务器配置了 TLS 证书；(2) 在 `launch()` 时显式指定 `share_server_protocol="https"` 和 `share_server_tls_certificate`。

**Q: 为什么需要 `--uc` 和 `--sd random` 两个参数？**
A: `--uc` 启用子域名功能，`--sd random` 指定子域名的生成策略为随机。两者配合使用实现随机子域名分配。
