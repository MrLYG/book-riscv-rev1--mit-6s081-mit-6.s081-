# Chapter 2 Operating system organization

操作系统一个关键的要求是一次支持多个任务。例如，使用在第一章描述的系统调用接口，一个进程可以使用fork创建一个新的进程。操作系统必须在这些进程间时间共享计算机的资源。举个例子，如果进程数量大于CPU的数量，操作系统必须确保所有的进程都有机会去执行。操作系统必须安排进程间的隔离。也就是说，如果一个进程有bug或者故障，他不应该影响不依赖于出bug进程的进程。然而，完全隔离太强了，因为进程应该有可能有意地进行互动；管道就是一个例子。因此操作系统必须实现三个需求:复用、隔离和交互。

这章提供了了一个概览，操作系统如何被组织来达到这三个需求。事实证明，有很多方法可以做到这一点，但本文重点介绍以单片机内核为中心的主流设计，许多Unix操作系统都采用这种设计。本章还提供了xv6进程的概述，它是xv6的隔离单元，以及xv6启动时第一个进程的创建。

Xv6在多核RISC-V微处理器上运行，它的许多底层功能（例如，它的进程实现）是专门针对RISC-V的。RISC-V是一个64位的CPU，xv6是用 "LP64 "C语言编写的，这意味着C编程语言中的long（L）和指针（P）是64位的，但int是32位的。本书假设读者已经在一些架构上做了一些机器级的编程，并将在出现RISC-V特定的想法时引入。RISC-V的一个有用的参考资料是 "The RISC-V Reader: An Open Architecture Atlas" [David Patterson and Andrew Waterman. The RISC-V Reader: an open architecture Atlas. Strawberry Canyon, 2017.]The user-level ISA[[https://riscv.org/specifications/isa-spec-pdf/](https://riscv.org/specifications/isa-spec-pdf/)]和the privileged architecture[The RISC-V instruction set manual: privileged architecture. [[https://riscv.org/specifications/privileged-isa/](https://riscv.org/specifications/privileged-isa/)]是官方规范。

一台完整的计算机中的CPU被支持硬件所包围，其中大部分是以I/O接口的形式存在。Xv6是为qemu的"-machine virt "选项所模拟的支持硬件编写的。这包括RAM、包含启动代码的ROM、与用户键盘/屏幕的串行连接，以及用于存储的磁盘。

[2.1 Abstracting physical resources](/2-1%20Abstracting%20physical%20resources.md)