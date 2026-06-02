---
title: "Unity Socket 网络编程"
type: concept
updated: 2026-05-11
tags: [unity, networking, socket, tcp, udp]
aliases: [Unity Socket, Unity网络编程, Unity TCP UDP]
---

# Unity Socket 网络编程

Unity 中使用 C# `System.Net.Sockets` 命名空间直接操作 Socket API 实现自定义网络通信。核心挑战：TCP 无边界粘包拆包、跨平台字节序、工作线程回调与主线程调度。

## Socket 基础

Socket 是操作系统提供的网络通信端点抽象，通过 `socket()` 系统调用创建，三个参数决定其行为：

| 参数 | C# 枚举 | 含义 |
|------|---------|------|
| 协议族 | `AddressFamily.InterNetwork` | IPv4 互联网协议族 |
| 套接字类型 | `SocketType.Stream` / `SocketType.Dgram` | 面向连接流式（TCP）/ 无连接数据报（UDP） |
| 协议 | `ProtocolType.Tcp` / `ProtocolType.Udp` | 具体传输层协议 |

**地址表示**：C# 使用 `IPEndPoint` 替代 C 的 `sockaddr_in`：

```csharp
var endpoint = new IPEndPoint(IPAddress.Any, 10010);
```

**字节序**：网络字节序采用大端序（Big-Endian），x86/x64 主机为小端序。跨平台通信需要转换，推荐使用 `System.Buffers.Binary.BinaryPrimitives` 的 `ReadInt16BigEndian` / `WriteInt16BigEndian` 方法。

### TCP vs UDP

| 特性 | TCP（`SOCK_STREAM`） | UDP（`SOCK_DGRAM`） |
|------|----------------------|----------------------|
| 连接方式 | 面向连接，需三次握手 | 无连接，直接发送 |
| 可靠性 | 可靠，确认 + 重传 | 不可靠，可能丢失或乱序 |
| 数据边界 | **无边界**（流式），需应用层分割 | **有边界**（数据报），每次一个完整消息 |
| 传输顺序 | 严格有序 | 不保证顺序 |
| 适用场景 | 文件传输、聊天、数据库 | 实时游戏状态同步、视频流、DNS |

## TCP 编程

### 核心流程

```
服务端：socket() → bind() → listen() → accept() → read/write → close()
客户端：socket() → connect() → read/write → close()
```

**关键区分**：`accept()` 返回**新的客户端通信套接字**，与监听套接字独立——监听套接字继续接受新连接，每个客户端拥有独立的通信套接字和生命周期。

### 三次握手与四次挥手

**三次握手（建立连接）**：

```
客户端 → [SYN]  seq=1000     → 服务端
客户端 ← [SYN+ACK] seq=2000, ack=1001 ← 服务端
客户端 → [ACK] seq=1001, ack=2001 → 服务端
```

**四次挥手（断开连接）**：TCP 支持半关闭，一方发送 FIN 后不再发送数据但仍可接收对方剩余数据，因此断开需四次交互。

### TCP 无边界问题

TCP 是流式协议，不保留应用层消息边界。表现为：

1. **粘包**：多次 `write()` 的数据被合并成一次 `read()` 收到
2. **拆包**：一次 `write()` 的数据被拆成多次 `read()` 收到

**解决方案——应用层协议**：

| 方案 | 原理 | 适用场景 |
|------|------|----------|
| 长度前缀 | 消息头携带消息体长度（Int16/Int32） | 通用推荐 |
| 分隔符 | 特殊字符（`\n`、`\0`）标记消息结束 | 文本协议 |
| 固定长度 | 所有消息长度固定 | 简单控制协议 |

## UDP 编程

UDP 使用 `sendto()` / `recvfrom()` 两个函数，每次调用携带/接收目标地址。C# 封装：

```csharp
// 服务端
var udpServer = new UdpClient(port);
IPEndPoint remoteEndPoint = new IPEndPoint(IPAddress.Any, 0);
byte[] data = udpServer.Receive(ref remoteEndPoint);

// 客户端
var udpClient = new UdpClient();
udpClient.Send(data, data.Length, serverEndPoint);
```

核心特性：

- **无连接**：无需 `connect()` / `accept()`，每个数据报独立寻址
- **有数据边界**：不存在粘包问题，一次 `sendto()` 对应一次 `recvfrom()`
- **客户端 bind 可选**：首次 `sendto()` 时系统自动分配地址
- **大小限制**：单个数据报通常不超过 64KB，建议控制在 1500 字节以内避免 IP 分片
- **不可靠**：数据可能丢失、重复或乱序——适用于对实时性要求高、允许偶尔丢包的场景

## Unity 异步实践

### APM 回调模式

Unity 传统方案使用 `BeginAccept` / `BeginReceive`（APM 异步编程模型）：

```csharp
// 异步 Accept
socket.BeginAccept(AcceptCallback, socket);

// AcceptCallback 中
Socket client = socket.EndAccept(result);
client.BeginReceive(buffer, 0, 1024, 0, ReceiveCallback, clientInfo);
socket.BeginAccept(AcceptCallback, socket);  // 继续接受新连接
```

