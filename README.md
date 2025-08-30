# 操作系统技术文章
[TOC]

聚焦Linux与Liteos_a等内核的核心机制，从原理到源码剖析进程调度、内存管理、IPC、文件系统等关键模块，覆盖面试高频考点与工程实践场景。

**核心价值**：

- 多内核横向对比（Linux vs Liteos_a）
- 源码级分析+工程化思考（含设计权衡与场景取舍）
- 配套面试考点映射与实践验证案例



# 1、仓库介绍

本仓库主要分享操作系统相关的技术文章，目标是对技术的核心原理进行剖析。

欢迎各位网友批评指正，如发现错误，请给出正确版本，验证之后会进行修改更新。

文章大致使用以下架构进行编写：

* 0、一句话总结该技术。

* 1、对技术的概念进行介绍。

* 2、分析该技术的通用实现流程。

* 3、借助具体的工程项目代码，按照通用流程进行分析，并总结在该工程项目中的具体流程。

文档写作工具：

* markdown工具：[typora](https://typora.io/)
* 思维导图工具：[知犀](https://www.zhixi.com/)（这个能在里面插入代码块）



# 2、机制分析

## 2.1、进程、线程/任务、调度

| 机制 | Liteos_a内核                                                 | Linux内核                                                    | 其他                                                         |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 进程 | [进程及Liteos_a内核进程分析.md](docs/进程及Liteos_a内核进程分析.md) | [Linux内核进程线程模型及其创建流程分析.md](docs/Linux内核进程线程模型及其创建流程分析.md) |                                                              |
| 线程 | [线程及Liteos_a内核线程分析.md](docs/线程及Liteos_a内核线程分析.md) | 同上                                                         |                                                              |
| 调度 | [调度及Liteos_a内核调度分析.md](docs/调度及Liteos_a内核调度分析.md) | 1、[Linux内核调度模型初始化和运行流程分析.md](docs/Linux内核调度模型初始化和运行流程分析.md) <br />2、[Linux内核CFS调度算法原理与源码分析.md](docs/Linux内核CFS调度算法原理与源码分析.md) |                                                              |
| 协程 |                                                              |                                                              | [协程Coroutine和C语言协程库libaco分析.md](other/协程Coroutine和C语言协程库libaco分析.md) |



## 2.2、ELF 可执行文件

| 机制    | Liteos_a内核                                                 | Linux内核 | 其他 |
| ------- | ------------------------------------------------------------ | --------- | ---- |
| ELF文件 | [ELF可执行文件及Liteos_a内核加载ELF文件分析.md](docs/ELF可执行文件及Liteos_a内核加载ELF文件分析.md) |           |      |



## 2.3、IPC 进程间通信机制

| 机制                   | Liteos_a内核                                                 | Linux内核                                                   | 其他 |
| ---------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- | ---- |
| 事件 Event             | [事件Event机制与Liteos_a内核事件机制分析.md](docs/事件Event机制与Liteos_a内核事件机制分析.md) |                                                             |      |
| 信号量 Semaphore       | [信号量Semaphore机制与Liteos_a内核信号量机制分析.md](docs/信号量Semaphore机制与Liteos_a内核信号量机制分析.md) |                                                             |      |
| 互斥锁 Mutex           | [互斥锁Mutex机制与Liteos_a内核互斥锁机制分析.md](docs/互斥锁Mutex机制与Liteos_a内核互斥锁机制分析.md) |                                                             |      |
| 读写锁 RWLock          | [读写锁RWLock机制与Liteos_a内核读写锁机制分析.md](docs/读写锁RWLock机制与Liteos_a内核读写锁机制分析.md) |                                                             |      |
| 用户态快速互斥锁 Futex | [用户态快速互斥锁Futex机制与Liteos_a内核Futex机制分析.md](docs/用户态快速互斥锁Futex机制与Liteos_a内核Futex机制分析.md) | [Linux内核Futex机制分析.md](docs/Linux内核Futex机制分析.md) |      |
| 消息队列 Queue         | [消息队列Queue机制与Liteos_a内核消息队列机制分析.md](docs/消息队列Queue机制与Liteos_a内核消息队列机制分析.md) |                                                             |      |
| 信号 Signal            | [信号Signal机制与Liteos_a内核信号机制分析.md](docs/信号Signal机制与Liteos_a内核信号机制分析.md) |                                                             |      |
| 共享内存 ShareMemory   | [共享内存Shm机制与Liteos_a内核Shm机制分析.md](docs/共享内存Shm机制与Liteos_a内核Shm机制分析.md) |                                                             |      |



## 2.4、物理内存、虚拟内存、虚实映射

| 机制                              | Liteos_a内核                                                 | Linux内核 | 其他 |
| --------------------------------- | ------------------------------------------------------------ | --------- | ---- |
| 物理内存 Physical Memory          | [物理内存与Liteos_a内核物理内存分析.md](docs/物理内存与Liteos_a内核物理内存分析.md) |           |      |
| 虚拟内存 Virtual Memory           | [虚拟内存与Liteos_a内核虚拟内存分析.md](docs/虚拟内存与Liteos_a内核虚拟内存分析.md) |           |      |
| 虚拟内存区域的红黑树RBTree管理    | [红黑树RBTree分析.zxm](docs/红黑树RBTree分析.zxm)            |           |      |
| 使用虚拟内存的堆内存管理算法TLSF  | [堆内存与Liteos_a内核堆内存TLSF机制分析.md](docs/堆内存与Liteos_a内核堆内存TLSF机制分析.md) |           |      |
| 虚拟内存管理算法SLAB              |                                                              |           |      |
| 虚实映射 Virtual-Physical Mapping | [虚实映射机制与MMU分析.md](docs/虚实映射机制与MMU分析.md)    |           |      |



## 2.5、中断与异常

| 机制                               | Liteos_a内核                                                 | Linux内核 | 其他 |
| ---------------------------------- | ------------------------------------------------------------ | --------- | ---- |
| 中断与异常 Interrupt and Exception | [中断异常机制与Liteos_a内核中断异常机制分析.md](docs/中断异常机制与Liteos_a内核中断异常机制分析.md) |           |      |
| 中断控制器 GIC                     | [中断控制器GICv2以及Liteos_a内核使用流程分析.md](docs/中断控制器GICv2以及Liteos_a内核使用流程分析.md) |           |      |



## 2.6、文件系统

| 机制                 | Liteos_a内核                                                 | Linux内核 | 其他 |
| -------------------- | ------------------------------------------------------------ | --------- | ---- |
| 文件系统 File System | [文件系统及Liteos_a内核文件系统分析.md](docs/文件系统及Liteos_a内核文件系统分析.md) |           |      |
| FATFS 分析           | [FAT文件系统分析.md](other/FAT文件系统分析.md)               |           |      |



# 3、面试高频考点映射

| 面试问题                     | 对应文档                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| CFS 调度的核心指标是什么？   | [Linux CFS分析](docs/Linux内核CFS调度算法原理与源码分析.md)  |
| Futex 如何减少系统调用开销？ | [Linux Futex 分析](docs/Linux内核Futex机制分析.md)           |
| TLSF 算法如何优化内存碎片？  | [TLSF机制分析](docs/堆内存与Liteos_a内核堆内存TLSF机制分析.md) |
| ELF文件的加载流程是什么？    | [ELF文件及加载分析](docs/ELF可执行文件及Liteos_a内核加载ELF文件分析.md) |
| 协程与线程的本质区别？       | [协程库libaco分析](other/协程Coroutine和C语言协程库libaco分析.md) |

> 持续更新中，欢迎提出高频问题补充



# 4、反馈与协作

发现错误？提交[Issue](https://github.com/ShareTechnologyForFree/OS-Kernel-Mechanism/issues)（附内核版本+代码行号）。

希望补充内容？在[讨论区](https://github.com/ShareTechnologyForFree/OS-Kernel-Mechanism/discussions)提出需求。

贡献代码？Fork仓库后提交PR（需遵循[贡献规范](CONTRIBUTING.md)）。

