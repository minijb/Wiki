---
title: "CSAPP 程序结构与机器级表示"
type: source-summary
updated: 2026-06-02
source: "raw/cs/architecture/csapp-program-structure.md"
tags: [cs, architecture, csapp, c, assembly, binary]
---

# CSAPP 程序结构与机器级表示

## 来源信息

- **原始文件**：`raw/cs/architecture/csapp-program-structure.md`
- **类型**：技术参考文档

## 要点

- **编译流程**：C 源码经预处理→编译→汇编→链接四个阶段生成可执行文件，链接器完成符号解析与重定位
- **硬件层次**：总线、I/O 设备（控制器/适配器）、DRAM 主存、CPU（PC + 寄存器文件 + ALU）；L1→L2→L3→主存→磁盘的存储金字塔
- **操作系统抽象**：进程（CPU 抽象）、虚拟内存（内存抽象）、文件（I/O 抽象）；进程虚拟地址空间从低到高依次为 .text/.data/.bss/堆/共享库/栈/内核空间
- **二进制表示**：补码表示、大端小端字节序、符号扩展/零扩展/截断规则、有符号与无符号混用陷阱
- **整数运算**：无符号加法为模运算、补码加法溢出检测（正+正→负，负+负→正）、位操作技巧
- **IEEE 754 浮点数**：符号+阶码+尾数三部分表示；向偶数舍入规则；浮点加法满足交换律但不满足结合律，需警惕大数吃小数
- **x86-64 寄存器**：16 个 64 位通用寄存器（%rax/%rbx/%rcx/%rdx/%rsi/%rdi/%rsp/%rbp/%r8-%r15），各寄存器用途约定与调用约定
- **寻址模式**：立即数、寄存器、内存（含 D(Rb,Ri,S) 基址+变址+比例因子模式）
- **数据传送**：mov 族指令（movq/movl/movb/movabsq/movsbl/movzbl），`leaq` 用于地址计算而不访问内存
- **条件码与控制流**：CF/ZF/SF/OF 条件码，cmp/test 只更新条件码不修改目标，setX/jX/cmov 指令集，if/while/switch 的编译模式（跳转表 vs 多次分支）
- **栈帧与调用约定**：call/ret 的 push/pop 机制，前 6 个参数通过寄存器传递，调用者保存 vs 被调用者保存寄存器
- **调试工具**：GDB（断点/监视/步进/TUI 窗口布局）、Valgrind（内存泄漏检测）、objdump（反汇编）

## 关联页面

- [[concepts/CSAPP程序结构与机器级表示|CSAPP 程序结构与机器级表示]] — 概念总览
- [[concepts/CSAPP系统级主题|CSAPP 系统级主题]] — Ch5-10 系统主题
- [[concepts/操作系统基础|操作系统基础]] — OS 概念总览
