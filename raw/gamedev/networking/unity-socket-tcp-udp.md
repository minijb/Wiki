---
title: "Unity Socket 网络编程"
type: source
updated: 2026-06-02
tags: [unity, networking, socket, tcp, udp]
---

# Unity Socket 网络编程

Socket 是操作系统提供的网络通信端点抽象，封装了 TCP/IP 协议栈的底层操作。Unity 中通过 C# 的 `System.Net.Sockets` 命名空间可以直接使用 Socket API 实现自定义网络通信，无需依赖 Unity 高层网络框架（如 UNet、Mirror）。本文从底层 Socket 原理出发，贯穿 TCP/UDP 协议细节、Unity 异步实践、粘包处理与高层框架选型，构成完整的 Unity 网络编程知识体系。

## Socket 基础

### 套接字创建

Socket 通过 `socket()` 系统调用创建，三个参数决定其行为：

```c
int socket(int domain, int type, int protocol);
```

**第一个参数：协议族（Protocol Family）**。定义在 `<sys/socket.h>` 中。

| 名称 | 协议族 | 说明 |
|------|--------|------|
| `PF_INET` | IPv4 互联网协议族 | 最常用，对应 `AF_INET` |
| `PF_INET6` | IPv6 互联网协议族 | 下一代 IP 协议 |
| `PF_LOCAL` | 本地通信 Unix 协议族 | 同一主机进程间通信 |
| `PF_PACKET` | 底层套接字协议族 | 直接操作链路层帧 |
| `PF_IPX` | IPX Novel 协议族 | 遗留协议 |

> [!note] PF vs AF
> 历史上 `PF_*`（Protocol Family）和 `AF_*`（Address Family）曾是不同的枚举，但在现代实现中它们完全相同。C# 中使用 `AddressFamily.InterNetwork` 即对应 `AF_INET`/`PF_INET`。

**第二个参数：套接字类型。**

| 类型 | 行为特征 |
|------|----------|
| `SOCK_STREAM` | 面向连接（TCP）。数据不会消失、按序传输、**无数据边界（Boundary）** |
| `SOCK_DGRAM` | 面向消息（UDP）。强调快速传输、可能丢失或损毁、**有数据边界**、限制每次传输大小 |

**第三个参数：具体协议。** 通常传 `IPPROTO_TCP` 或 `IPPROTO_UDP`，也可传 `0` 让系统根据前两个参数自动选择。

C# 中的等效写法：

```csharp
// TCP Socket
var tcpSocket = new Socket(
    AddressFamily.InterNetwork,   // IPv4
    SocketType.Stream,            // 面向连接、可靠、流式
    ProtocolType.Tcp
);

// UDP Socket
var udpSocket = new Socket(
    AddressFamily.InterNetwork,
    SocketType.Dgram,             // 无连接、不可靠、数据报
    ProtocolType.Udp
);
```

### 地址与字节序

**sockaddr_in 结构体**（C 语言层面）描述一个 IPv4 地址 + 端口：

```c
struct sockaddr_in {
    sa_family_t    sin_family;  // 地址族（AF_INET）
    uint16_t       sin_port;    // 16 位端口号，网络字节序
    struct in_addr sin_addr;    // 32 位 IP 地址，网络字节序
    char           sin_zero[8]; // 填充，不使用
};

struct in_addr {
    in_addr_t s_addr;           // 32 位 IPv4 地址
};
```

C# 中使用 `IPEndPoint` 替代：

```csharp
var endpoint = new IPEndPoint(IPAddress.Any, 10010);
```

**字节序转换**是网络编程的关键概念。网络字节序采用大端序（Big-Endian），而主机可能采用小端序（Little-Endian，如 x86/x64）。

| 函数 | 含义 |
|------|------|
| `htons()` | Host to Network Short（16 位，用于端口号） |
| `ntohs()` | Network to Host Short |
| `htonl()` | Host to Network Long（32 位，用于 IP 地址） |
| `ntohl()` | Network to Host Long |

命名规则记忆：**h** = host（主机字节序）、**n** = network（网络字节序）、**s** = short（16 位端口）、**l** = long（32 位 IP）。

