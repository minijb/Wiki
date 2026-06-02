---
title: CSAPP 程序结构与机器级表示 (Ch1-4)
updated: 2026-06-02
tags: [cs, architecture, csapp, c, assembly, binary]
sources:
  - CSAPP-1 C programme and System
  - CSAPP-2 Bit Byte and Integer
  - CSAPP-3 float
  - CSAPP-4 machine-level
  - CSAPP - debug
  - CSAPP-Assert
  - CS61C-1 basic_C
---

# CSAPP 程序结构与机器级表示

> [!abstract] 覆盖范围
> CSAPP 第 1-4 章：C 程序与系统概览、位/字节/整数表示、浮点数、x86-64 机器级编程。涵盖从 C 源码到可执行文件的过程、硬件体系结构、机器级数据表示与汇编语言。

## 第 1 章：C 程序与系统概览

### 编译的四个阶段

C 源码到可执行文件经历四个阶段：

1. **预处理 (Preprocessing)**：处理 `#` 开头的指令，展开宏、包含头文件，生成 `.i` 文件
2. **编译 (Compilation)**：编译器 `cc1` 将 `.i` 翻译为 `.s` 汇编文件
3. **汇编 (Assembly)**：汇编器 `as` 将 `.s` 翻译为机器指令，打包为**可重定位目标程序**，生成 `.o` 文件
4. **链接 (Linking)**：链接器 `ld` 合并多个 `.o` 和库文件，解析符号引用，生成可执行文件

> [!note] 重定位
> 汇编器从地址 0 开始生成代码和数据节。链接器在重定位步骤中合并输入模块，为每个节和符号分配运行时地址——此时程序中的每条指令和全局变量才拥有唯一运行时地址。

### 硬件组成

| 组件 | 描述 |
|------|------|
| **总线** | 传送定长字节块（word），连接各组件 |
| **I/O 设备** | 通过控制器（芯片组）或适配器（插卡）与总线连接 |
| **主存** | DRAM 芯片组，临时存放程序和数据 |
| **CPU** | 核心为 PC（程序计数器），执行寄存器文件与 ALU 间的操作 |

### 操作系统抽象

操作系统提供三个核心抽象：

- **进程**：对处理器、主存和 I/O 的抽象。通过上下文切换实现并发。
- **虚拟内存**：对主存和磁盘 I/O 的抽象。每个进程拥有独立的虚拟地址空间。
- **文件**：对 I/O 设备的抽象。

### 进程虚拟地址空间布局

```
高地址 ┌─────────────────────┐
      │  内核虚拟内存        │
      ├─────────────────────┤
      │  用户栈 (运行时扩展)  │  ← rsp 栈指针
      ├─────────────────────┤
      │  共享库映射区        │
      ├─────────────────────┤
      │  运行时堆 (malloc)   │  ← brk 指针
      ├─────────────────────┤
      │  .bss (未初始化数据)  │
      ├─────────────────────┤
      │  .data (已初始化数据) │
      ├─────────────────────┤
      │  .text (程序代码)    │
低地址 └─────────────────────┘
```

### 存储层次

L1 → L2 → L3 → 主存(DRAM) → 磁盘(SSD/HDD) → 远程存储

- L1 缓存：距离核心最近，分为指令缓存 (I-cache) 和数据缓存 (D-cache)
- L2 缓存：容量更大，速度次之，指令和数据共享
- L3 缓存：多核共享，容量最大

## 第 2 章：位、字节与整数

### 补码 (Two's Complement)

$$
B2T_w(\bar{x}) = -x_{w-1}2^{w-1} + \sum_{i=0}^{w-2} x_i 2^i
$$

| 值 | 二进制 (8-bit) |
|----|----------------|
| TMax | 01111111 = 127 |
| TMin | 10000000 = -128 |
| -1 | 11111111 |
| 0 | 00000000 |

> [!warning] TMin 的特殊性
> `~TMin + 1 = TMin`，即 TMin 的补码仍是自身。对 TMin 取负会导致溢出。

### 有符号与无符号转换

$T2U_w(x)$：位模式不变，解释方式改变。

$$
T2U_w(x) = \begin{cases}
x + 2^{w} & ,x < 0\\
x & ,x \geq 0
\end{cases}
$$

> [!danger] 混用陷阱
> 有符号数与无符号数运算时，**两者都被提升为无符号数**。`-1 > 0U` 结果为真，因为 `-1` 被解释为 `UMAX`。

```c
// 无限循环示例
int i = 100;
for (; i - sizeof(char) >= 0; i--) {
    // sizeof 返回 unsigned，i 被转为 unsigned → 永不为负
}
```

### 扩展与截断

| 操作 | 无符号 | 有符号 |
|------|--------|--------|
| 扩展 | 零扩展 | 符号扩展 |
| 截断 | mod $2^k$ | $U2T(x \bmod 2^k)$ |

> [!tip] short → unsigned 的双重转换
> `(unsigned) short x` 实际执行 `(unsigned)(int) short x`：先扩展长度，再转换类型。

### 整数运算

