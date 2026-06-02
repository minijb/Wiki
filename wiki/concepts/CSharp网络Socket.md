---
title: "CSharp网络Socket"
tags:
  - csharp
  - socket
  - network
  - tcp
created: 2026-06-02
---

# CSharp网络Socket

C# Socket 网络编程，涵盖 TCP/UDP 通信、同步与异步接收模式、粘包拆包处理、完整服务器实现。

## 核心概念

- **TCP 流式协议**：无消息边界，需自定义应用层协议（长度前缀/分隔符/固定长度）
- **三种接收模式**：同步 `Receive`（阻塞）→ `BeginReceive`/APM（回调，有 IAsyncResult 分配）→ `SocketAsyncEventArgs`（零分配）→ `ReceiveAsync`/TAP（推荐）
- **recv=0**：对方优雅关闭连接，应立即关闭本地 Socket
- **Unity 线程安全**：Socket 回调在工作线程，需通过消息队列回主线程操作 GameObject/UI

## TCP 三次握手

```
客户端 → [SYN] seq=1000 → 服务端
客户端 ← [SYN+ACK] seq=2000, ack=1001 ← 服务端
客户端 → [ACK] seq=1001, ack=2001 → 服务端
```

## 与已有页面关联

- [[CSharp异步模型]] — Socket 的 TAP 模式（ReceiveAsync）与 CancellationToken 取消
- [[CSharp序列化IO]] — NetworkStream 与自定义协议消息的序列化
- [[CSharp文件IO]] — Stream 抽象在 Socket 中的应用
- [[CSharp进程管理]] — 进程间通信与 Socket 的对比
- [[Unity-Socket网络编程]] — Unity 端 Socket 实践与粘包处理方案

## 来源

- [[csharp-socket-network-摘要]]