**IP 地址转换函数：**

```c
in_addr_t inet_addr(const char *string);   // 字符串 → 32 位大端整数
int inet_aton(const char *string, struct in_addr *addr);  // 同上，更常用
char *inet_ntoa(struct in_addr adr);       // 整数 → 字符串
```

**服务端地址初始化示例（C）：**

```c
struct sockaddr_in addr;
char *serv_ip = "211.217.168.13";
char *serv_port = "9190";
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = inet_addr(serv_ip);  // 字符串 IP → 网络字节序
addr.sin_port = htons(atoi(serv_port));     // 字符串端口 → 网络字节序
```

`INADDR_ANY` 定义为 `0.0.0.0`，表示服务端绑定到所有可用的网络接口。C# 中对应 `IPAddress.Any`。

### TCP vs UDP

| 特性 | TCP（`SOCK_STREAM`） | UDP（`SOCK_DGRAM`） |
|------|----------------------|----------------------|
| 连接方式 | 面向连接，需三次握手 | 无连接，直接发送 |
| 可靠性 | 可靠，确认 + 重传 | 不可靠，可能丢失或乱序 |
| 数据边界 | 无边界（流式），需应用层分割 | 有边界（数据报），每次一个完整消息 |
| 传输顺序 | 严格有序 | 不保证顺序 |
| 传输大小 | 无限制（流式） | 单次有大小上限 |
| 速度 | 较慢（有握手和确认开销） | 较快 |
| 适用场景 | 文件传输、网页、聊天、数据库 | 实时游戏状态同步、视频流、DNS |

## TCP 服务器与客户端

### 函数调用流程

TCP 通信的核心流程固定：

```
服务端：socket() → bind() → listen() → accept() → read/write → close()
客户端：socket() → connect() → read/write → close()
```

**listen()** —— 将套接字转为可接受连接状态，同时指定等待队列长度：

```c
int listen(int sockfd, int backlog);
// backlog: 连接请求等待队列的最大长度
// 内核为每个监听套接字维护两个队列：
//   1. 未完成握手队列（SYN_RCVD 状态）
//   2. 已完成握手队列（ESTABLISHED 状态）
// backlog 约等于两者之和的上限
```

**accept()** —— 受理客户端连接请求，返回**新的套接字文件描述符**：

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// 返回值：新的客户端套接字 fd（用于与该客户端通信）
// 原监听套接字 sockfd 继续用于接受新连接
```

> [!warning] 关键区分
> `accept()` 返回的套接字**不等于服务端监听套接字**。监听套接字只负责接受新连接，每个客户端拥有独立的通信套接字，各自需要 `close()` 关闭。

**connect()** —— 客户端发起连接：

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

### 回声服务器实现

以下是一个完整的 **TCP 回声服务器**（C++），服务器接收客户端发来的字符串后原样返回：

```c
// 服务端
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 1024

