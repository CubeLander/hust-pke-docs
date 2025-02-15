# lab1_3 外部中断
## 实验目标
- 理解和补充RISC-V代理内核中，外部中断功能的部分源代码
- 实现内核时钟中断对用户程序的打断。
```cpp
int main(void) {
    printu("Hello world!\n");
    int i;
    for (i = 0; i < 100000000; ++i) {
      if (i % 5000000 == 0) printu("wait %d\n", i);
    }
 
    exit(0);
 
    return 0;
}
```
## 主要内容
- 时钟中断的实现原理
- 实验目标的具体实现

## 引用文档
- [什么是外部时钟中断](../doc/什么是外部时钟中断.md)
- [时钟中断的硬件实现原理](../doc/时钟中断的硬件实现原理.md)
- [CPU如何同步自然时间](../doc/CPU如何同步自然时间.md)
- [硬中断到软中断的切换过程](../doc/硬中断到软中断的切换过程.md)

## 时钟中断的实现原理
### 时钟中断的初始化过程
- lab1_3在`m_start`中增加了对时钟中断功能的初始化：

`void m_start(uintptr_t hartid, uintptr_t dtb)`
```cpp
void m_start(uintptr_t hartid, uintptr_t dtb) {

  spike_file_init();
  sprint("In m_start, hartid:%d\n", hartid);
  init_dtb(dtb);
  write_csr(mscratch, &g_itrframe);
  write_csr(mstatus, ((read_csr(mstatus) & ~MSTATUS_MPP_MASK) | MSTATUS_MPP_S));
  write_csr(mepc, (uint64)s_start);
  write_csr(mtvec, (uint64)mtrapvec);
  write_csr(mstatus, read_csr(mstatus) | MSTATUS_MIE);
  delegate_traps();

  // 还启用了在 supervisor 模式下的中断处理。添加于 @lab1_3
  write_csr(sie, read_csr(sie) | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // 初始化定时器。添加于 @lab1_3
  timerinit(hartid);

  asm volatile("mret");
}
```

`void timerinit(uintptr_t hartid)`
```cpp
void timerinit(uintptr_t hartid) {
  // 从现在开始，在 TIMER_INTERVAL 时间后触发定时器中断（irq）。
  *(uint64*)CLINT_MTIMECMP(hartid) = *(uint64*)CLINT_MTIME + TIMER_INTERVAL;

  // 在 MIE（机器中断使能）控制寄存器中启用机器模式定时器中断 irq。
  write_csr(mie, read_csr(mie) | MIE_MTIE);
}
```

## 时钟中断的进入和处理
- 时钟中断需要进行硬中断和软中断的处理过程，主要是因为操作系统需要将硬件层面上的定时事件与高层的系统任务分开处理。硬中断主要负责触发定时事件并更新硬件状态（如更新定时器、设置下一次中断），而软中断则用于在操作系统层面处理与定时相关的任务，如时间片管理和进程调度。通过这种分工，硬件层与操作系统的高层逻辑得以解耦，确保系统效率和稳定性。
### 时钟中断的硬件响应
- 简单来说，来自硬件的中断请求会打断用户程序的执行，进入`mtrapvec`硬中断入口，同时
  记录`mcause = CAUSE_MTIMER`，这样执行到在`handle_mtrap()`函数中时，就会进入时钟中断对应的处理程序`handle_timer()`
- 最后在`handle_timer()`置软中断位为1，从而在出硬中断服务程序后，让硬件进入软中断服务程序。其中硬件会自动设软中断原因为硬中断原因相同：`scause=mcause`。

`static void handle_timer()`
```cpp
// added @lab1_3
static void handle_timer() {
  int cpuid = 0;
  // setup the timer fired at next time (TIMER_INTERVAL from now)
  *(uint64*)CLINT_MTIMECMP(cpuid) = *(uint64*)CLINT_MTIMECMP(cpuid) + TIMER_INTERVAL;

  // setup a soft interrupt in sip (S-mode Interrupt Pending) to be handled in S-mode
  write_csr(sip, SIP_SSIP);
}
```
### 时钟中断的软件响应
- [硬中断到软中断的切换过程](../doc/硬中断到软中断的切换过程.md)：出硬中断后，硬件自动进行切换到软中断过程的详细描述。
- 在软中断服务程序中，会进行`smode_trap_vector`->`void smode_trap_handler(void)`->`handle_mtimer_trap()`的调用过程。我们需要补充的源代码就在于此。
- 实验目标仅仅是增加`g_ticks`全局变量，并复位软中断标志位。
- 虽然学校的时钟中断服务程序只涉及增加 g_ticks 和清除 SIP 标志位，但在实际操作系统中，时钟中断通常用于进程调度、上下文切换、系统时钟更新和延迟任务处理等，目的是高效管理进程并维持系统的时间精度。所以`handle_mtimer_trap()`中其实要干很多事情。

## 任务实现
- 在`handle_mtimer_trap()`中补充任务要求的源代码。

`handle_mtimer_trap()`
```cpp
static uint64 g_ticks = 0;
void handle_mtimer_trap() {
  sprint("Ticks %d\n", g_ticks);
  // TODO (lab1_3): increase g_ticks to record this "tick", and then clear the "SIP"
  // field in sip register.
  // hint: use write_csr to disable the SIP_SSIP bit in sip.
  panic( "lab1_3: increase g_ticks by one, and clear SIP field in sip register.\n" );
}
```