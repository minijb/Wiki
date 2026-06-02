---
title: "Unity Socket 网络编程 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/networking/unity-socket-tcp-udp.md"
tags: [unity, networking, socket, tcp, udp]
---

# Unity Socket 网络编程 — 摘要

来源：`raw/gamedev/networking/unity-socket-tcp-udp.md`

## 概述

从底层 Socket 原理出发，贯穿 TCP/UDP 协议细节、Unity 异步实践、粘包处理与高层框架选型，构成完整的 Unity 网络编程知识体系。核心使用 C# `System.Net.Sockets` 命名空间，无需依赖 Unity 高层网络框架。

## 要点

1. **Socket 创建三参数**：协议族（`AddressFamily.InterNetwork` 对应 IPv4）、套接字类型（`SocketType.Stream`=TCP 流式 / `SocketType.Dgram`=UDP 数据报）、协议（`ProtocolType.Tcp` / `ProtocolType.Udp`）
2. **地址与字节序**：C# 使用 `IPEndPoint` 描述地址 + 端口；网络字节序为大端序（Big-Endian），与 x86/x64 小端序相反，跨平台需用 `BinaryPrimitives.ReadInt16BigEndian` 等方法转换
3. **TCP 核心流程**：服务端 `Bind → Listen → (Accept → Read/Write) × N → Close`，客户端 `Connect → Read/Write → Close`。`accept()` 返回新的通信套接字，与监听套接字独立
4. **TCP 流式无边界**（核心难点）：多次 `write` 可被一次 `read` 收到（粘包），一次 `write` 可被多次 `read` 拆开（拆包）。必须通过应用层协议定义消息边界——长度前缀（最通用）、分隔符、或固定长度
5. **三次握手与四次挥手**：SYN → SYN+ACK → ACK 建立连接；四次挥手因为 TCP 支持半关闭（一方发送 FIN 后仍可接收对方剩余数据）
6. **UDP 编程**：无连接，`sendto()`/`recvfrom()` 每次携带地址。C# 使用 `UdpClient` 封装。有数据边界，不存在粘包问题，但不可靠——数据可能丢失、重复或乱序。适用实时游戏状态同步、语音/视频流
7. **Unity 异步模式**：使用 APM 模式的 `BeginAccept`/`BeginReceive`（回调在 I/O 工作线程执行），不能直接操作 Unity API。通过 `Queue<Action>` + 主线程 `Update()` 出队执行实现线程安全
8. **自定义协议**：`[Serializable]` 基类 + `JsonUtility` 序列化 + `类型名|JSON` 分隔格式实现消息类型分发。生产环境推荐 Protobuf 或 MessagePack 二进制序列化
9. **粘包处理——长度前缀方案**：发送端在消息体前附加 2 字节（Int16）或 4 字节（Int32）长度；接收端维护累积缓冲区，递归解析完整消息后将剩余数据前移
10. **大端序最佳实践**：统一约定网络传输为大端序，发送前 `WriteInt16BigEndian`，接收后 `ReadInt16BigEndian`，避免手动判断 `BitConverter.IsLittleEndian`
11. **WebSocket**：客户端 [NativeWebSocket](https://github.com/endel/NativeWebSocket)（Unity），服务端 [Fleck](https://www.nuget.org/packages/Fleck)（C# 无 ASP.NET 依赖），序列化可选 JSON / Protobuf / MessagePack
12. **Mirror 高层框架**：`NetworkManager` 管理连接与场景，`[Command]`/`[ClientRpc]`/`[SyncVar]` 自动同步。底层默认 KCP over UDP（可靠 + 低延迟）。适合标准 Unity 多人游戏，裸 Socket 更适合非 Unity 客户端通信或极致性能场景

## 关联 Wiki 页面

- [[concepts/Unity-Socket网络编程|Unity Socket 网络编程]] — 概念综合页
- [[concepts/CSharp网络Socket|C# 网络 Socket]] — C# Socket 基础概念与三种接收模式
- [[concepts/CSharp异步模型|C# 异步模型]] — TAP 模式与 CancellationToken
