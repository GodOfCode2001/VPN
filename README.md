# 基于 Cloudflare Workers 的 VLESS 代理服务
流量转发知识库

## 目录
- [一、概述](#一概述)
- [二、核心概念](#二核心概念)
- [三、整体架构](#三整体架构)
- [四、代码逐层解析](#四代码逐层解析)
- [五、流量转发流程详解](#五流量转发流程详解)
- [六、协议解析机制](#六协议解析机制)
- [七、部署与配置](#七部署与配置)
- [八、常见问题](#八常见问题)
- [九、扩展与优化](#九扩展与优化)
- [附录](#附录)

---

## 一、概述

### 1.1 什么是 Cloudflare Workers
Cloudflare Workers 是一个基于 V8 引擎的 Serverless 平台，允许开发者在全球 300+ 个数据中心运行 JavaScript/WebAssembly 代码。Workers 可以拦截和处理所有经过 Cloudflare CDN 的 HTTP/WebSocket 请求。

### 1.2 代理服务的核心价值
本项目实现的是一个 VLESS 协议代理服务，利用 Cloudflare Workers 的特性：
- 全球节点覆盖，提供低延迟访问
- 无需维护服务器，免运维成本
- WebSocket 长连接支持，适合代理场景
- 可自定义路由和转发规则

### 1.3 适用场景
| 场景 | 说明 |
| --- | --- |
| 科学上网 | VLESS + WebSocket + TLS 加密传输 |
| API 网关 | 请求路由与负载均衡 |
| 流量中转 | 通过 SOCKS5 或自定义路由转发流量 |
| DNS 解析 | 内置 DNS over TCP 代理 |

---

## 二、核心概念

### 2.1 VLESS 协议
VLESS 是 V2Ray 项目中的一种轻量级代理协议，特点：
- 无状态：不需要握手确认
- UUID 鉴权：使用 UUID 验证客户端身份
- 支持多种传输方式：TCP、WebSocket、mKCP 等
- 内置加密：支持 TLS 加密传输

### 2.2 WebSocket 与 TCP 桥接
```text
┌─────────────┐     WebSocket      ┌─────────────┐      TCP      ┌─────────────┐
│   客户端     │ ◄──────────────►  │    Worker   │ ◄───────────► │  目标服务器  │
│ (VLESS客户端)│                    │   (代理中转) │               │   (目标网站) │
└─────────────┘                    └─────────────┘               └─────────────┘
```
WebSocket 提供全双工通信，TCP Socket API 实现与目标服务器连接，二者通过 Streams API 桥接。

### 2.3 Cloudflare Socket API
```javascript
import { connect } from 'cloudflare:sockets';
```
这是 Cloudflare Workers 特有的 API，允许 Worker 主动建立 TCP 连接：
- 支持 IPv4/IPv6
- 支持 TLS
- 返回 Socket 对象，包含 readable 和 writable 流

---

## 三、整体架构

### 3.1 系统分层
```text
┌─────────────────────────────────────────────────────────────────────────┐
│                           HTTP 请求入口 (fetch)                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────┐    ┌─────────────────────────────────────┐     │
│  │   普通 HTTP 请求      │    │       WebSocket 升级请求            │     │
│  │   (路径路由处理)      │    │       (VLESS 代理处理)              │     │
│  └─────────────────────┘    └─────────────────────────────────────┘     │
│            │                              │                              │
│            ▼                              ▼                              │
│  ┌─────────────────────┐    ┌─────────────────────────────────────┐     │
│  │  /  → 返回 CF 信息   │    │   VLESS 协议头解析                  │     │
│  │  /{uuid} → 订阅配置  │    │   (processVlessHeader)              │     │
│  └─────────────────────┘    └─────────────────────────────────────┘     │
│                                         │                              │
│                                         ▼                              │
│                              ┌─────────────────────────────────────┐     │
│                              │   目标地址解析                       │     │
│                              │   (域名 / IPv4 / IPv6)              │     │
│                              └─────────────────────────────────────┘     │
│                                         │                              │
│                    ┌────────────────────┼────────────────────┐         │
│                    ▼                    ▼                    ▼         │
│             ┌──────────┐       ┌──────────┐       ┌──────────┐        │
│             │ TCP 转发  │       │ UDP DNS  │       │ SOCKS5   │        │
│             │          │       │ 处理     │       │ 代理     │        │
│             └──────────┘       └──────────┘       └──────────┘        │
│                    │                    │                    │         │
│                    └────────────────────┼────────────────────┘         │
│                                         ▼                              │
│                              ┌─────────────────────────────────────┐     │
│                              │   Cloudflare Socket API             │     │
│                              │   connect(hostname, port)           │     │
│                              └─────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 数据流向
```text
客户端 → Worker(入口) → 协议解析 → 路由决策 → 目标服务器
客户端 ← Worker(出口) ← 数据封装 ← 响应接收 ← 目标服务器
```

---

## 四、代码逐层解析

### 4.1 模块导入与全局配置
```javascript
import { connect } from 'cloudflare:sockets';

let userID = 'f974a9f4-7bcb-4961-9913-f87e75984635';
let proxyIP = '';
let sub = 'vless-4ca.pages.dev';
let subconverter = 'api.v1.mk';
let subconfig = "[https://raw.githubusercontent.com/.../ACL4SSR_Online_Full_MultiMode.ini](https://raw.githubusercontent.com/.../ACL4SSR_Online_Full_MultiMode.ini)";
let socks5Address = '';
let RproxyIP = 'false';
```

关键变量说明：
| 变量 | 类型 | 说明 |
| --- | --- | --- |
| `userID` | `string` | VLESS 鉴权 UUID |
| `proxyIP` | `string` | 中转 IP，将流量先发往此 IP |
| `sub` | `string` | 订阅生成器地址 |
| `subconverter` | `string` | 订阅转换后端 |
| `socks5Address` | `string` | SOCKS5 代理地址 |
| `RproxyIP` | `string` | 是否在订阅中启用 proxyIP |

### 4.2 主入口：fetch 函数
```javascript
export default {
    async fetch(request, env, ctx) {
        // 1. 环境变量覆盖
        userID = (env.UUID || userID).toLowerCase();
        proxyIP = env.PROXYIP || proxyIP;
        socks5Address = env.SOCKS5 || socks5Address;
        
        // 2. 解析 SOCKS5 地址
        if (socks5Address) {
            parsedSocks5Address = socks5AddressParser(socks5Address);
            enableSocks = true;
        }
        
        // 3. 判断请求类型
        const upgradeHeader = request.headers.get('Upgrade');
        const url = new URL(request.url);
        
        if (!upgradeHeader || upgradeHeader !== 'websocket') {
            // 普通 HTTP 请求处理
            return handleHTTPRequest(request, url);
        } else {
            // WebSocket 代理处理
            return await vlessOverWSHandler(request);
        }
    }
};
```
设计要点：
- 环境变量优先级最高，方便部署时动态修改
- 通过 Upgrade 头区分 WebSocket 和普通请求
- 利用 `url.pathname` 实现路由分发

### 4.3 普通 HTTP 请求处理
```javascript
switch (url.pathname.toLowerCase()) {
    case '/':
        // 返回访问者的 Cloudflare 信息
        return new Response(JSON.stringify(request.cf), { status: 200 });
        
    case `/${userID}`:
        // 返回 VLESS 配置或订阅
        const vlessConfig = await getVLESSConfig(...);
        return new Response(vlessConfig, {
            headers: {
                "Content-Disposition": "attachment; filename=edgetunnel",
                "Content-Type": "text/plain;charset=utf-8",
                "Profile-Update-Interval": "6",
                "Subscription-Userinfo": `upload=${UD}; download=${UD}; ...`
            }
        });
        
    default:
        return new Response('Not found', { status: 404 });
}
```
订阅头信息说明：
- `Profile-Update-Interval`: 客户端更新间隔（6小时）
- `Subscription-Userinfo`: 流量统计信息（上传/下载/总量/过期时间）

### 4.4 VLESS 协议头解析
`processVlessHeader` 函数是核心解析器：
```javascript
function processVlessHeader(vlessBuffer, userID) {
    // 最小长度校验：24 字节
    if (vlessBuffer.byteLength < 24) {
        return { hasError: true, message: 'invalid data' };
    }
    
    // 1. 版本号 (第1字节)
    const version = new Uint8Array(vlessBuffer.slice(0, 1));
    
    // 2. UUID 验证 (第2-17字节)
    if (stringify(new Uint8Array(vlessBuffer.slice(1, 17))) !== userID) {
        return { hasError: true, message: 'invalid user' };
    }
    
    // 3. 可选长度 (第18字节)
    const optLength = new Uint8Array(vlessBuffer.slice(17, 18))[0];
    
    // 4. 命令类型 (跳过可选字段后)
    const command = new Uint8Array(
        vlessBuffer.slice(18 + optLength, 18 + optLength + 1)
    )[0];
    // 0x01 = TCP, 0x02 = UDP, 0x03 = MUX
    
    // 5. 端口 (大端序)
    const portBuffer = vlessBuffer.slice(portIndex, portIndex + 2);
    const portRemote = new DataView(portBuffer).getUint16(0);
    
    // 6. 地址类型和地址
    const addressType = addressBuffer[0];
    // 1 = IPv4, 2 = 域名, 3 = IPv6
}
```

VLESS 协议头结构：
| 偏移量 (字节) | 长度 (字节) | 字段说明 |
| --- | --- | --- |
| 0 | 1 | 协议版本 (0x00) |
| 1 | 16 | UUID (128位) |
| 17 | 1 | 可选字段长度 N |
| 18 | N | 可选字段 |
| 18+N | 1 | 命令类型 (1-TCP, 2-UDP, 3-MUX) |
| 19+N | 2 | 目标端口 (大端序) |
| 21+N | 1 | 地址类型 (1-IPv4, 2-域名, 3-IPv6) |
| 22+N | 变长 | 目标地址 |

### 4.5 WebSocket 流处理
```javascript
async function vlessOverWSHandler(request) {
    // 1. 创建 WebSocket 对
    const webSocketPair = new WebSocketPair();
    const [client, webSocket] = Object.values(webSocketPair);
    webSocket.accept();
    
    // 2. 获取 0-RTT 数据
    const earlyDataHeader = request.headers.get('sec-websocket-protocol') || '';
    
    // 3. 将 WebSocket 转为 ReadableStream
    const readableWebSocketStream = makeReadableWebSocketStream(
        webSocket, earlyDataHeader, log
    );
    
    // 4. 流处理管道：读取 → 解析 → 转发
    readableWebSocketStream.pipeTo(new WritableStream({
        async write(chunk, controller) {
            // 处理每个数据块
            const parsed = processVlessHeader(chunk, userID);
            // 根据解析结果决定转发目标
        }
    }));
    
    return new Response(null, {
        status: 101,  // WebSocket 升级成功
        webSocket: client,
    });
}
```

### 4.6 TCP 连接与数据转发
```javascript
async function handleTCPOutBound(remoteSocket, addressType, addressRemote, 
                                 portRemote, rawClientData, webSocket, ...) {
    // 1. 建立 TCP 连接（支持直接连接或通过代理）
    async function connectAndWrite(address, port, socks = false) {
        const tcpSocket = socks ? 
            await socks5Connect(addressType, address, port, log) :
            connect({ hostname: address, port: port });
            
        remoteSocket.value = tcpSocket;
        
        // 写入客户端发送的首批数据
        const writer = tcpSocket.writable.getWriter();
        await writer.write(rawClientData);
        writer.releaseLock();
        
        return tcpSocket;
    }
    
    // 2. 尝试连接
    let tcpSocket = await connectAndWrite(addressRemote, portRemote);
    
    // 3. 配置重试机制（如果连接失败或没有数据返回）
    async function retry() {
        tcpSocket = await connectAndWrite(proxyIP || addressRemote, portRemote);
        remoteSocketToWS(tcpSocket, webSocket, vlessResponseHeader, null, log);
    }
    
    // 4. 将 TCP 数据转发到 WebSocket
    remoteSocketToWS(tcpSocket, webSocket, vlessResponseHeader, retry, log);
}
```

### 4.7 数据流管道构建
```javascript
function makeReadableWebSocketStream(webSocketServer, earlyDataHeader, log) {
    return new ReadableStream({
        start(controller) {
            // WebSocket 消息监听
            webSocketServer.addEventListener('message', (event) => {
                controller.enqueue(event.data);
            });
            
            // 关闭事件处理
            webSocketServer.addEventListener('close', () => {
                safeCloseWebSocket(webSocketServer);
                controller.close();
            });
            
            // 0-RTT 数据：在 WebSocket 握手完成前发送的数据
            const { earlyData } = base64ToArrayBuffer(earlyDataHeader);
            if (earlyData) {
                controller.enqueue(earlyData);
            }
        },
        cancel(reason) {
            // 流取消时关闭 WebSocket
            safeCloseWebSocket(webSocketServer);
        }
    });
}
```

### 4.8 SOCKS5 代理支持
```javascript
async function socks5Connect(addressType, addressRemote, portRemote, log) {
    // 1. 连接 SOCKS5 服务器
    const socket = connect({
        hostname: parsedSocks5Address.hostname,
        port: parsedSocks5Address.port,
    });
    
    // 2. 发送握手请求 (无认证/用户名密码认证)
    const socksGreeting = new Uint8Array([5, 2, 0, 2]);
    await writer.write(socksGreeting);
    
    // 3. 处理认证响应
    // ...
    
    // 4. 发送连接请求
    const socksRequest = new Uint8Array([5, 1, 0, ...DSTADDR, portRemote >> 8, portRemote & 0xff]);
    await writer.write(socksRequest);
    
    // 5. 返回已连接的 Socket
    return socket;
}
```

SOCKS5 握手流程：
```text
客户端 → 服务器: [VER(5), NMETHODS, METHODS...]
服务器 → 客户端: [VER(5), METHOD]
客户端 → 服务器: [VER(5), CMD(1), RSV(0), ATYP, DST.ADDR, DST.PORT]
服务器 → 客户端: [VER(5), REP, RSV, ATYP, BND.ADDR, BND.PORT]
```

---

## 五、流量转发流程详解

### 5.1 完整数据流时序图
```text
客户端          Cloudflare Worker         目标服务器
  │                    │                       │
  │── WebSocket 握手 ──▶│                       │
  │◀── 101 Switching ──│                       │
  │                    │                       │
  │── VLESS 协议头 ────▶│                       │
  │  (UUID + 目标地址)  │                       │
  │                    │                       │
  │                    │── TCP connect ────────▶│
  │                    │                       │
  │                    │◀── TCP established ───│
  │                    │                       │
  │── TLS ClientHello ─▶│── 转发 ─────────────▶│
  │                    │                       │
  │◀── TLS ServerHello─│◀── 转发 ─────────────│
  │                    │                       │
  │── HTTP 请求 ───────▶│── 转发 ─────────────▶│
  │                    │                       │
  │◀── HTTP 响应 ──────│◀── 转发 ─────────────│
  │                    │                       │
  │── WebSocket 关闭 ──▶│── 关闭连接 ──────────▶│
  │                    │                       │
```

### 5.2 关键转发决策
```javascript
// 在 WritableStream 的 write 方法中
async write(chunk, controller) {
    // 情况1: DNS 查询 (UDP 53)
    if (isDns) {
        return await handleDNSQuery(chunk, webSocket, null, log);
    }
    
    // 情况2: 已建立连接，直接转发
    if (remoteSocketWapper.value) {
        const writer = remoteSocketWapper.value.writable.getWriter();
        await writer.write(chunk);
        writer.releaseLock();
        return;
    }
    
    // 情况3: 首次数据，解析 VLESS 头
    const parsed = processVlessHeader(chunk, userID);
    
    // 根据协议类型分发
    if (parsed.isUDP) {
        // UDP → DNS 处理
        handleDNSQuery(...);
    } else {
        // TCP → 建立连接并转发
        handleTCPOutBound(...);
    }
}
```

### 5.3 连接复用与重试机制
```javascript
async function remoteSocketToWS(remoteSocket, webSocket, vlessResponseHeader, retry, log) {
    let hasIncomingData = false;
    
    await remoteSocket.readable.pipeTo(
        new WritableStream({
            async write(chunk) {
                hasIncomingData = true;  // 标记有数据到达
                webSocket.send(chunk);    // 转发到 WebSocket
            },
            close() {
                // 连接关闭处理
            }
        })
    );
    
    // 如果连接建立后没有数据返回，触发重试
    if (hasIncomingData === false && retry) {
        log(`retry`);
        retry();  // 使用 proxyIP 重试
    }
}
```

重试策略：
- 首次连接使用 VLESS 头中的目标地址
- 如果连接成功但无数据返回，使用 proxyIP 重试
- 支持 SOCKS5 代理作为备选路由

---

## 六、协议解析机制

### 6.1 UUID 验证
```javascript
function isValidUUID(uuid) {
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[4][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    return uuidRegex.test(uuid);
}

function stringify(arr, offset = 0) {
    const uuid = unsafeStringify(arr, offset);
    if (!isValidUUID(uuid)) {
        throw TypeError("Stringified UUID is invalid");
    }
    return uuid;
}
```

### 6.2 地址解析
| 地址类型 | 值 | 长度 | 格式 |
| --- | --- | --- | --- |
| IPv4 | 1 | 4 | 141.2.3.4 |
| 域名 | 2 | 变长 | example.com |
| IPv6 | 3 | 16 | 2001:db8::1 |

```javascript
switch (addressType) {
    case 1:  // IPv4
        addressValue = new Uint8Array(vlessBuffer.slice(...)).join('.');
        break;
    case 2:  // 域名
        addressLength = vlessBuffer[addressValueIndex];
        addressValue = new TextDecoder().decode(vlessBuffer.slice(...));
        break;
    case 3:  // IPv6
        // 每 2 字节一组，转成 16 进制
        for (let i = 0; i < 8; i++) {
            ipv6.push(dataView.getUint16(i * 2).toString(16));
        }
        addressValue = ipv6.join(':');
        break;
}
```

### 6.3 Base64 0-RTT 数据处理
```javascript
function base64ToArrayBuffer(base64Str) {
    // URL-safe Base64 转换 (rfc4648)
    base64Str = base64Str.replace(/-/g, '+').replace(/_/g, '/');
    const decode = atob(base64Str);
    const arryBuffer = Uint8Array.from(decode, (c) => c.charCodeAt(0));
    return { earlyData: arryBuffer.buffer, error: null };
}
```

---

## 七、部署与配置

### 7.1 部署步骤
1. **注册 Cloudflare 账号**
2. **创建 Worker**
   - 进入 Cloudflare Dashboard → Workers & Pages
   - 创建新的 Worker
   - 粘贴代码
3. **配置环境变量**
   ```bash
   # 在 Worker 设置中添加环境变量
   UUID = "你的 UUID"
   PROXYIP = "中转 IP"
   SOCKS5 = "user:pass@host:port"
   SUB = "订阅生成器地址"
   SUBAPI = "订阅转换后端"
   ```
4. **绑定自定义域名（可选）**
   - Worker → Triggers → Custom Domains
   - 添加你自己的域名

### 7.2 客户端配置

V2Ray / Xray 客户端配置：
```json
{
  "inbounds": [...],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "your-worker.workers.dev",
            "port": 443,
            "users": [
              {
                "id": "your-uuid",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "wsSettings": {
          "path": "/",
          "headers": {
            "Host": "your-worker.workers.dev"
          }
        }
      }
    }
  ]
}
```

Clash-Meta 配置：
```yaml
proxies:
  - type: vless
    name: CF-Worker
    server: your-worker.workers.dev
    port: 443
    uuid: your-uuid
    network: ws
    tls: true
    sni: your-worker.workers.dev
    ws-opts:
      path: /
      headers:
        Host: your-worker.workers.dev
```

### 7.3 订阅使用
访问以下地址获取订阅：
```text
[https://your-worker.workers.dev/your-uuid](https://your-worker.workers.dev/your-uuid)
```
返回内容包含：
- VLESS 链接（V2Ray 格式）
- Clash-Meta 配置
- Base64 订阅内容

---

## 八、常见问题

### 8.1 WebSocket 连接失败
**原因：**
- Worker 域名被墙
- TLS 证书问题
- 请求头不完整

**解决：**
```javascript
// 检查 Upgrade 头
if (!upgradeHeader || upgradeHeader !== 'websocket') {
    // 不是 WebSocket 请求
}

// 确保使用 HTTPS
const url = new URL(request.url);
if (url.protocol !== 'https:') {
    return new Response('Please use HTTPS', { status: 426 });
}
```

### 8.2 连接超时
**Cloudflare Workers 限制：**
- 免费套餐：10ms CPU 时间，请求总时长 30s
- 付费套餐：50ms CPU 时间，请求总时长 600s

**优化策略：**
```javascript
// 使用流式处理，减少内存占用
readableWebSocketStream.pipeTo(new WritableStream({
    write(chunk) {
        // 增量处理，不缓存
    }
}));
```

### 8.3 DNS 解析问题
**内置 DNS 代理：**
```javascript
async function handleDNSQuery(udpChunk, webSocket, vlessResponseHeader, log) {
    // 使用固定的 DNS 服务器
    const dnsServer = '8.8.4.4';
    const dnsPort = 53;
    
    const tcpSocket = connect({
        hostname: dnsServer,
        port: dnsPort,
    });
    // 转发 DNS 查询
}
```

### 8.4 流量统计不准确
```javascript
// Subscription-Userinfo 计算
const today = new Date(now);
today.setHours(0, 0, 0, 0);
const UD = Math.floor(((now - today.getTime()) / 86400000) * 24 * 1099511627776 / 2);
// 每天递增，模拟流量使用
```

---

## 九、扩展与优化

### 9.1 多用户支持
```javascript
const userMap = {
    'user1-uuid': { name: 'user1', quota: 100 },
    'user2-uuid': { name: 'user2', quota: 200 },
};

function processVlessHeader(vlessBuffer) {
    const userUUID = stringify(new Uint8Array(vlessBuffer.slice(1, 17)));
    const userInfo = userMap[userUUID];
    if (!userInfo) {
        return { hasError: true, message: 'invalid user' };
    }
    // 继续处理...
}
```

### 9.2 负载均衡
```javascript
const proxyIPs = ['ip1', 'ip2', 'ip3'];
let currentIndex = 0;

function getNextProxyIP() {
    const ip = proxyIPs[currentIndex % proxyIPs.length];
    currentIndex++;
    return ip;
}
```

### 9.3 缓存优化
```javascript
// 缓存 DNS 解析结果
const dnsCache = new Map();

async function cachedDNSLookup(domain) {
    if (dnsCache.has(domain)) {
        const { ip, expire } = dnsCache.get(domain);
        if (Date.now() < expire) {
            return ip;
        }
    }
    
    // 执行 DNS 查询
    const ip = await dnsLookup(domain);
    dnsCache.set(domain, { ip, expire: Date.now() + 300000 }); // 5分钟缓存
    return ip;
}
```

### 9.4 监控与日志
```javascript
// 结构化日志
function log(level, message, data) {
    console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        level,
        message,
        data,
        worker: 'vless-proxy'
    }));
}

// 使用示例
log('info', 'connection established', {
    clientIP: request.headers.get('CF-Connecting-IP'),
    target: addressRemote,
    port: portRemote
});
```

### 9.5 安全增强
```javascript
// IP 白名单
const ALLOWED_IPS = ['192.168.1.0/24', '10.0.0.0/8'];

function isIPAllowed(ip) {
    // 实现 IP 匹配逻辑
    return ALLOWED_IPS.some(range => ipInRange(ip, range));
}

// 请求频率限制
const rateLimit = new Map();

function checkRateLimit(ip) {
    const now = Date.now();
    const window = 60000; // 1分钟
    const limit = 100; // 100次/分钟
    
    if (!rateLimit.has(ip)) {
        rateLimit.set(ip, { count: 1, reset: now + window });
        return true;
    }
    
    const record = rateLimit.get(ip);
    if (now > record.reset) {
        record.count = 1;
        record.reset = now + window;
        return true;
    }
    
    record.count++;
    return record.count <= limit;
}
```

---

## 附录

### A. 完整代码结构
```text
cloudflare-vless-proxy/
├── index.js              # 主入口文件
├── src/
│   ├── handler.js        # 请求处理器
│   ├── vless.js          # VLESS 协议解析
│   ├── socket.js         # Socket 连接管理
│   ├── dns.js            # DNS 代理
│   ├── socks5.js         # SOCKS5 客户端
│   ├── subscribe.js      # 订阅生成
│   └── utils.js          # 工具函数
├── wrangler.toml         # Cloudflare 配置
└── README.md             # 项目文档
```

### B. 相关资源
- Cloudflare Workers 文档
- VLESS 协议规范
- SOCKS5 协议 RFC 1928
- WebSocket 协议 RFC 6455
- https://tzang.net/cloudflare-worker-proxy-2/
- https://blog.fxcxy.com/2024/03/29/CloudflareWorker%E5%85%8D%E8%B4%B9%E6%90%AD%E5%BB%BA%E7%A7%91%E5%AD%A6%E4%B8%8A%E7%BD%91%E8%8A%82%E7%82%B9/
- 使用 cloudflare workers 实现科学上网

### C. 许可证
本项目仅供学习研究使用，请遵守当地法律法规。

本文档基于 Cloudflare Workers VLESS 代理服务源码编写，版本 1.0