int main(int argc, char *argv[]) {
    int serv_sock, clnt_sock;
    char message[BUF_SIZE];
    int str_len, i;
    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t clnt_adr_sz;

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));
    listen(serv_sock, 5);

    clnt_adr_sz = sizeof(clnt_adr);
    for (i = 0; i < 5; i++) {  // 依次服务 5 个客户端
        clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
        // 循环读取直到客户端关闭（read 返回 0）
        while ((str_len = read(clnt_sock, message, BUF_SIZE)) != 0)
            write(clnt_sock, message, str_len);  // 原样返回
        close(clnt_sock);
    }
    close(serv_sock);
    return 0;
}
```

对应客户端（回声模式：发送一行，等待服务器返回后再发送下一行）：

```c
// 客户端
int main(int argc, char *argv[]) {
    int sock;
    char message[BUF_SIZE];
    int str_len;
    struct sockaddr_in serv_adr;

    sock = socket(PF_INET, SOCK_STREAM, 0);
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_adr.sin_port = htons(atoi(argv[2]));

    connect(sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));

    while (1) {
        fputs("Input message(Q to quit): ", stdout);
        fgets(message, BUF_SIZE, stdin);
        if (!strcmp(message, "q\n") || !strcmp(message, "Q\n")) break;
        write(sock, message, strlen(message));
        str_len = read(sock, message, BUF_SIZE - 1);
        message[str_len] = 0;
        printf("Message from server: %s", message);
    }
    close(sock);
    return 0;
}
```

### TCP 无边界问题

TCP 是**流式协议**，不存在数据边界。具体表现为：

1. **多次 `write()` 的数据可能被合并成一次 `read()` 收到**（粘包）
2. **一次 `write()` 的数据可能被拆成多次 `read()` 收到**（拆包）
3. 原因：TCP 只保证字节流的顺序和可靠性，不保留应用层的消息边界

对于回声服务器，由于客户端**知道自己发送了多少字节**，可以用循环读取直到收齐等量数据：

```c
str_len = write(sock, message, strlen(message));
recv_len = 0;
while (recv_len < str_len) {  // 循环读直到达到发送量
    recv_cnt = read(sock, &message[recv_len], BUF_SIZE - 1);
    if (recv_cnt == -1) error_handling("read() error");
    recv_len += recv_cnt;
}
message[recv_len] = 0;
```

### 应用层协议设计

当通信不再是简单的回声（客户端不知道期望接收多少字节），就需要**应用层协议**来定义消息边界。常见方案：

1. **长度前缀（Length-Prefix）**：消息头携带消息体长度
2. **分隔符（Delimiter）**：用特殊字符（如 `\n`、`\0`）标记消息结束
3. **固定长度（Fixed-Length）**：所有消息长度固定

以下是一个**操作数计数协议**的示例：第一个字节表示操作数个数，后续每个操作数占 4 字节，最后 1 字节为运算符。

发送端（客户端）：

```c
fputs("Operand count: ", stdout);
scanf("%d", &opnd_cnt);
opmsg[0] = (char)opnd_cnt;
for (i = 0; i < opnd_cnt; i++) {
    printf("Operand %d: ", i + 1);
    scanf("%d", (int*)&opmsg[i * OPSZ + 1]);
}
fputs("Operator: ", stdout);
scanf("%c", &opmsg[opnd_cnt * OPSZ + 1]);
write(sock, opmsg, opnd_cnt * OPSZ + 2);  // 发送：1 字节计数 + N×4 字节操作数 + 1 字节运算符
read(sock, &result, RLT_SIZE);
```

接收端（服务端）：

```c
read(clnt_sock, &opnd_cnt, 1);  // 先读第一个字节：操作数个数
recv_len = 0;
while ((opnd_cnt * OPSZ + 1) > recv_len) {  // 已知期望字节数，循环读完
    recv_cnt = read(clnt_sock, &opinfo[recv_len], BUF_SIZE - 1);
    recv_len += recv_cnt;
}
result = calculate(opnd_cnt, (int*)opinfo, opinfo[recv_len - 1]);
write(clnt_sock, (char*)&result, sizeof(result));
```

C# 中更推荐用 `[Serializable]` 标记协议类 + JSON/Protobuf 序列化 + 长度前缀打包的方式。详见后文「粘包处理」与「自定义协议」章节。

### 三次握手与四次挥手

**三次握手（建立连接）：**

```
客户端 → [SYN]      seq=1000, ack=-           → 服务端
客户端 ← [SYN+ACK]  seq=2000, ack=1001        ← 服务端
客户端 → [ACK]      seq=1001, ack=2001        → 服务端
```

SYN 是同步序列号（Synchronize Sequence Numbers）。握手结束后，双方都确认了对方的收发能力，进入 `ESTABLISHED` 状态。

**数据传输确认机制：**

```
发送方 → [SEQ=1200 + 100 bytes data] → 接收方
发送方 ← [ACK=1301]                  ← 接收方   (确认收到，期望下一个 SEQ=1301)
```

若发送方未在超时内收到 ACK，会重传数据。ACK 值 = 已收到的连续字节数 + 1（即期望的下一个 SEQ）。

**四次挥手（断开连接）：**

```
主动方 → [FIN]  seq=5000, ack=-      → 被动方
主动方 ← [ACK]  seq=7500, ack=5001   ← 被动方   (半关闭：被动方可能还有数据要发)
主动方 ← [FIN]  seq=7501, ack=5001   ← 被动方   (被动方也准备关闭)
主动方 → [ACK]  seq=5001, ack=7502   → 被动方   (确认对方关闭)
```

之所以是四次而非三次，是因为 TCP 支持**半关闭（Half-Close）**：一方发送 FIN 后不再发送数据，但仍可接收对方尚未发完的数据。

## UDP 编程

UDP 使用 `sendto()` 和 `recvfrom()` 两个函数，每次调用都需要携带/接收目标地址信息：

```c
ssize_t sendto(int sock, void *buff, size_t nbytes, int flags,
               struct sockaddr *to, socklen_t addrlen);
