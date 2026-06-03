---
title: "C# Socket与网络编程深度解析"
tags:
  - csharp
  - socket
  - network
type: source
updated: 2026-06-02
source_files:
  - drafts/My_Vault/files/C sharp socket recieve and beginReceive.md
  - drafts/My_Vault/files/Unity Socket - 1.md
  - drafts/My_Vault/files/Socket-1.md
  - drafts/My_Vault/files/Socket-2 TCP 服务器.md
---

# C# Socket与网络编程深度解析

## 1. Socket 基础概念

Socket 是网络通信的端点，封装了 TCP/IP 协议栈的操作。核心流程：

```
服务端：socket() → bind() → listen() → accept() → read/write → close()
客户端：socket() → connect() → read/write → close()
```

### 1.1 创建 Socket

```csharp
// TCP Socket
var socket = new Socket(
    AddressFamily.InterNetwork,   // IPv4
    SocketType.Stream,            // TCP（面向连接、可靠、流式）
    ProtocolType.Tcp              // 明确 TCP
);

// UDP Socket
var udpSocket = new Socket(
    AddressFamily.InterNetwork,
    SocketType.Dgram,             // UDP（无连接、不可靠、数据报）
    ProtocolType.Udp
);
```

| SocketType | 协议 | 特征 |
|-----------|------|------|
| `Stream` | TCP | 面向连接、可靠有序、无数据边界 |
| `Dgram` | UDP | 无连接、可能丢失/乱序、有数据边界 |

### 1.2 地址绑定与监听

```csharp
// 服务端绑定
var endpoint = new IPEndPoint(IPAddress.Any, port: 10010);
socket.Bind(endpoint);
socket.Listen(backlog: 10);  // 等待队列最大长度

// 接受连接 —— 阻塞版本
Socket clientSocket = socket.Accept();

// 接受连接 —— 异步版本
socket.BeginAccept(AcceptCallback, socket);
```

---

## 2. TCP 三次握手与四次挥手

### 2.1 三次握手（建立连接）

```
客户端 → [SYN] seq=1000 → 服务端
客户端 ← [SYN+ACK] seq=2000, ack=1001 ← 服务端
客户端 → [ACK] seq=1001, ack=2001 → 服务端
```

### 2.2 四次挥手（断开连接）

```
主动方 → [FIN] seq=5000 → 被动方
主动方 ← [ACK] seq=7500, ack=5001 ← 被动方
主动方 ← [FIN] seq=7501, ack=5001 ← 被动方
主动方 → [ACK] seq=5001, ack=7502 → 被动方
```

### 2.3 数据传输确认

```
发送方 → [SEQ 1200 + 100 bytes data] → 接收方
发送方 ← [ACK 1301] ← 接收方       (确认收到，期望下一个 SEQ=1301)
```

---

## 3. 同步接收 vs 异步接收

### 3.1 同步 Receive —— 阻塞调用

```csharp
Socket client = socket.Accept();  // 阻塞直到有连接
byte[] buffer = new byte[1024];
int received = client.Receive(buffer);  // 阻塞直到收到数据
string message = Encoding.UTF8.GetString(buffer, 0, received);
```

**问题**：一个线程只能处理一个连接；连接数增加时需要大量线程。

### 3.2 异步 BeginReceive / EndReceive（APM）

```csharp
// 发起异步接收
socket.BeginReceive(buffer, 0, buffer.Length, SocketFlags.None,
    out SocketError errorCode, ReceiveCallback, socket);

// 回调处理
private void ReceiveCallback(IAsyncResult ar)
{
    Socket socket = (Socket)ar.AsyncState;
    int received = socket.EndReceive(ar, out SocketError errorCode);

    if (received == 0)
    {
        // 对方关闭连接
        socket.Close();
        return;
    }

    string message = Encoding.UTF8.GetString(buffer, 0, received);
    ProcessMessage(message);

    // 继续接收 —— 循环保持
    socket.BeginReceive(buffer, 0, buffer.Length, SocketFlags.None,
        out errorCode, ReceiveCallback, socket);
}
```

### 3.3 SocketAsyncEventArgs（高性能）

