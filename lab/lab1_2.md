# lab1_2 硬件中断
## 实验目标
- 理解和补充RISC-V代理内核中，处理硬件中断功能的部分源代码
- 用户程序会执行一个非法指令，我们需要完善操作系统内核代码，从而实现拦截这个指令，并处理硬件异常的过程。
```cpp
int main(void) {
  printu("Going to hack the system by running privilege instructions.\n");
  // we are now in U(user)-mode, but the "csrw" instruction requires M-mode privilege.
  // Attempting to execute such instruction will raise illegal instruction exception.
  asm volatile("csrw sscratch, 0");
  exit(0);
}
```
## 主要内容
- 硬件中断的实现原理
- 实验具体实现

## 引用文档
- [软硬中断的触发机制](../doc/软硬中断的触发机制.md)
- [硬件对硬中断的响应过程](../doc/硬件对硬中断的响应过程.md)
- [什么是panic](../doc/什么是panic.md)
## 引用源代码
- [代理内核启动程序](../code/代理内核启动程序.md)
- [硬中断入口程序](../code/硬中断入口程序.md)
## 硬中断的实现原理
### 硬中断的触发和响应
- [软硬中断的触发机制](../doc/软硬中断的触发机制.md)
- [硬件对硬中断的响应过程](../doc/硬件对硬中断的响应过程.md)：
  本文档详细描述了RISC-V架构下硬件中断和异常的触发条件及操作系统的响应过程。中断和异常分为同步异常（如非法指令、特权级不符等）和异步中断（如外部设备中断、定时器中断）。当中断或异常发生时，CPU自动保存当前状态并根据mtvec寄存器跳转到相应的异常处理程序。异常处理过程包括保存上下文、切换内核栈、调用特定异常处理函数、恢复用户上下文并安全返回用户模式。该流程保证了系统能有效响应各种硬件中断，维持系统稳定。
### 硬中断入口的初始化
- lab1_2在`m_start`中增加了硬件中断入口`mtvec`的初始化：
  
```cpp
void m_start(uintptr_t hartid, uintptr_t dtb) {
  spike_file_init();
  sprint("In m_start, hartid:%d\n", hartid);

  init_dtb(dtb);

  // 将中断处理的trap frame地址保存到mscratch寄存器中，用于M模式下的中断处理。@lab1_2
  write_csr(mscratch, &g_itrframe);

  write_csr(mstatus, ((read_csr(mstatus) & ~MSTATUS_MPP_MASK) | MSTATUS_MPP_S));
  write_csr(mepc, (uint64)s_start);

  // 设置M模式下的中断处理向量，处理所有中断请求。@lab1_2
  write_csr(mtvec, (uint64)mtrapvec);


  delegate_traps();
  asm volatile("mret");
}

```
### 硬中断处理程序
- [硬中断处理程序](../code/硬中断处理程序.md)：本程序段描述了在RISC-V架构的M模式下的硬中断的服务程序。具体包括中断向量入口 (`mtrapvec`)、中断处理函数 (`handle_mtrap`) 和非法指令处理函数 (`handle_illegal_instruction`) 的实现。程序通过保存和恢复寄存器的状态、切换栈空间，并调用相应的中断处理函数来响应和处理不同类型的硬中断和异常。

## 具体实现
- 补充调用过程中的`handle_illegal_instruction`调用即可