### 线程安全与主线程调度

`BeginReceive` 回调在**工作线程（I/O 完成端口线程）**上执行，Unity API 非线程安全。通过 **Action 队列模式**解决：

```csharp
// 工作线程中入队
GameManager.instance.actions.Enqueue(() => {
    UIManager.instance.contentText.text = receivedMessage;
});

// 主线程 Update 中出队执行
private void Update() {
    lock (actions) {
        while (actions.Count > 0)
            actions.Dequeue()?.Invoke();
    }
}
```

> [!tip] 现代替代方案
> .NET 4.5+ / Unity 2020+ 推荐 `ReceiveAsync`（TAP 模式）配合 `async`/`await`，代码更简洁，但 APM 回调链在 Unity 老项目中仍广泛存在。

### 自定义协议

当通信场景超越简单文本时，通过 `[Serializable]` + `JsonUtility` 定义协议类族，以 `类型名|JSON` 格式实现消息类型分发。生产环境推荐 Protobuf 或 MessagePack 二进制序列化替代 JSON 文本协议。

## 粘包处理

### 长度前缀方案（最通用）

**发送端**——消息体前附加长度（2 字节 Int16）：

```csharp
byte[] bodyBytes = Encoding.UTF8.GetBytes(jsonMessage);
byte[] lengthBytes = BitConverter.GetBytes((Int16)bodyBytes.Length);
byte[] toSend = lengthBytes.Concat(bodyBytes).ToArray();
socket.Send(toSend);
```

**接收端**——累积缓冲区 + 递归解析：

```csharp
void DealData(ClientInfo info) {
    if (bufferCount <= 2) return;                     // 长度前缀未收齐
    Int16 len = BitConverter.ToInt16(info.buffer, 0);
    if (bufferCount < 2 + len) return;                // 消息体未收齐

    byte[] messageBody = new byte[len];
    Array.Copy(info.buffer, 2, messageBody, 0, len);
    ProcessMessage(messageBody);

    // 剩余数据前移，递归处理下一条
    int start = 2 + len;
    int count = bufferCount - start;
    Array.Copy(info.buffer, start, info.buffer, 0, count);
    bufferCount -= start;
    DealData(info);
}
```

**接收偏移设置**（关键）：`BeginReceive` 的 offset 参数设为 `bufferCount`，将新数据接到缓冲区已有数据之后，避免覆盖未处理的消息片段。

### 长度字段选择

| 类型 | 最大消息体 | 适用场景 |
|------|-----------|----------|
| Int16（2 字节） | ~32KB | 聊天、简单状态同步 |
| Int32（4 字节） | ~2GB | 文件传输、大 JSON |
| Varint（变长） | 按需 | Protobuf 场景 |

### 大小端处理

`BitConverter` 使用主机字节序，跨平台必须统一为网络大端序：

```csharp
// 推荐（.NET Core 2.1+）
BinaryPrimitives.WriteInt16BigEndian(span, (short)bodyBytes.Length);
BinaryPrimitives.ReadInt16BigEndian(span);
```

## WebSocket

当需要浏览器支持或标准化 Web 通信协议时，WebSocket 是裸 Socket 的现代替代：

| 角色 | 推荐库 | 说明 |
|------|--------|------|
| Unity 客户端 | [NativeWebSocket](https://github.com/endel/NativeWebSocket) | 轻量级，支持 UPM |
| C# 服务端 | [Fleck](https://www.nuget.org/packages/Fleck) | 无需 ASP.NET，轻量部署 |
| 序列化 | JSON / Protobuf / MessagePack | 按性能需求选择 |

### Mirror 高层框架

[Mirror](https://mirror-networking.com/) 是 Unity 社区最流行的高层网络框架（继承自 UNet），底层默认 KCP over UDP（可靠 + 低延迟）：

| 特性 | 裸 Socket | Mirror |
|------|-----------|--------|
| 控制粒度 | 完全控制每个字节 | 框架自动处理同步 |
| 同步机制 | 手动序列化 + Send | `[SyncVar]` / `NetworkTransform` 自动同步 |
| RPC | 自定义协议分发 | `[Command]` / `[ClientRpc]` 声明式 |
| 适用场景 | 自定义协议、非 Unity 客户端 | 标准 Unity 多人游戏 |

裸 Socket 适合：非 Unity 客户端通信、自定义二进制协议、极致性能优化、Mirror 不足以表达业务需求时作为底层补充。

## 参见

- [[sources/Unity-Socket网络编程-摘要|Unity Socket 网络编程 — 来源摘要]]
- [[concepts/CSharp网络Socket|C# 网络 Socket]] — C# Socket 基础概念与三种接收模式对比
- [[concepts/CSharp异步模型|C# 异步模型]] — TAP 模式（ReceiveAsync）与 CancellationToken
- [[concepts/CSharp序列化IO|C# 序列化与 I/O]] — NetworkStream 与自定义协议序列化