```csharp
// 更高效的异步模式 —— 避免 IAsyncResult 分配
var args = new SocketAsyncEventArgs();
args.SetBuffer(new byte[1024], 0, 1024);
args.Completed += OnReceiveCompleted;

// 发起接收
if (!socket.ReceiveAsync(args))
{
    // 同步完成（罕见）
    ProcessReceive(args);
}

private void OnReceiveCompleted(object sender, SocketAsyncEventArgs e)
{
    if (e.SocketError == SocketError.Success)
    {
        ProcessReceive(e);
        // 继续接收
        e.AcceptSocket.ReceiveAsync(e);
    }
}
```

| 方式 | 特点 | 适用场景 |
|------|------|----------|
| `Receive` (同步) | 阻塞线程 | 简单场景，连接数极少 |
| `BeginReceive` (APM) | 异步，IAsyncResult 有分配 | 中等连接数 |
| `SocketAsyncEventArgs` | 零分配异步，性能最佳 | 高并发服务器 |
| `ReceiveAsync` (TAP) | 现代 async/await | 推荐新项目使用 |

---

## 4. TCP 数据边界问题

TCP 是**流式协议**，无消息边界。一次 `Send` 的数据可能被拆成多次 `Receive`，多次 `Send` 也可能被合并成一次 `Receive`。

### 4.1 长度前缀方案

```csharp
// 发送：先发长度（4字节），再发数据
public async Task SendMessageAsync(Socket socket, byte[] data)
{
    byte[] lengthPrefix = BitConverter.GetBytes(data.Length);
    await socket.SendAsync(lengthPrefix, SocketFlags.None);
    await socket.SendAsync(data, SocketFlags.None);
}

// 接收：先读长度，再循环读到足够字节
public async Task<byte[]> ReceiveMessageAsync(Socket socket)
{
    byte[] lengthBuffer = new byte[4];
    await ReceiveExactAsync(socket, lengthBuffer, 4);
    int length = BitConverter.ToInt32(lengthBuffer, 0);

    byte[] data = new byte[length];
    await ReceiveExactAsync(socket, data, length);
    return data;
}

// 确保读取指定字节数
private async Task ReceiveExactAsync(Socket socket, byte[] buffer, int totalBytes)
{
    int received = 0;
    while (received < totalBytes)
    {
        int count = await socket.ReceiveAsync(
            new ArraySegment<byte>(buffer, received, totalBytes - received),
            SocketFlags.None);
        if (count == 0) throw new SocketException();  // 连接关闭
        received += count;
    }
}
```

### 4.2 分隔符方案

```csharp
// 使用特定分隔符（如 \n）标记消息边界
// 缺点：消息体不能包含分隔符；需要缓冲+扫描
```

---

## 5. 完整的异步 TCP 服务器示例