// to: 目标地址

ssize_t recvfrom(int sock, void *buff, size_t nbytes, int flags,
                 struct sockaddr *from, socklen_t *addrlen);
// from: 输出参数，保存发送端地址信息
```

UDP 的关键特性：

- **无连接**：不需要 `connect()` 和 `accept()`，每个数据报独立寻址
- **有数据边界**：一次 `sendto()` 对应一次 `recvfrom()`（如果数据报未超过缓冲区），不存在粘包问题
- **客户端 `bind()` 可选**：首次调用 `sendto()` 时系统会自动分配 IP 和端口，且该地址保持到程序结束
- **不可靠**：数据可能丢失、重复或乱序，应用层需要自行处理可靠性（如果需要）
- **大小限制**：单个 UDP 数据报通常不超过 64KB（实际路径 MTU 限制更小，约 1500 字节以内避免 IP 分片）

C# 中 UDP 的基本使用：

```csharp
// 服务端
var udpServer = new UdpClient(port);
IPEndPoint remoteEndPoint = new IPEndPoint(IPAddress.Any, 0);
byte[] data = udpServer.Receive(ref remoteEndPoint);  // 阻塞直到收到数据报
// remoteEndPoint 被填充为发送方地址

// 客户端
var udpClient = new UdpClient();
udpClient.Send(data, data.Length, serverEndPoint);
```

UDP 适用于对实时性要求高、允许偶尔丢包的场景：游戏状态同步（位置、朝向、动作）、语音/视频流、DNS 查询。

## Unity 中的 Socket 实践

### 异步聊天室

Unity 中使用 C# 的 APM（Asynchronous Programming Model）模式——`BeginAccept`/`EndAccept` 和 `BeginReceive`/`EndReceive`——实现非阻塞的网络 I/O。

**服务端核心结构：**

```csharp
public static Socket socket;
public static List<ClientInfo> clients = new List<ClientInfo>();

public class ClientInfo {
    public Socket socket;
    public byte[] buffer = new byte[1024];
    public ClientInfo(Socket socket) { this.socket = socket; }
}
```

**启动服务端：**

```csharp
public static void Main(string[] argv) {
    socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
    socket.Bind(new IPEndPoint(IPAddress.Any, 10010));
    socket.Listen(0);
    socket.BeginAccept(AcceptCallback, socket);  // 异步 accept
    Console.ReadLine();  // 防止主线程退出
}
```

**AcceptCallback** —— 接受新连接并启动接收循环：

```csharp
public static void AcceptCallback(IAsyncResult result) {
    try {
        Socket socket = (Socket)result.AsyncState;
        Socket client = socket.EndAccept(result);
        ClientInfo clientInfo = new ClientInfo(client);
        clients.Add(clientInfo);
        // 开始异步接收该客户端数据
        client.BeginReceive(clientInfo.buffer, 0, 1024, 0, ReceiveCallback, clientInfo);
        // 继续接受新连接
        socket.BeginAccept(AcceptCallback, socket);
    } catch (Exception e) {
        Console.WriteLine(e.ToString());
    }
}
```

**ReceiveCallback** —— 收到数据后广播给所有客户端：

```csharp
public static void ReceiveCallback(IAsyncResult result) {
    try {
        ClientInfo info = (ClientInfo)result.AsyncState;
        int count = info.socket.EndReceive(result);
        if (count == 0) {  // 客户端关闭连接
            info.socket.Close();
            clients.Remove(info);
            Console.WriteLine("one client closed");
            return;
        }
        // 广播给所有客户端
        foreach (var c in clients)
            c.socket.Send(info.buffer, 0, count, 0);
        // 继续接收
        info.socket.BeginReceive(info.buffer, 0, 1024, 0, ReceiveCallback, info);
    } catch (Exception e) {
        Console.WriteLine(e.ToString());
    }
}
```

**客户端连接与发送：**

```csharp
public class NetworkManager : MonoBehaviour {
    public static Socket socket;
    public static byte[] buffer = new byte[1024];

