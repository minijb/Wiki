---
title: "游戏帧同步技术"
type: source
updated: 2026-06-02
tags: [unity, networking, frame-sync, lockstep, kcp]
---

# 游戏帧同步技术

帧同步（Frame Synchronization / Lockstep）是实时多人游戏中确保所有客户端逻辑一致性的核心技术。与状态同步（State Sync）不同，帧同步仅在客户端之间同步**操作指令**（input/command），各客户端独立执行相同的确定性逻辑以产生一致的游戏状态。

## 帧同步基本原理

帧同步的核心思想：**不同客户端可以显示完全不同的视觉表象，但必须共享统一的逻辑层**。将视觉表现与逻辑计算解耦，是理解帧同步的第一原则。

### 网络性能指标

帧同步系统的设计受两个网络指标的根本约束：

- **带宽（Bandwidth）**：限制实时游戏的**同时在线人数容量**。帧同步仅传输操作指令，天然具有极低的带宽需求。
- **延迟（Latency）**：决定实时游戏的**最低反应时间**。在普遍用户网络环境（约 200ms RTT）中，延迟是不可消除的物理限制，只能通过技术手段隐藏。

> [!important] 核心认知
> 延迟是不可消除的物理限制。帧同步的目标不是消除延迟，而是**隐藏延迟**——让玩家感知不到延迟的存在。

### 十二条应对策略

以下十二条策略构成了帧同步工程化的方法论基础：

1. **最小化数据传输**（Minimize Data Transfer）：只传输必要的操作指令，避免同步冗余状态。
2. **线程池阻塞通信**（Thread-Pool Blocking Comm）：将阻塞的网络 I/O 放入线程池，避免阻塞游戏主循环。
3. **永不为数据等待而暂停游戏**（Never Block Game for Data）：游戏主循环绝不因等待网络数据而停止推进。
4. **预测与插值**（Prediction & Interpolation）：利用历史数据预测未来状态，平滑过渡到真实数据。
5. **传输速度与加速度**（Send Velocity + Acceleration）：预测插值时，只发位置不够——必须附带速度和加速度以支持客户端外推。
6. **输入队列化**（Queue Input Until Send Time）：将键盘/触控输入缓冲到队列，在固定的发送时刻批量上传，在原理层面隐藏延迟。
7. **事件调度表**（Event Scheduling Table）：需要在所有客户端同时发生的事件，提前广播到各客户端并排定执行时间。
8. **多次攻击击杀**（Multi-Hit Kills）：减少一次性、确定性、延迟敏感的致命事件——让击杀需要多次命中，降低单次判定不一致的影响。
9. **延长弹道飞行时间**（Extend Projectile Flight Time）：在弹道飞行期间各客户端进行预测插值，为网络同步争取缓冲时间。
10. **所有移动需耗时**（All Movement Takes Time）：禁止"瞬间移动"（teleport）设计——任何位移都必须有过渡时间。
11. **可预测轨迹**（Predictable Trajectories with Inertia）：让实体按可预测轨迹运行，如移动中增加惯性，减少状态跳变。
12. **创造性合并延迟事件**（Creative Merging of Delayed Events）：在策划层面配合技术，合并前后相关事件，规避高延迟敏感的设计。

### 三大 MMO 技术难题

帧同步之外，大型多人在线游戏的同步面临三个根本性挑战：

1. **服务器响应问题**：如何在人数持续增长的情况下，提供最高的服务器响应吞吐。
2. **同步问题**：在有限网络响应条件下，实现实时互动体验。
3. **分布式世界统一问题**：如何在统一用户数据的前提下，利用分布式架构将分散的"世界"统一为一个整体。

## 帧锁定算法

帧锁定算法（Lockstep Algorithm）是最经典的帧同步实现方案，核心思路是：客户端在**关键帧**处等待服务器广播的统一操作数据，确保所有客户端步调一致。

### 严格帧锁定

严格帧锁定要求服务器收集**所有**客户端的操作数据后，才广播下一个关键帧的更新。

**客户端逻辑**：