```csharp
public class AsyncTcpServer
{
    private Socket _listenSocket;
    private readonly List<ClientInfo> _clients = new();

    public void Start(int port)
    {
        _listenSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        _listenSocket.Bind(new IPEndPoint(IPAddress.Any, port));
        _listenSocket.Listen(10);

        _listenSocket.BeginAccept(AcceptCallback, _listenSocket);
        Console.WriteLine($"Server started on port {port}");
    }

    private void AcceptCallback(IAsyncResult ar)
    {
        try
        {
            Socket listener = (Socket)ar.AsyncState;
            Socket client = listener.EndAccept(ar);

            var clientInfo = new ClientInfo(client);
            lock (_clients) { _clients.Add(clientInfo); }

            // 开始接收此客户端数据
            client.BeginReceive(clientInfo.Buffer, 0, clientInfo.Buffer.Length,
                SocketFlags.None, ReceiveCallback, clientInfo);

            // 继续接受新连接
            listener.BeginAccept(AcceptCallback, listener);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Accept error: {ex.Message}");
        }
    }

    private void ReceiveCallback(IAsyncResult ar)
    {
        var info = (ClientInfo)ar.AsyncState;
        try
        {
            int count = info.Socket.EndReceive(ar);
            if (count == 0)
            {
                DisconnectClient(info);
                return;
            }

            string message = Encoding.UTF8.GetString(info.Buffer, 0, count);
            Console.WriteLine($"Received: {message}");

            // 广播给所有客户端
            Broadcast(message, excludeSocket: info.Socket);

            // 继续接收
            info.Socket.BeginReceive(info.Buffer, 0, info.Buffer.Length,
                SocketFlags.None, ReceiveCallback, info);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Receive error: {ex.Message}");
            DisconnectClient(info);
        }
    }

    private void Broadcast(string message, Socket excludeSocket)
    {
        byte[] data = Encoding.UTF8.GetBytes(message);
        ClientInfo[] clients;
        lock (_clients) { clients = _clients.ToArray(); }

        foreach (var client in clients)
        {
            if (client.Socket != excludeSocket)
            {
                try { client.Socket.Send(data); }
                catch { DisconnectClient(client); }
            }
        }
    }

    private void DisconnectClient(ClientInfo info)
    {
        try { info.Socket.Close(); } catch { }
        lock (_clients) { _clients.Remove(info); }
        Console.WriteLine("Client disconnected");
    }
}

public class ClientInfo
{
    public Socket Socket { get; }
    public byte[] Buffer { get; } = new byte[1024];

    public ClientInfo(Socket socket)
    {
        Socket = socket;
    }
}
```

---

## 6. 自定义应用层协议

当 TCP 流无法满足消息边界需求时，定义应用层协议：

```csharp
// 协议：第一个字节 = 操作数个数，接着每个操作数 4 字节，最后 1 字节运算符
// 例如：2 + 3 → [0x02] [0x00,0x00,0x00,0x02] [0x00,0x00,0x00,0x03] [0x2B]

[Serializable]
public class ProtoBase
{
    public string Name { get; set; }
}

[Serializable]
public class PlayProto : ProtoBase
{
    public int X { get; set; }
    public int Y { get; set; }
    public int Color { get; set; }
}

// 序列化为 JSON/Binary 后加上长度前缀发送
```

---

## 7. Unity 主线程调度

Socket 回调在**工作线程**执行，不能直接访问 Unity 对象（GameObject、Transform 等）：

```csharp
// 方案：消息队列 + Update 轮询
public class NetworkManager : MonoBehaviour
{
    private readonly Queue<Action> _mainThreadActions = new();

    // Socket 回调中入队
    private void ReceiveCallback(IAsyncResult ar)
    {
        // ... 解析消息 ...
        string message = Encoding.UTF8.GetString(buffer, 0, count);

        // 将 UI 更新操作入队
        lock (_mainThreadActions)
        {
            _mainThreadActions.Enqueue(() =>
            {
                uiText.text += "\n" + message;
            });
        }
    }

    // Update 在主线程执行
    private void Update()
    {
        lock (_mainThreadActions)
        {
            while (_mainThreadActions.Count > 0)
                _mainThreadActions.Dequeue()?.Invoke();
        }
    }
}
```

---

## 8. 面试要点

1. **TCP vs UDP**：TCP 面向连接/可靠/流式；UDP 无连接/不可靠/数据报
2. **三次握手**：SYN → SYN+ACK → ACK，用于建立可靠连接
3. **TCP 粘包/拆包**：TCP 无消息边界，需自定义协议（长度前缀/分隔符/固定长度）
4. **同步 vs 异步**：同步阻塞线程；BeginReceive 基于回调；SocketAsyncEventArgs 零分配；推荐 ReceiveAsync(TAP)
5. **recv=0 的含义**：对方优雅关闭了连接（发送了 FIN）
6. **自定义协议**：序列化消息 → 加长度前缀 → 发送；接收时按长度循环读到完整消息

## 9. 交叉引用

- [[csharp-async-awaiter-摘要|C# 异步模型]] — Socket 的 TAP 模式（ReceiveAsync）与 CancellationToken 取消
- [[csharp-serialization-摘要|C# 序列化与IO]] — NetworkStream 与自定义协议消息的序列化
- [[Unity-Socket网络编程]] — Unity 端 Socket 实践与粘包处理方案
