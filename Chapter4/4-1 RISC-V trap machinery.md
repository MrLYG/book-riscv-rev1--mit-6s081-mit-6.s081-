# 4.1 RISC-V trap machinery

每一个RISC-V CPU有一系列内核写入的控制器寄存器去告诉CPU如何处理traps，并且内核可以读这些寄存器去发现已经发生的trap。RISC-V文档包含更多内容，riscv.h([xv6-riscv/riscv.h at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L1))包含xv6使用的定义。这里是最重要的寄存器的概述:

- stvec: 内核在这里写下trap处理程序的地址；RISC-V到这里来处理trap。
- sepc: When a trap occurs, RISC-V saves the program counter here (since the pc is then
overwritten with stvec). The sret (return from trap) instruction copies sepc to the pc.
The kernel can write to sepc to control where sret goes. 
当trap发生时，RISC-V会将程序计数器PC保存在这里（因为**PC**会被**stvec**覆盖）。**sret**(从trap中返回)指令将**sepc**复制到pc中。内核可以写**sepc**来控制**sret**的返回到哪里。
- scause: RISC-V在这里写入一个数字用于描述此次trap发生的原因
- sscratch: 内核会在这里放一个值，这个值在trap处理程序开始的地方会很有用。
- sstatus: sstatus中的SIE位控制设备中断是否被启用。如果内核清除了SIE，RISC-V会推迟设备中断知道内核设置了SIE。SSP位指明一个trap是来自用户模式还是监督者模式，并控制sret返回到什么模式。

上述关于traps的寄存器在监督者模式被下处理，这些寄存器不能在用户模式下被读或者写。也有同样的一些列控制器用户机器模式下的traps处理；xv6只在定时器中断的特殊情况下使用他们。

多核芯片的每个CPU都有一组自己的这些寄存器，并且同一时间不止一个CPU处理trap。

当需要执行trap时，RISC-V硬件对所有的trap类型（除定时器中断外）进行以下操作:

1. 如果trap是一个设备中断，并且sstatus的SIE位被清除，不做一下任何操作。
2. 清除SIE位以禁用中断。
3. 复制PC(程序计数器)到sepc寄存器
4. 保存目前的模式(用户模式或者监督者模式)到sstatus的SPP位。
5. 在scause寄存器设置该次trap发生的原因
6. 将模式设为监督者。
7. 将stvec寄存器的内容复制到pc寄存器。(stvec寄存器存储trap处理程序的地)。
8. 执行新的pc

注意: CPU不会切换内核页表，不会切换内核栈也不会存储处理pc之外的任何寄存器。内核软件必须执行这些任务。在trap时期CPU做很少工作的一个原因是为了给软件提供更好的灵活性；例如，在一些情况下，一些操作系统不要求页表的切换，这样可以提高性能。
You might wonder whether the CPU hardware’s trap handling sequence could be further simplified. For example, suppose that the CPU didn’t switch program counters. Then a trap could switch
to supervisor mode while still running user instructions. Those user instructions could break the
user/kernel isolation, for example by modifying the satp register to point to a page table that
allowed accessing all of physical memory. It is thus important that the CPU switch to a kernelspecified instruction address, namely stvec.

你或许想，CPU硬件的trap处理流程是否可以进一步简化。例如，假设CPU不切换PC程序计数器。那么trap切换到监督者模式时，还在运行用户指令。这些用户指令可以打破用户空间/内核空间的隔离，例如通过修改satp寄存器指向一个允许访问所有物理内存的页表。因此，CPU必须切换到内核指定的指令地址，即**stvec**。（也就是说上述的trap处理流程已经是最简化的了，不能再简化了）