**无符号加法**：$UAdd_w(u,v) = (u+v) \bmod 2^w$，溢出时丢弃进位。

**补码加法**：位级操作与无符号相同，但溢出被检测为符号变化：
- 正数 + 正数 → 负数：正溢出
- 负数 + 负数 → 正数：负溢出

**乘法**：结果需要 $2w$ 位空间，一般只取低 $w$ 位。

**与 2 的幂运算**：
- 乘法：左移 $k$ 位 (相当于 $\times 2^k$)
- 无符号除法：逻辑右移
- 有符号除法：算术右移

### 大端与小端

| 字节序 | 0x100 | 0x101 | 0x102 | 0x103 |
|--------|-------|-------|-------|-------|
| 大端 (Big Endian) | 01 | 23 | 45 | 67 |
| 小端 (Little Endian) | 67 | 45 | 23 | 01 |

- 大端：最高有效字节在最低地址 (网络字节序)
- 小端：最低有效字节在最低地址 (x86/x64 主流)
- 大多数系统使用小端，网络协议和少数架构 (SPARC) 使用大端

> [!tip] 实用技巧
> - `x ^ y` 可实现无临时变量的值交换
> - 全 1 掩码推荐使用 `~0` 而非 `0xFFFFFFFF`

## 第 3 章：浮点数

### IEEE 754 表示

$$
V = (-1)^s \times M \times 2^E
$$

| 精度 | 符号 | 阶码 | 尾数 | 总位数 |
|------|------|------|------|--------|
| 单精度 (float) | 1 bit | 8 bits | 23 bits | 32 bits |
| 双精度 (double) | 1 bit | 11 bits | 52 bits | 64 bits |

### 规格化值

$E = e - Bias$，$M = 1 + f$（隐含前导 1）
- float: Bias = 127
- double: Bias = 1023

### 舍入规则

默认使用**向偶数舍入 (Round-to-Even)**：
- 中间值 (100...)，若最低有效位为 0 则向下舍入，为 1 则向上舍入
- 在十进制中即"四舍六入五成双"

### 浮点运算特点

| 性质 | 加法 | 乘法 |
|------|------|------|
| 交换律 | ✅ | ✅ |
| 结合律 | ❌ | ❌ |
| 分配律 | ❌ | ❌ |

> [!warning] 大数吃小数
> 一个大浮点数加一个小浮点数，结果可能仍是大浮点数（精度不足时小数的贡献完全丢失）。

## 第 4 章：x86-64 机器级编程

### 基本工具链

```bash
gcc -Og -S xxx.c    # 生成汇编代码
gcc -c xxx.c        # 仅编译为 .o
objdump -d xxx.o    # 反汇编
```

### x86-64 寄存器

| 64-bit | 32-bit | 16-bit | 8-bit | 用途 |
|--------|--------|--------|-------|------|
| `%rax` | `%eax` | `%ax` | `%al` | 返回值 |
| `%rbx` | `%ebx` | `%bx` | `%bl` | 被调用者保存 |
| `%rcx` | `%ecx` | `%cx` | `%cl` | 第 4 参数 |
| `%rdx` | `%edx` | `%dx` | `%dl` | 第 3 参数 |
| `%rsi` | `%esi` | `%si` | `%sil` | 第 2 参数 |
| `%rdi` | `%edi` | `%di` | `%dil` | 第 1 参数 |
| `%rsp` | `%esp` | `%sp` | `%spl` | 栈指针 |
| `%rbp` | `%ebp` | `%bp` | `%bpl` | 基址指针 |
| `%r8` | `%r8d` | `%r8w` | `%r8b` | 第 5 参数 |
| `%r9` | `%r9d` | `%r9w` | `%r9b` | 第 6 参数 |

### 数据传送指令

| 指令 | 说明 |
|------|------|
| `movq S, D` | 传送 8 字节 |
| `movl S, D` | 传送 4 字节 (高 32 位自动清零) |
| `movb S, D` | 传送 1 字节 |
| `movsbl S, D` | 符号扩展传送 (byte→long) |
| `movzbl S, D` | 零扩展传送 (byte→long) |
| `movabsq I, R` | 传送 64 位立即数 |

限制：两个操作数不能同时为内存地址；`movq` 的立即数只能为 32 位（自动符号扩展到 64 位）。

### 寻址模式

```
(R)          → Mem[Reg[R]]
D(R)         → Mem[Reg[R] + D]
D(Rb, Ri, S) → Mem[Reg[Rb] + S × Reg[Ri] + D]
```

其中 S = 1, 2, 4, 8；D 为偏移量（1/2/4/8 字节）。

### `leaq` —— 加载有效地址

```asm
leaq a(b, c, d), %rax   # rax = a + b + c×d
```

`leaq` 不访问内存，仅计算地址表达式。常用于：`p = &x[i]`、`x + k×y` 等。

### 算术与逻辑指令

