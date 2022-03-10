# 2.6 Code: starting xv6 and the first process

为了使得xv6更加具体，我们将概述内核如何启动和运行第一个进程的。接下来的章节会更细节的描述概述中使用到的机制。

当RISC-V计算机通电后，它会初始化自己，并运行一个boot  loader，它存储在一个只读的内存中。boot loader加载xv6内核到内存。然后，在机器模式下，CPU在_entry([xv6-riscv/entry.S at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/entry.S#L6))开始执行xv6. RISC-V启动时禁止分页硬件；虚拟地址直接映射到物理地址。

loader加载xv6内核到地址为0X80000000的内存中。把内核放到0x80000000下而不是0x0下，是因为地址0x0到地址0x80000000中存储一些I/O设备。

entry指令设置了一个栈，让xv6可以运行C语言代码。Xv6在文件start.c([xv6-riscv/start.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/start.c#L11))中声明了一个初始的栈空间，stack0。_entry加载栈指针寄存器sp到地址为stack0+4096，也就是栈的顶部，因为RISC-V的栈是从上到下的。现在内核有了一个栈，_entry调用在start([xv6-riscv/start.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/start.c#L21))中C语言代码。

sart函数执行一些只被允许在机器模式下的配置，然后切换到监督者模式。为了进入监督者模式，RISC-V提供了mret指令。这个指令经常被用作从一个监督者模式下的调用返回到机器模式。start并不是从这样的调用中返回，而是把事情设置的像曾经发生过一样:它在mstatus寄存器中设置之前的特权模式为监督者模式，通过把main的地址写进寄存器mepc来设置返回地址为main，通过在页表寄存器satp中写0来禁用监督者模式下的虚拟地址转换，并把所有中断和异常委托给监督者模式。

在进入特权者模式之前，**start**还要执行一项任务：对时钟芯片进行编程以初始化定时器中断。在完成了这些内务工作后，**start**通过调用**mret**  "返回" 到监督者模式。这将导致程序计数器变为main（[xv6-riscv/main.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/main.c#L11)）的地址。

在**main**([xv6-riscv/main.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/main.c#L11))初始化几个设备和子系统后，它通过调用**userinit**([xv6-riscv/proc.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L212))来创建第一个进程。第一个进程执行一个用RISC-V汇编编写的小程序**initcode.S**（[xv6-riscv/initcode.S at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/initcode.S#L1)），它通过调用**exec**系统调用重新进入内核。正如我们在第一章中所看到的，**exec**用一个新的程序（本例中是/init）替换当前进程的内存和寄存器。一旦内核完成**exec**，它就会在**/init**进程中返回到用户空间。**init** ([xv6-riscv/init.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/init.c#L15))在需要时会创建一个新的控制台设备文件，然后以文件描述符0、1和2的形式打开它。然后它在控制台上启动一个shell。这样系统就启动了。