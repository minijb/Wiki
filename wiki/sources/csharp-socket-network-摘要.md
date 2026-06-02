---
title: "CSharp网络Socket-摘要"
source: "raw/cs/languages/csharp-socket-network.md"
type: source-summary
created: 2026-06-02
---

# CSharp网络Socket-摘要

> 源文件: [[CSharp网络Socket]]

## 核心内容

讲解 C# Socket 网络编程：TCP/UDP Socket 创建与绑定、三次握手四次挥手原理、同步 Receive（阻塞）vs 异步 BeginReceive（APM）vs SocketAsyncEventArgs（高性能）三种模式对比、TCP 流式协议的数据边界问题（粘包/拆包）及长度前缀解决方案、完整的异步 TCP 服务器实现、自定义应用层协议设计、Unity 中跨线程消息调度模式。

## 关键要点

1. **TCP 无消息边界**：需自定义协议（长度前缀/分隔符/固定长度）处理粘包拆包
2. **三种接收模式**：同步阻塞线程 → BeginReceive(APM) 有 IAsyncResult 分配 → SocketAsyncEventArgs 零分配 → ReceiveAsync(TAP) 推荐新项目
3. **recv=0 的含义**：对方优雅关闭连接（FIN），应立即关闭本地 Socket
4. **Unity 线程安全**：Socket 回调在工作线程，需通过消息队列回主线程操作 UI
5. **连接管理**：BeginAccept 回调中递归调用自身以持续接受新连接

## 相关页面

- [[CSharp网络Socket]]
- [[CSharp异步模型]]
- [[CSharp文件IO]]
