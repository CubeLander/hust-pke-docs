# lab3_1 fork

## 实验目标

实现系统调用fork，创建子进程
对于父进程返回子进程的pid，对于子进程返回0

## 实现过程

在alloc_process中会在进程创建时，给用户栈分配一个页。
但是在fork()时，父进程的栈会被复制给子进程。

我们在不使用写时拷贝的技术时，需要在alloc_process中加一个开关，显式指定要不要创建用户栈。

我们不在alloc_process中初始化用户栈，分开实现一个init_user_stack的函数，为新创建的进程初始化用户栈。
把用户栈的初始化和用户堆的初始化都从alloc_process中解耦出来，在load_user_program中做显式调用。
这样方便我们在fork时，为子进程拷贝父进程的堆栈。

我们还需要在do_fork中做数据段的复制。