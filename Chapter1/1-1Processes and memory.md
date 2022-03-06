# 1.1 Processes and memory

xv6进程由两部分组成，分别是用户空间的内存(指令、数据、堆栈)，和内核私有的每个进程的状态。xv6分时共享进程：他透明地在待执行的进程组中选择可用的CPU。当一个进程不在执行的时候，xv6会保存他的CPU寄存器，并在他下次运行该进程的时候恢复它们。内核将一个进程标识符，即PID，与每个进程联系起来。

进程使用fork系统调用创建一个新的进程。Fork创建的进程叫子进程，和调用进程（也叫作父进程）拥有完全一样的内存内容。Fork在父进程和子进程中都有返回。在父进程中，fork返回子进程的pid；在子进程中，fork返回0. 例如，考虑如下由C语言编写的程序片段：

```c
int pid = fork();
if(pid > 0){
    printf("parent: child=%d\n", pid);
    pid = wait((int *) 0);
    printf("child %d is done\n", pid);
} else if(pid == 0){
    printf("child: exiting\n");
    exit(0);
} else {
    printf("fork error\n");
}
```

exit系统调用会使正在被调用的进程停止运行并会释放相关的资源，例如内存和打开的文件。Exit需要一个整型的状态参数，通常0代表成功，1代表失败。wait系统调用返回当前进程中退出（或者被killed）的子进程的PID，并复制子进程的退出状态到传递给wait函数入参的地址上（将退出状态传回(int *)）。如果没有调用者的子进程退出，wait会等待一个子进程退出。如果调用者没有子进程，wait立即返回-1. 如果父进程不关心子进程的退出状态，可以传一个0的地址给wait函数入参。

在这个示例，输出行可能以任何一种顺序出来，这取决于父或子谁先到达`printf`调用。子进程退出后，父进程的`wait`返回，导致父进程打印。

```c
parent: child=1234 
child: exiting
parent: child 1234 is done
```

尽管子进程最初和父进程有相同的内存内容，但是子进程和父进程在不同的内存空间和不同的寄存器中执行。例如，当wait的返回值被存储到父进程的pid中时，它并没有改变子进程中的变量pid。子进程中的pid的值仍然是0。

**exec系统调用**使用一个新的从文件中加载出来的内存镜像—这个文件存储在文件系统中—替换掉调用进程的内存。这个文件必须拥有一个特定的格式，它需要指定那部分是命令，那部分是数据，在那个命令开始，等等。 xv6使用的是ELF格式，在第三章会进行更详细的讨论。当exec执行成功，他不会对调用进程进行返回；相反，从文件中加载的指令在ELF头文件中声明的入口点开始执行。Exec需要两个参数：包含可执行文件的文件名和一个字符串参数的数组。例如：

```c
char *argv[3];
argv[0] = "echo"; 
argv[1] = "hello"; 
argv[2] = 0; 
exec("/bin/echo", argv); 
printf("exec error\n");
```

这段代码用一个运行着参数列表echo hello的/bin/echo程序的实例替换了调用程序。大多数程序都忽略了参数数组的第一个元素，它通常是程序的名称。

xv6 shell使用上述的调用代表用户去执行程序。Shell的主要结构是简单的；可见 main ([xv6-riscv/sh.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L145)). 在main()函数中的循环使用getcmd命令从用户那里读取一行输入。然后它会调用fork系统调用，这个系统调用会创建一个shell进程的副本。当时子进程执行命令时，父进程（也就是父shell进程）会调用wait系统调用。举个例子，如果用户输入“echo hello”给shell，main()函数将会调用runcmd()，并将“echo hello”作为参数传入。然后runcmd()会运行实际的命令。对于“echo hello”，runcmd()会调用exec（[xv6-riscv/sh.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L78)）。如果exec成功，子进程会从echo指令而不是runcmd指令开始执行。在某一时刻，echo将会调用exit，exit将会导致父进程的main函数从wait中返回。（[xv6-riscv/sh.c at riscv · mit-pdos/xv6-riscv (github.com)](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L145)）【建议结合代码】

你可能会想为什么fork和exec没有合并在一个调用中；我们将在后面看到shell在其I/O重定向的实现中利用了这种分离。为了避免创建一个重复的进程然后立即替换它（用exec）的浪费，操作内核通过使用虚拟内存技术（copy-on-write）来优化fork的实现。（见 section 4.6）。

xv6隐式分配了大部分用户空间内存：fork分配子进程拷贝父进程所需要的内存，exec分配足够的内存去保存可执行的文件。在运行时需要更多内存的进程(可能是malloc)可以调用 sbrk(n)将其数据内存增加n个字节; sbrk返回新内存的位置。