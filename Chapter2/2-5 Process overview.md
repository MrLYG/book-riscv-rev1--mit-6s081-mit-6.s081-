# 2.5 Process overview

xv6的隔离单元（像其他Unix系统一样）是进程。进程的抽象阻止一个进程破坏或者窥视其他进程的内存，CPU，文件描述符，等。它也阻止进程破坏内核本身，因此一个进程不能颠覆内核的隔离机制。内核必须小心的实现进程的抽象，因为一个有bug的或者恶意的程序或许会欺骗内核或者硬件去做一些不好事情（例如，规避隔离）。被内核使用去实现进程的机制包括用户、监督者模式标志，地址空间，以及线程时间分割。

为了帮助隔离，进程抽象提供给程序一个幻象，即程序独自拥有一个私有机器。一个进程提供给程序似乎私有的内存系统或者地址空间，其他的进程不能对其进行读写。进程也提供给程序似乎独自拥有CPU去执行程序的指令。

XV6使用页表（通过硬件实现）来给每个进程他自己的地址空间。RISC-V页表一个虚拟地址（RISC-V指令操作的地址）转换（或“映射”）到一个物理地址（CPU芯片发送到主存的地址）。

XV6给每一个进程维护一个页表，页表定义了该进程的地址空间。正如图2.3所示，地址空间，包括进程的用户内存，是由虚拟地址0开始的。指令在前，然后是全局变量，然后是栈，最后是堆（用于malloc），进程可以根据需要进程扩展。有大量的因素限制进程地址空间的最大长度：RISC-V的指针宽度为64位；在页表中寻找虚拟地址时，硬件仅使用低39位；并且xv6仅使用39位中的38位。因此，最大地址是
$$ 
2^{38}-1=0x3fffffffff 
$$
也就是**MAXVA**（[xv6-riscv/riscv.h at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L348)）。在地址空间的顶部，xv6为trampoline保留了一个页面，并为进程的trapframe映射了一个页面，以便切换到内核，我们将在第四章解释。

xv6内核为每一个进维护了许多状态，它将这些状态收集到proc结构体中([xv6-riscv/proc.h at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.h#L86))。进程内核状态中最重要的部分是页表，内核栈和他的运行状态。我们将使用p->xxx代表proc结构体中的元素。例如，p->pagetable是一个指向进程页表的指针。

每一个进程都有一个执行线程(简称线程)，它执行进程的指令。线程可以被暂停然后恢复。为了透明地在进程间切换，内核暂停当前运行的线程并且恢复其他进程的线程。很多线程的状态(本地变量，函数调用的返回地址)被存到线程栈中。每一个进程有两个栈:一个用户栈，一个内核栈(p->kstack)。当进程执行用户指令，只有的用户栈被使用，他的内核栈是空的。当进程进入内核(由于系统调用或者中断)，内核代码在进程的内核栈上执行；当进程在内核中时，它的用户栈依旧包含被存储的数据，但这些数据并未被使用。进程的线程在用户栈和内核栈间交替执行。内核栈是独立的(并且是被保护的，不受到用户代码的影响) 因此，即使在一个进程的用户栈被破坏时，内核依旧可以执行。

一个进程可以通过执行RISC-V的ecall指令来进行系统调用。这个指令可以提升硬件权限级别和改变程序的计数器到内核定义的入口点。在入口点的代码切换到内核栈，执行内核指令，这些指令实现系统调用。当系统调用完成，内核切换回用户栈，并且返回到用户空间通过调用sret指令，它降低硬件权限级别并且在系统调用指令后恢复执行用户指令。进程的线程可以在内核中阻塞等待I/O，当I/O完成后，再从离开的地方恢复。

p->state表示进程是否被分配、准备运行、正在运行、等待I/O或退出。 

p->pagetable持有进程的页表，格式是RISC-V硬件所期望的。xv6使分页硬件在用户空间执行进程时使用该进程的p->pagetable。进程的页表也作为记录，记录被分配用于存储进程内存的物理页的地址。