1. 判断当前帧 `F` 是否为关键帧 `K1`：若不是，跳转至步骤 7 直接执行该帧。
2. 若是关键帧，等待 `K1` 的 `UPDATE` 数据到达。
3. 采集 `K1` 时刻的本地输入作为 `CTRL` 数据，与 `K1` 编号一起发送给服务器。
4. 从 `UPDATE K1` 中解析下一个关键帧号 `K2` 以及 `K1→K2` 之间的输入数据 `I`。
5. 将 `K1→K2` 之间的虚拟输入统一设为 `I`（用服务器数据覆盖本地操作）。
6. 令 `K1 = K2`。
7. 执行当前帧逻辑。
8. 跳转至步骤 1。

**服务端逻辑**：

1. 收集所有客户端本关键帧 `K1` 的 `CTRL` 数据，阻塞直到全部到齐。
2. 根据所有 `CTRL-K`，计算 `K2` 的 `Update` 及 `K3` 的编号。
3. 将 `Update` 广播给所有客户端。
4. 令 `K1 = K2`。
5. 跳转至步骤 1。

> [!warning] 严格锁定的致命缺陷
> **最慢的客户端会拖慢所有人**。一台客户端的网络延迟直接导致所有玩家等待。这是严格帧锁定在公网环境下几乎不可用的根本原因。

### 乐观帧锁定

乐观帧锁定（Optimistic Lockstep）解决了严格锁定中"慢客户端阻塞快客户端"的问题。

**核心思路**：服务器按固定时间间隔广播，**无论客户端是否上报操作**——未上报则视为无操作（空输入）。

**服务端**：

- 每秒 20–50 次向所有客户端广播更新（包含各客户端操作和递增帧号）。

**客户端**：

- 收到 `UPDATE` 即播放；无数据则等待。
- **不是在每帧收集操作**，而是仅在按键按下/松开时即时发送操作给服务器。这极大减少了上行流量。

**指令缓存与公平执行**：

客户端收到指令后按顺序执行。为保证公平性，采用**轮询执行**策略：

```
对每个玩家的指令队列，依次取出第一条执行 → 弹出 → 进入下一轮
直到所有玩家指令队列清空
```

这种策略在即时战略游戏（RTS）中，250ms 一个同步帧（每秒 4 次）已足够。优秀的实现还可以根据网络质量**自适应调整帧率**：网速快时提升至每秒 10 帧，网速慢时降级为每秒 2–4 帧。

## 影子跟随算法

影子跟随算法（Shadow Follow Algorithm）由 Dead Reckoning（DR / 航位推算）算法发展而来，核心隐喻：**屏幕上的实体（entity）不停追逐它的"影子"（shadow）**。

### 算法原理

1. 服务器向各客户端发送各个**影子**的状态改变：坐标、方向、速度、时间（帧号）。
2. 各客户端收到后，根据运动方向和当前时间**重新插值**修正影子的位置。
3. 影子状态是**跳变**的（收到新包立即更新），但实体追赶影子是**连续平滑**的。
4. 实体追上影子后与之保持同步。

状态数据格式：`ID + 坐标 + 方向 + 速度 + 时间（帧号）`

> [!note] 帧间同步 vs 插值同步
> - **帧间同步**（Lockstep）：各客户端每帧显示相同内容，输入传到服务器确认后各终端响应。适用于 RTS（如红警）、格斗游戏（如街霸 II 网络版）。延迟敏感，复杂度低。
> - **插值同步**（Dead Reckoning）：各客户端显示不同步但状态一致。适用于竞速游戏和 FPS。效果好，复杂度高。

### 相位滞后与惯性

**相位滞后（Phase Lag）**：实体和影子之间始终保持一个固定距离（相位差）。当控制者**突然停止**时，实体因为滞后不会"冲过头再拉回来"，避免了来回拉扯的视觉异常。

**惯性（Inertia）**：如果移动的开始、停止、转向都加入加速度/减速度平滑，则可以**不需要相位滞后**，通过惯性自然实现平滑过渡。这是更优雅的方案。

两者可以组合使用，也可二选一。惯性方案对玩家操作手感更友好，但实现复杂度更高。

### CS 与马车案例

**Counter-Strike 案例**：

- **惯性移动**：开始移动和停止时加入 0.5–1s 的加速/减速过程，避免相位滞后造成的拉扯感。
- **客户端判定射击**：开枪击中判定由客户端执行——"我"看到敌人在墙前，开枪命中，则向服务器发送命中消息。实际敌人可能在墙后（客户端差异），表现为"我"看到命中但对方感知为穿墙。**减血由服务器裁决**。
- **关键状态缓存**：跳跃的起跳和落地状态需要缓存。若每次都取中间帧状态，影子可能在最高点反复刷新，导致实体"悬浮"。实体必须逐一追赶每个关键状态影子。

