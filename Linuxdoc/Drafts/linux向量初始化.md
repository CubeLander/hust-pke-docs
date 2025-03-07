在Linux内核的启动流程中，`start_kernel()`函数是内核初始化的主要入口点。硬件中断的设置是在这个函数调用链中的几个关键位置完成的：

1. 在`start_kernel()`函数中，会调用`setup_arch()`进行架构相关的初始化，这其中包含了一些与中断控制器相关的初始化。

2. 之后会调用`init_IRQ()`函数，这是专门用于设置和初始化中断系统的函数。它会初始化中断控制器并设置中断描述符表等。

3. 在`trap_init()`中会设置异常处理程序，这些处理程序响应CPU自身的异常，如页错误、除零错误等。

4. 最后，在调用`time_init()`时会初始化时钟中断，这是系统中最基本也是最频繁的中断之一。

更具体来说，对于不同架构，中断初始化的实现细节会有所不同：

- 在x86架构上，中断描述符表(IDT)会在`trap_init()`中设置
- 中断控制器(如APIC或旧的PIC)会在`init_IRQ()`中初始化
- 时钟中断通常在`time_init()`函数中设置

记住，Linux内核的具体初始化过程可能因版本而异。如果你需要查看特定版本的实现细节，最好直接查看该版本的源代码。

arch/riscv/kernel/setup.c/setup_arch(char**)