    public static void Connect() {
        socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        socket.Connect(new IPEndPoint(IPAddress.Parse("116.198.200.12"), 10010));
        socket.BeginReceive(buffer, 0, 1024, 0, ReceiveCallback, socket);
    }

    public static void Send(string message) {
        socket.Send(System.Text.Encoding.UTF8.GetBytes(message));
    }
}
```

### 线程安全与主线程调度

`BeginReceive` 的回调在**工作线程（I/O 完成端口线程）**上执行，不能直接操作 Unity 的 GameObject、Transform、UI 等对象（Unity API 非线程安全）。

**解决方案：Action 队列模式**——在工作线程中将操作入队，在主线程 `Update()` 中出队执行：

```csharp
public class GameManager : MonoBehaviour {
    public Queue<Action> actions = new Queue<Action>();

    private void Update() {
        lock (actions) {
            while (actions.Count > 0)
                actions.Dequeue()?.Invoke();
        }
    }
}

// Socket 回调中使用：
public static void ReceiveCallback(IAsyncResult result) {
    Socket socket = (Socket)result.AsyncState;
    int count = socket.EndReceive(result);
    // 将 UI 更新入队到主线程
    GameManager.instance.actions.Enqueue(() => {
        UIManager.instance.contentText.text += "\n" + "Server:" +
            System.Text.Encoding.UTF8.GetString(buffer, 0, count);
    });
    socket.BeginReceive(buffer, 0, 1024, 0, ReceiveCallback, socket);
}
```

> [!tip] 现代替代方案
> 在 .NET 4.5+ / Unity 2020+ 中推荐使用 `ReceiveAsync`（TAP 模式）配合 `async`/`await`。但 Apollo 模式的 `BeginReceive` 回调链在 Unity 老项目中仍广泛存在，且避免了 `async` 状态机的分配开销。

### 自定义协议

当通信场景超越简单文本广播时，需要自定义协议来区分消息类型。以下是五子棋对战的协议设计：

**协议基类与派生类型：**

```csharp
[Serializable]
public class ProtoBase {
    public string name;
}

[Serializable]
public class PlayProto : ProtoBase {   // 落子
    public int x;
    public int y;
    public int color;
}

[Serializable]
public class ReadyProto : ProtoBase { }  // 准备就绪

[Serializable]
public class ColorProto : ProtoBase {    // 分配颜色
    public int color;
}

[Serializable]
public class MessageProto : ProtoBase {  // 文本消息
    public string message;
}

[Serializable]
public class WinProto : ProtoBase {      // 胜负结果
    public int color;
}
```

**协议编解码器（Encoder / Decoder）：**

```csharp
// 编码：协议类型名 + "|" + JSON
private static string Encode(object o) {
    return JsonUtility.ToJson(o);
}

public static void Send(ProtoBase proto) {
    socket.Send(Encoding.UTF8.GetBytes(proto.name + "|" + Encode(proto)));
}