**Mario Kart 案例**：

- 影子和实体均使用惯性平滑。
- **道具拾取**：玩家碰到道具后**立刻**在本地隐藏道具并通知服务器，由服务器裁决归属。因为视觉即时隐藏，玩家不会感知到服务器仲裁延迟。
- **延迟爆炸道具**：如"炸毁第一名"道具，描述为 `原角色 + 对象角色 + 约定发生时间`。所有客户端在约定时间同时触发爆炸——类似 RTS 的事件调度。
- **弹道发射物**（如乌龟壳）：服务器判断起决定作用，客户端同步判断。双方都判中 → 命中；客户端判中服务器未判中 → 客户端短暂显示击中效果后目标恢复速度。这是典型的**服务器权威 + 客户端预测**混合模式。

## KCP 可靠传输协议

帧同步依赖可靠、低延迟的传输层。TCP 的队头阻塞（Head-of-Line Blocking）对实时游戏是致命的，因此基于 UDP 的可靠传输协议成为刚需。

### 协议原理

KCP 不是底层网络协议，而是一个**算法层**——可以理解为运行在 UDP 之上的可靠传输层。它的核心目标：在不可靠的 UDP 之上提供**快速、可靠、有序**的数据传输。

KCP 与 TCP 的关键区别：

- **可配置的 RTO 策略**：TCP 使用固定的超时重传倍增策略；KCP 允许自定义重传策略。
- **选择性重传**：比 TCP 的 SACK 更激进，可配置快速重传阈值。
- **流量控制可调**：`NoDelay`、`WndSize`、`SetMtu` 等参数允许根据游戏场景调优。

**粘包处理**：

- KCP 默认使用**包模式**（packet mode），自动处理消息边界，无需额外处理粘包。
- 流模式（stream mode）下需要应用层自行处理消息分帧。

### C# 实现

KCP 在 Unity/C# 环境下的典型实现步骤：

1. **初始化组件**：
   - 创建 `UdpClient` 绑定本地端口。
   - 实现 `IKcpCallback` 接口（`Out` 发送回调 + `Recv` 接收回调）。
   - 创建 KCP 对象，绑定回调。
2. **Out → UDP 发送**：KCP 产出数据后通过 `Out` 回调，调用 `UdpClient.Send()`。
3. **接收线程**：单独线程持续 `UdpClient.ReceiveAsync()`，将收到的数据通过 `kcp.Input()` 喂入 KCP。
4. **TryRecv 取消息**：主逻辑通过 `kcp.Recv()` 获取解码后的完整消息。
5. **Update 循环**：定时调用 `kcp.Update(now)` 驱动 KCP 内部状态机（重传、ACK 等）。

`KCPSession` 完整实现：

```csharp
public class KCPSession
{
    private SimpleSegManager.Kcp kcp;
    private KCPHandle _kcpHandle;
    private UdpClient _udpClient;
    private IPEndPoint _targetEndPoint;
    private CancellationTokenSource _cts;
    private CancellationToken _ct;

    public Action<byte[]> OnReceivePack;

    public void InitSession()
    {
        _udpClient = new UdpClient(0);
        _kcpHandle = new KCPHandle();
        _cts = new CancellationTokenSource();
        _ct = _cts.Token;
    }

    public void ClientInit(IPEndPoint target, int userID)
    {
        this._targetEndPoint = target;

        _kcpHandle.Out = (Memory<byte> buffer) =>
        {
            byte[] toSend = buffer.ToArray();
            if (_udpClient != null)
                _udpClient.Send(toSend, toSend.Length, _targetEndPoint);
        };

        _kcpHandle.Recv = (byte[] buffer) =>
        {
            OnReceivePack(buffer);
        };

        kcp = new SimpleSegManager.Kcp((uint)userID, _kcpHandle);
        kcp.NoDelay(1, 10, 2, 1);   // 快速模式
        kcp.WndSize(64, 64);         // 收发窗口 64
        kcp.SetMtu(512);             // MTU 512 字节

        Task.Run(_update, _ct);
        Task.Run(_Receive, _ct);
    }

    // KCP 内部更新循环
    private async void _update()
    {
        while (true)
        {
            DateTime now = DateTime.UtcNow;
            if (_ct.IsCancellationRequested) break;

            kcp.Update(now);
            int len;
            while ((len = kcp.PeekSize()) > 0)
            {
                var buffer = new byte[len];
                if (kcp.Recv(buffer) >= 0)
                    _kcpHandle.Receive(buffer);
            }
            await Task.Delay(10);
        }
    }

    // UDP 接收线程
    private async void _Receive()
    {
        while (true)
        {
            if (_ct.IsCancellationRequested) break;

            UdpReceiveResult receive = await _udpClient.ReceiveAsync();
            if (!_targetEndPoint.Equals(receive.RemoteEndPoint)) continue;

            kcp.Input(receive.Buffer.AsSpan());
        }
    }

    public void SendPack(byte[] input)
    {
        kcp.Send(input);
    }
}
```

