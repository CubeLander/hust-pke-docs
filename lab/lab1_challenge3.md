# lab1_challenge3 多核启动和运行
由于这个实验需要管理多个进程的内存，所以需要在lab2中实现mmu以后再完成。
什么？通过头歌的测试？连MMU都没有谈多核有任何意义吗？

## 参考资料

[信号量和自旋锁](../doc/信号量和自旋锁.md)



## 实验目标
之前的实验都在单核环境下进行。在本次实验中，你需要修改操作系统内核使其支持两核并发运行，并且在每个核上加载一个程序运行，等到两个程序都执行完毕后退出并关闭模拟器。

### 给定应用

#### user/app0.c

```c
#include "user_lib.h"
#include "util/types.h"

int main(void) {
  printu(">>> app0 is expected to be executed by hart0\n");
  exit(0);
}
```

#### user/app1.c

```c
#include "user_lib.h"
#include "util/types.h"

int main(void) {
  printu(">>> app1 is expected to be executed by hart1\n");
  exit(0);
}
```

在本次实验中，给定两个简单的用户程序，每个程序会输出一句话。你需要让每个核分别加载一个程序，并能够正确运行，输出相应内容然后退出。

## 实验原理

## 需要修改的部分

### elf.c
需要分别加载两个程序

### kernel.c

#### 全局变量
```c
// process is a structure defined in kernel/process.h
process user_app[2];
```

#### m_start
只需要做一次的部分：
spike文件接口初始化
设备树初始化
```c
volatile static int counter = 0;
void m_start(uintptr_t hartid, uintptr_t dtb) {
  if (hartid == 0) {
    spike_file_init();
    init_dtb(dtb);
  }
  sync_barrier(&counter, NCPU);
  // 这一个函数是用来同步不同核心的任务的。
  sprint("In m_start, hartid:%d\n", hartid);
  write_tp(hartid);

  // save the address of trap frame for interrupt in M mode to "mscratch". added
  // @lab1_2
```
#### load_user_program
虽然说还没有引入调度器，但是由于在中断恢复的时候要读进tp寄存器，所以要做一下初始化。
```
  proc->trapframe->regs.tp = read_tp();
```

### process.c

```c
// current points to the currently running user-mode application.
process* current[2];

```

然后我们需要修改所有对current的使用，都改成current[hartid]