// 解码：根据类型名字段分发
private static ProtoBase Decode(string str) {
    var args = str.Split('|');
    return args[0] switch {
        "color"   => JsonUtility.FromJson<ColorProto>(args[1]),
        "play"    => JsonUtility.FromJson<PlayProto>(args[1]),
        "message" => JsonUtility.FromJson<MessageProto>(args[1]),
        "ready"   => JsonUtility.FromJson<ReadyProto>(args[1]),
        "win"     => JsonUtility.FromJson<WinProto>(args[1]),
        _         => null,
    };
}
```

**服务端接收回调中的协议分发：**

```csharp
public static void ReceiveCallback(IAsyncResult result) {
    Socket socket = (Socket)result.AsyncState;
    int count = socket.EndReceive(result);
    string content = Encoding.UTF8.GetString(buffer, 0, count);
    ProtoBase proto = Decode(content);

    if (proto is MessageProto)
        GameManager.Instance.ReceiveMessage(proto as MessageProto);
    else if (proto is ColorProto)
        GameManager.Instance.ReceiveColor(proto as ColorProto);
    else if (proto is PlayProto)
        GameManager.Instance.ReceivePlay(proto as PlayProto);
    else if (proto is ReadyProto)
        GameManager.Instance.ReceiveReady(proto as ReadyProto);
    else if (proto is WinProto)
        GameManager.Instance.ReceiveWin(proto as WinProto);

    socket.BeginReceive(buffer, 0, 1024, 0, ReceiveCallback, socket);
}
```

> [!warning] 文本协议 vs 二进制协议
> 上述 `name|JSON` 格式属于**文本协议**，人类可读但带宽效率低。生产环境推荐使用 Protobuf 或 MessagePack 等**二进制序列化**方案，配合长度前缀解决粘包问题。

## 粘包处理

### 长度前缀方案

由于 TCP 无边界，客户端可能一次收到多个消息的拼接体（粘包）或一个消息的片段（拆包）。最通用的解决方案是**长度前缀**：每个消息前附加固定字节数（如 2 字节 `Int16` 或 4 字节 `Int32`）表示消息体长度。

**发送端——在消息体前拼上长度：**

```csharp
byte[] bodyBytes = Encoding.UTF8.GetBytes(jsonMessage);
byte[] lengthBytes = BitConverter.GetBytes((Int16)bodyBytes.Length);
byte[] toSend = lengthBytes.Concat(bodyBytes).ToArray();
socket.Send(toSend);
```

**接收端——累积缓冲区 + 递归解析：**

```csharp
int bufferCount;  // 当前缓冲区中已累积的字节数

public void DealData(ClientInfo info) {
    if (bufferCount <= 2) return;  // 连长度前缀都没收齐
    Int16 len = BitConverter.ToInt16(info.buffer, 0);       // 读取长度
    if (bufferCount < 2 + len) return;                      // 消息体未收齐

    // 提取完整消息（buffer[2] ~ buffer[2+len-1]）
    byte[] messageBody = new byte[len];
    Array.Copy(info.buffer, 2, messageBody, 0, len);
    ProcessMessage(messageBody);

    // 将剩余数据移到缓冲区头部
    int start = 2 + len;
    int count = bufferCount - start;
    Array.Copy(info.buffer, start, info.buffer, 0, count);
    bufferCount -= start;

    // 递归处理下一条消息
    DealData(info);
}
```

**`BeginReceive` 时的偏移设置：**

```csharp
// offset = bufferCount，把新数据接到缓冲区已有数据之后
client.BeginReceive(info.buffer, bufferCount, 1024 - bufferCount, 0, ReceiveCallback, info);
```

```csharp
// ReceiveCallback 中：
void ReceiveCallback(IAsyncResult result) {
    ClientInfo info = (ClientInfo)result.AsyncState;
    int count = info.socket.EndReceive(result);
    if (count == 0) { /* 断开处理 */ return; }
    bufferCount += count;
    DealData(info);
    // 继续接收
    info.socket.BeginReceive(info.buffer, bufferCount, 1024 - bufferCount, 0, ReceiveCallback, info);
}
```

> [!note] 长度字段的位数选择
> - `Int16`（2 字节）：最大消息体 32767 字节（约 32KB），适合聊天、简单状态同步
> - `Int32`（4 字节）：最大消息体约 2GB，适合文件传输、大JSON
> - `Int32` + 变长编码（如 Protobuf Varint）：兼顾紧凑与容量

### 大小端处理

`BitConverter` 使用**主机字节序**，而网络传输约定使用**大端序（Big-Endian）**。在跨平台场景（如 x86 服务器 ↔ ARM 移动端）中可能不一致：

```csharp
// 检查并处理
if (!BitConverter.IsLittleEndian) {
    Array.Reverse(lengthBytes);
}
```

手动字节交换（不依赖 `BitConverter.IsLittleEndian`）：

```csharp
byte[] n = new byte[2];
if (!BitConverter.IsLittleEndian) {
    n[0] = buffer[1];
    n[1] = buffer[0];
    Int16 len = BitConverter.ToInt16(n, 0);
}
```

> [!tip] 最佳实践
> 统一约定网络传输为大端序（Network Byte Order），所有平台在发送前将多字节整数转为大端，接收后转回本机字节序。C# 中可使用 `System.Buffers.Binary.BinaryPrimitives` 的 `WriteInt16BigEndian` / `ReadInt16BigEndian` 等方法（.NET Core 2.1+），避免手动判断和交换。

## WebSocket 与高层框架

### WebSocket 库

当项目需要浏览器支持或希望使用标准化的 Web 通信协议时，可以考虑 WebSocket 替代裸 Socket：

**客户端（Unity）：**

- [NativeWebSocket](https://github.com/endel/NativeWebSocket) — 轻量级 Unity WebSocket 客户端
- 安装：Package Manager → Add from Git URL → `https://github.com/endel/NativeWebSocket.git#upm`