**调优参数**：

- `NoDelay(1, 10, 2, 1)`：启用快速模式，`interval=10ms`，`resend=2`（快速重传阈值），`nc=1`（禁用拥塞控制）
- `WndSize(sndwnd, rcvwnd)`：发送/接收窗口大小，影响吞吐和内存
- `SetMtu(512)`：最大传输单元，应根据网络路径 MTU 调整

## 预测回滚

预测回滚（Prediction & Rollback）是帧同步在格斗游戏和 MOBA 中广泛使用的补偿技术。它允许客户端在未收到服务器确认前**本地预测执行**，收到服务器数据后若不一致则**回滚重算**。

### 回滚机制

**双缓冲区架构**：

- `LBuffer`（Local Buffer）：本地预测输入。
- `SBuffer`（Server Buffer）：服务器下发的权威输入。

客户端 `Update` 始终使用 `SBuffer` 的数据。当客户端领先于服务器时（如客户端帧号 7，服务器帧号 2）：

1. 客户端恢复到帧 **1 结束时的状态**（快照/存档）。
2. 使用服务器下发的输入 **2** 重新模拟。

这就是"回滚"——**备份 → 还原 → 用服务器输入重新执行**。

**Recover 绑定**：回滚后的状态恢复需要业务层配合。实现方式：
1. 写入文件持久化状态。
2. 使用命令模式（Command Pattern）记录可重放的操作序列。

### 回滚实现方式

| 方式 | 优点 | 缺点 |
|------|------|------|
| **手动** | 完全可控 | 工作量极大，易遗漏 |
| **反射** | 自动化程度高 | 性能开销大 |
| **代码生成（推荐）** | 零运行时反射开销，类型安全 | 需预编译步骤 |
| **memcpy 内存拷贝** | 最快 | 要求数据紧凑排列（无引用类型），跨平台兼容性差 |

> [!tip] 代码生成方案
> 代码生成是目前帧同步回滚的最佳实践——通过预编译自动生成 `SaveState()` / `LoadState()` 方法，兼顾性能与可维护性。

**跨平台确定性**：多实例环境（如客户端 A、B）中，需要回滚的状态必须声明为 `static`（单例模式），确保各实例的状态快照独立且可互换。

## 工程实践

### 王者荣耀案例

王者荣耀是帧同步在移动端 MOBA 的标杆实现：

**服务端架构**：

- 收集玩家操作阶段 → 到达固定时间间隔 → 发送给所有玩家。
- `frameid`：帧率 ID，标识下一帧要进入的编号，初始 `frameid = 1`。
- `match_frames`：保存**所有玩家每一帧操作**的服务端存储。用途：
  - **录像回放**：完整记录比赛所有帧的操作序列。
  - **断线重连**：将缺失帧数据补发给重连客户端。
  - **丢包时序修复**：确认某帧已被客户端处理。
- `next_frame_opt`：收集到的 `{ frameid, opts }` 结构，opts 为操作集合。
- **定时器**：每 **66ms** 执行一次帧操作（约 15fps 逻辑帧率）。
- 比赛开始后 **5s 缓冲**用于各客户端切换场景。
- 帧操作流程：将 opts 存入 `match_frames` → 遍历所有玩家发送操作 → 清空 `next_frame_opt` 进入下一收集周期。

**客户端关键机制**：

