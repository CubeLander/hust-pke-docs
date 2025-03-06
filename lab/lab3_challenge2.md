# lab3_challenge2 实现信号量

## 实验目标
实现信号量机制，为用户提供sem_new, sem_P和sem_V三个接口。
因为内核空间逐渐变得复杂，所以我们需要预先实现一个内核堆。

后来发现不用这个内核堆也能实现需要的功能，不过它也能方便以后的开发（希望吧）。

实现唤醒某个进程的方法，是把这个进程从阻塞队列中移出之后放入准备队列中，然后调用schedule。

## 实验原理
- 为用户程序提供sem_new、sem_P、 sem_V 三个接口
- 由于需要分离进程与进程之间、以及进程和系统之间的信号量，所以说在信号量结构中额外添加了ps字段，用来在用户程序调用信号量时检验是否为owner

需要注意的是，从sem_P返回的时候，需要进入内核态，而不是用户态。
所以说，我们需要在sem_P阻塞进程的时候，保存内核态的上下文, 并且在调用schedule时返回到内核态的上下文。

我们需要在PCB中增加一个内核态上下文ktrapframe,还有一个内核态中断标志位，用来分辨返回时是到ktrapframe里去还是到trapframe里去。
或者利用内核栈是否为kernel_stack_top来判断目前是否还有内核态过程没有结束。
所以说我们只要保存sem_P当中的栈状态，在schedule中切换到这个栈状态，然后执行返回，就能够将内核的运行状态切换到sem_P的调用者，调用sem_P之后的那条指令了。

## 内核上下文的实现

### 在PCB中增加ktrapframe字段
- 修改process_t，增加`trapframe* ktrapframe;`
- 修改alloc_process(), 将ktrapframe初始化为null
- 在process.c中增加save_kernel_context和restore_kernel_context函数


## 后记

成熟的wait实现，是和内核的“异步信号量”共享一个出口的，我们只是利用了sem的队列而已。sem只是服务用户的。