| 双操作数 | 效果 | 单操作数 | 效果 |
|----------|------|----------|------|
| `addq S, D` | D += S | `incq D` | D += 1 |
| `subq S, D` | D -= S | `decq D` | D -= 1 |
| `imulq S, D` | D *= S | `negq D` | D = -D |
| `salq k, D` | D <<= k | `notq D` | D = ~D |
| `sarq k, D` | D >>= k (算术) | | |
| `shrq k, D` | D >>= k (逻辑) | | |
| `xorq S, D` | D ^= S | | |
| `andq S, D` | D &= S | | |
| `orq S, D` | D \|= S | | |

### 条件码与控制流

**条件码寄存器**（每个 ALU 操作后自动更新）：

| 标志 | 含义 | 用途 |
|------|------|------|
| CF | Carry Flag | 无符号溢出 |
| ZF | Zero Flag | 结果为零 |
| SF | Sign Flag | 结果为负 |
| OF | Overflow Flag | 有符号溢出 |

**比较与测试指令**（只更新条件码，不修改目标寄存器）：

- `cmpq S2, S1`：计算 `S1 - S2`，设置条件码
- `testq S2, S1`：计算 `S1 & S2`，设置条件码

**`setX` 指令**：根据条件码将单字节寄存器设为 0 或 1。

**条件跳转**：
- 无符号：`ja`, `jb`, `jae`, `jbe`
- 有符号：`jg`, `jl`, `jge`, `jle`
- 通用：`je`, `jne`, `jmp`

> [!important] 有符号比较 `a < b`
> 使用 `SF ^ OF` 判断。当发生溢出时：
> - `a < b` 且 `a - b > 0`（正溢出）→ SF=0, OF=1 → `SF^OF = 1` ✅
> - `a > b` 且 `a - b < 0`（负溢出）→ SF=1, OF=1 → `SF^OF = 0` ✅

### `cmov` 条件传送

条件传送指令 (`cmovg`, `cmovl`, `cmovge`, `cmovle`, `cmove`) 可避免分支预测失败带来的流水线惩罚。适用于简单赋值的条件分支。

### 控制流翻译模式

**if-else 翻译**：
```c
// C 代码
if (t) { A; } else { B; }

// 等价于
if (!t) goto b;
A;
goto done;
b: B;
done:
```

**switch**：使用跳转表实现 O(1) 分支，比一连串 if-else 的多次 cmp/jmp 高效。

**while 的两种模式**：
- "跳到中间"：先 goto test，循环体后回到 test
- "guarded-do"：先判断 `!t` 跳过，循环体 + 条件判断组成 do-while

### 栈帧与调用约定

```
高地址  ┌─────────────────────┐
       │  参数 7+ (大于6个)    │  ← 调用者栈帧
       ├─────────────────────┤
       │  返回地址             │
       ├─────────────────────┤
       │  保存的寄存器          │  ← 被调用者栈帧
       ├─────────────────────┤
       │  局部变量             │
       ├─────────────────────┤
低地址 │  (rsp)               │
       └─────────────────────┘
```

- `call` = `push %rip; jmp target`
- `ret` = `pop %rip`
- `push` = `sub $8, %rsp; mov src, (%rsp)`
- `pop` = `mov (%rsp), dst; add $8, %rsp`
- 前 6 个参数通过寄存器传递：`%rdi, %rsi, %rdx, %rcx, %r8, %r9`
- 栈传递参数的最小单位为 8 字节（向 8 对齐）

### 调用者保存 vs 被调用者保存

| 类型 | 寄存器 | 规则 |
|------|--------|------|
| 调用者保存 | `%rax, %rcx, %rdx, %rsi, %rdi, %r8-%r11` | 调用者需要在 call 前保存需要的值 |
| 被调用者保存 | `%rbx, %rbp, %r12-%r15` | 被调用者如果使用必须恢复原值 |

### 汇编伪指令

| 伪指令 | 含义 |
|--------|------|
| `.data` | 可读写数据段（全局/静态变量） |
| `.text` | 只读可执行代码段 |
| `.section .rodata` | 只读数据段（字符串常量、跳转表） |

### GDB 调试

```bash
gdb ./program              # 启动调试
gdb --args gcc -O2 -c foo.c  # 带参数启动
gdb -tui -c core ./program # 调试 core dump
gdb -p <pid>               # 附加到运行中的进程
```

**常用命令**：
- `break <location>` / `watch <expr>`：设置断点/监视点
- `run` / `continue` / `step` / `next` / `finish`：执行控制
- `print <expr>` / `set variable x = val`：查看/修改变量
- `backtrace` / `info locals` / `info registers`：查看状态
- `layout src` / `layout asm` / `layout split` / `layout reg`：TUI 窗口布局

**内存检测**：
```bash
valgrind --leak-check=full --show-reachable=yes ./program
```

## 相关资源

- [CSAPP 官网](https://csapp.cs.cmu.edu/3e/home.html)
- [CMU 15-213 课程](https://www.cs.cmu.edu/~213/)
- [GDB 快速参考](https://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)
- [Beej's GDB Guide](https://beej.us/guide/bggdb/)
- [南大 PA 实验](https://nju-projectn.github.io/ics-pa-gitbook/ics2019/)
- [Compiler Explorer (在线汇编)](https://godbolt.org/)
