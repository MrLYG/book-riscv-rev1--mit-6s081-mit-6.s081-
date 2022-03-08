# 2.2 User mode, supervisor mode, and system calls

强隔离需要在应用和操作系统间有一个硬分界线。如果一个应用出来错误，我们不希望操作系统或者其他应用也失败。相反，操作系统应该有能力清除失败的应用然后继续运行其他应用。为了达到强隔离性，操作系统必须安排应用不可以修改（甚至不可以读）操作系统的数据结构和指令，并安排应用不能访问其他进程的内存。

CPU提供硬件支持强隔离性。举个例子，RISC-V有三种执行命令的模式，分别是machine mode，supervisor mode，和 user mode。在machine mode下指令执行拥有全部的权限；CPU的正式以machine mode开始运行的。machine mode 主要的被用于去配置计算机。XV6在machine mode下执行一小段代码后将模式转为supervisor mode。

在supervisor mode，CPU被允许去执行特权指令（privileged instructions）：例如，启用和禁用中断，读写保存页表地址的寄存器，等等。如果一个在user模式下的应用去执行一个特权指令，CPU不会执行这个指令，而是切换到supervisor模式，因此supervisor模式的代码可以终止这个应用，因为他做了他不应该做的事情（行为不符合user mode） 。第一章中的图1.1说明了这种组织结构。

一个应用只能执行user模式的指令（例如，数字相加，等等）被称为运行在用户空间，然而运行在supervisor模式的软件也可以执行特权指令，被称为运行在内核空间。运行在内核空间（或者说是supervisor模式下）的软件被称为内核。

一个应用想要调用内核函数（例如xv6的read系统调用）必须过渡到讷河。CPU提供了一个特殊的指令，这个指令可以切换CPU从user模式到supervisor模式，并且在内核指定的入口处进入内核。(RISC-V为这此提供了ecall指令)。一旦CPU切换到supervisor模式，内核就可以验证系统调用的参数，决定是否允许应用程序执行要求的操作，然后拒绝它或者执行它。内核控制过渡到supervisor模式的入口是非常重要的；如果应用程序可以决定内核的入口在哪里，例如，一个恶意的应用可以在跳过参数校验后的点进入内核。