- `sync_frameid`：客户端已同步到的帧号。
- `logic_pos`：66ms 逻辑帧间隔，以此时基进行位置和结果迭代。
- **帧间差异一致性**：播放动画的帧与帧之间存在时间差，若不统一 `logic_pos` 会导致不同步。相同操作在不同客户端因处理时间差异产生不同结果，必须使用统一时基。
- **跳帧（Frame Skipping）**：从旧帧快速追上最新帧，如 90→100，通过在循环中连续迭代补帧实现。

**客户端上报**：

- 采集自己的操作，上报给服务器的帧号是**客户端自认为的下一帧**（非服务器真实的下一帧）。
- 如果客户端上传的帧比服务器慢，则对该帧进行更新处理。

**服务端同步确认**：

```
server: 98(已处理完成), 99 → 发送到 client
client: 处理 99, 上传 100 → server
server: 收到 100 → 意味着 99 已同步完成 → 将 98 更新为 99
```

### 丢包处理

帧同步在 UDP 传输下的丢包处理策略：

**客户端丢包**：

- 服务器**补发**客户端尚未同步的帧。每个玩家对象维护 `sync_frameid`，服务器发送 `sync_frameid + 1` 到最新帧之间的所有帧数据。
- 收到帧号 `≠ next_frame_opts.frameid` 的过时帧 → 直接丢弃。对玩家手感有轻微影响，但可控。

**服务端丢包**：

- 影响极小。服务端丢包意味着某一帧的上行操作未到达——下一帧即可处理，不会造成级联问题。

**乱序处理**：UDP 存在"先发后到/后发先到"问题。客户端收到帧号小于 `local_sync_frameid` 的数据 → 直接丢弃。因为帧重传机制已保证：处理第 100 帧时，第 99 帧及之前必定已完成补发。

### 调试方法

帧同步的 Bug 往往表现为**不同客户端状态不一致**，调试需要系统化方法：

**帧状态 Dump**：

- Dump 每帧的完整状态和执行过程（行为日志）。
- 对比两个客户端 `C1`、`C2` 的日志，定位不一致出现的帧号。
- 分析该帧的输入数据与逻辑执行差异。

**预测回滚调试**：

- 维护 `Dictionary<状态, 输入, 行为>` 映射表。
- 回滚时，将老状态 Dump 到独立存储，仅保留最新状态。
- 使用 **diff 对比**定位状态差异的具体字段。

### 确定性保障

帧同步要求所有客户端的逻辑执行结果**完全相同**，这依赖于严格的确定性编程规范：

> [!danger] 两条黄金法则
> 1. **禁止使用 `float`**——必须使用**定点数（fixed-point math / `Fix64`）**。IEEE 754 浮点数在不同平台/架构上精度行为不一致。
> 2. **禁止使用多线程**——帧同步逻辑必须**单线程确定顺序执行**。多线程的调度差异会破坏确定性。

**其他确定性要求**：

- 随机数生成器必须使用**相同种子**的确定性 PRNG。
- 所有集合的遍历顺序必须稳定（如使用 `SortedDictionary` 替代 `Dictionary` 当需要遍历时）。
- 物理引擎必须使用**整数/定点数物理**，如 [[bepuphysics1int]]（deterministic integer-based physics）。
- 寻路算法必须确定性，如 [[DotRecast]]（deterministic navmesh pathfinding）。
- 跨平台（iOS/Android/PC）的确定性需要额外验证，因为 IL2CPP/Mono 在 `float` 运算、`Dictionary` 迭代顺序等方面存在差异。

**生态资源**：

- [[enet]]（reliable UDP library）：C 语言编写的轻量级可靠 UDP 通信库，可用于帧同步传输层。
- [[bepuphysics1int]]：基于整数的确定性 3D 物理引擎，C# 实现。
- [[DotRecast]]：基于 Recast&Detour 的确定性寻路库，C# 移植版。

> [!summary] 帧同步 vs 状态同步
> - **状态同步**：服务器计算游戏结果，将新状态广播给客户端。客户端直接呈现。流量大，适合 RPG/MMO。
> - **帧同步**：服务器到时间点就发送操作指令集合（无输入则发空白帧），客户端接收后 step 执行。流量极小，适合 MOBA/RTS/格斗。代价是客户端计算负载高，且要求严格确定性。