**服务端（C#）：**

- [Fleck](https://www.nuget.org/packages/Fleck) — 轻量级 C# WebSocket 服务端，无需 ASP.NET
- 安装：`dotnet add package Fleck`

**序列化方案：**

- **JSON**：简单、人类可读、Unity 内置 `JsonUtility`，适合中小型消息
- **Protobuf**：Google 的二进制序列化协议，压缩率高、解析快、有 `.proto` schema，适合高性能场景
- **MessagePack**：类 JSON 的二进制格式，比 Protobuf 灵活，C# 支持良好

### Mirror 框架

[Mirror](https://mirror-networking.com/) 是 Unity 社区最流行的**高层网络框架**，继承自已弃用的 UNet（HLAPI），提供开箱即用的网络同步。

核心概念：

- **`NetworkManager`**：连接管理、场景切换、Player 生成
- **`[Command]` / `[ClientRpc]` / `[TargetRpc]`**：RPC 远程过程调用
- **`[SyncVar]`**：自动同步字段（服务端 → 客户端）
- **`NetworkTransform`**：自动同步 Transform 位置/旋转
- **`NetworkBehaviour`**：替代 `MonoBehaviour` 的网络行为基类

对比裸 Socket 与 Mirror：

| 维度 | 裸 Socket | Mirror |
|------|-----------|--------|
| 控制粒度 | 完全控制每个字节 | 框架自动处理同步 |
| 学习曲线 | 高（需理解 TCP/UDP、粘包、序列化） | 中（理解框架概念即可） |
| 适用场景 | 自定义协议、高性能、非 Unity 客户端 | 标准 Unity 多人游戏 |
| 传输层 | 直接 TCP/UDP | 默认 KCP over UDP（可靠+低延迟） |
| 序列化 | 自定义 | 内置 Weaved 代码生成 |
| 服务器架构 | 独立 .NET 进程 | Headless Unity Build 或独立 Server |

> [!note] Mirror 学习资源
> 参考 [知乎专栏：Mirror 网络框架入门](https://zhuanlan.zhihu.com/p/403184517) 了解基本用法与 Demo 搭建。

对于大多数 Unity 多人游戏项目，Mirror 是比裸 Socket 更高效的选择——它解决了同步、序列化、对象生成等通用问题，让开发者专注于游戏逻辑。裸 Socket 更适合以下场景：与非 Unity 客户端通信、自定义二进制协议、极致性能优化、或在 Mirror 不足以表达业务需求时作为底层补充。
