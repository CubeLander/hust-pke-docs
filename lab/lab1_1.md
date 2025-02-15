# lab1_1 系统调用


## 实验目标
- 理解和补充RISC-V代理内核中，系统调用功能的部分源代码
- 实现用户程序中printu在内核中的过程
```cpp
int main(void) {
    printu("Hello world!\n");
    exit(0);
}
```


## 主要内容
- 代理内核的启动过程
- 进程控制块和中断上下文的数据结构
- 系统调用的实现原理
- 具体实现

## 引用文档
- [RISC-V通用寄存器概述](../doc/RISC-V通用寄存器概述.md)
- [RISC-V控制状态寄存器概述](../doc/RISC-V控制状态寄存器概述.md)
- [ELF文件概述](../doc/ELF文件概述.md)
- [数据段、堆栈和内存](../doc/数据段、堆栈和内存.md)
- [C语言嵌入汇编语法](../doc/C语言嵌入汇编语法.md)
- [软硬中断的触发机制](../doc/软硬中断的触发机制.md)
- [硬件对软中断的响应过程](../doc/硬件对软中断的响应过程.md)


## 引用源代码
- [代理内核启动程序](../code/代理内核启动程序.md)
- [软中断入口程序](../code/软中断入口程序.md)
- [系统调用服务程序](../code/系统调用服务程序.md)



## 数据结构
### 进程控制块process
- 作为一个全局变量直接存储在数据段，在`switch_to`函数中初始化。
```cpp
// the extremely simple definition of process, used for begining labs of PKE
typedef struct process_t {
  // pointing to the stack used in trap handling.
  uint64 kstack;
  // trapframe storing the context of a (User mode) process.
  trapframe* trapframe;
}process;
```

### 中断上下文trapframe
- 用来保存和恢复中断前后的用户进程执行状态，并且向用户进程收集和传递中断程序的执行结果。
```cpp
typedef struct trapframe_t {
  // space to store context (all common registers)
  /* offset:0   */ riscv_regs regs;

  // process's "user kernel" stack
  /* offset:248 */ uint64 kernel_sp;
  // pointer to smode_trap_handler
  /* offset:256 */ uint64 kernel_trap;
  // saved user process counter
  /* offset:264 */ uint64 epc;
}trapframe;
```

## 代理内核的启动过程
- [代理内核启动程序](../code/代理内核启动程序.md)
- 本部分介绍了 RISC-V 代理内核的启动过程，包括从机器模式到 Supervisor 模式的切换，以及最终进入用户模式运行用户程序的过程。启动过程分为多个重要函数，每个函数在内核启动过程中起着关键作用。

  - **_mentry**  
   入口点 `_mentry` 是 RISC-V 代理内核的起始位置，负责设置硬件线程（HART）并分配栈空间。它初始化 `mscratch` 寄存器，配置每个 HART 的栈空间，最后跳转到 `m_start` 函数开始内核初始化。

  - **m_start**  
   `m_start` 函数是机器模式下的 C 语言入口点，主要负责将操作系统从机器模式切换到 Supervisor 模式。在这个过程中，它初始化文件接口，配置设备树二进制（DTB）并设置中断与异常处理方式，最后通过执行 `mret` 指令切换到 Supervisor 模式。

   - **s_start**  
   在 `m_start` 切换到 Supervisor 模式后，`s_start` 函数进一步完成内核的初始化工作。它加载用户程序，并在完成初始化后切换到用户模式开始执行用户程序。

   - **load_user_program**  
   该函数负责加载 ELF 格式的用户程序，并为其分配必要的资源，如虚拟地址空间和栈。它确保用户程序能够在内核环境中运行，并将其准备好进入执行阶段。

   - **switch_to**  
   `switch_to` 函数用于切换到指定的用户进程，它设置进程的上下文和中断处理函数入口，确保在执行过程中能够正确处理中断，并在中断结束后将控制权切换到用户程序。

此过程展示了从内核启动到用户程序运行的完整流程，强调了 RISC-V 代理内核的各个步骤，包括栈空间分配、系统初始化、特权级切换以及用户程序加载等关键操作。

- [硬件线程](../docs/硬件线程概述.md)

## 系统调用的实现原理
### 软硬中断的触发机制
- [软硬中断的触发机制](../doc/软硬中断的触发机制.md)
：本文档概述了 RISC-V 系统中的中断触发机制，主要包括两种触发方式：系统调用（ecall）和硬件异常。系统调用通过用户程序发起，用于请求内核服务，触发过程包括将程序计数器保存、跳转到内核处理程序，并在处理后恢复用户程序的执行。硬件异常，如缺页异常，则由系统硬件检测到的错误或特殊情况引发，通过内存管理单元（MMU）产生异常，并跳转到异常处理程序进行处理，恢复执行。每种触发方式都会保存当前的执行状态，跳转到适当的处理程序，确保程序能够在处理异常后继续运行。
---
### 硬件对软中断的响应过程
- [硬件对软中断的响应过程](../doc/硬件对软中断的响应过程.md)
：本文档概述了 RISC-V CPU 在中断触发后自动执行的一系列硬件操作，以确保正确处理中断并跳转到异常处理程序。主要过程包括：保存当前程序计数器（epc）到 sepc 寄存器、保存状态寄存器（sstatus）的当前值、保存异常原因（scause）以及异常地址（stval）。此外，硬件会禁用中断，防止中断嵌套，并跳转到由 stvec 寄存器指定的异常处理程序。这些操作确保了系统能够在处理异常时保持一致性，并在异常处理完成后恢复程序的执行。
  
---

### 系统调用入口的初始化过程

`void switch_to(process* proc) `
```cpp
void switch_to(process* proc) {
  assert(proc);  
  current = proc;

  // stvec控制寄存器：存储软中断的入口位置。
  // 设置系统调用入口地址为 smode_trap_vector
  write_csr(stvec, (uint64)smode_trap_vector);

  // 初始化进程控制块proc中的中断上下文trapframe
  // 初始化内核栈位置，以及进程的中断处理函数，从而在中断服务程序smode_trap_vector执行过程中读取和使用
  proc->trapframe->kernel_sp = proc->kstack;
  proc->trapframe->kernel_trap = (uint64)smode_trap_handler;

  // 用户模式开中断，并设置 sret 返回的目标模式为用户模式
  unsigned long x = read_csr(sstatus);
  x &= ~SSTATUS_SPP;  
  x |= SSTATUS_SPIE;  
  write_csr(sstatus, x);

  // 设置 S 模式下的异常程序计数器（sepc 寄存器），指向进程的 ELF 文件入口地址（即用户程序的入口地址）。
  write_csr(sepc, proc->trapframe->epc);

  // 在return_to_user()中通过 sret 进入到用户模式的main函数。
  return_to_user(proc->trapframe);
}
```
---
### 用户程序的系统调用接口
- 以printu为例，用户程序的系统调用申请是通过库函数，以及对应的`do_user_call`用户态调用接口，执行ecall实现的。
`int printu(const char* s, ...)`
```cpp
int printu(const char* s, ...) {
  va_list vl;
  va_start(vl, s);

  char out[256];  // fixed buffer size.
  int res = vsnprintf(out, sizeof(out), s, vl);
  va_end(vl);
  const char* buf = out;
  size_t n = res < sizeof(out) ? res : sizeof(out);

  // make a syscall to implement the required functionality.
  return do_user_call(SYS_user_print, (uint64)buf, n, 0, 0, 0, 0, 0);
}
```
`int do_user_call(uint64 sysnum, uint64 a1, uint64 a2, uint64 a3, uint64 a4, uint64 a5, uint64 a6,uint64 a7)`
```cpp
int do_user_call(uint64 sysnum, uint64 a1, uint64 a2, uint64 a3, uint64 a4, uint64 a5, uint64 a6,uint64 a7) {
  int ret;

  // before invoking the syscall, arguments of do_user_call are already loaded into the argument
  // registers (a0-a7) of our (emulated) risc-v machine.
  asm volatile(
      "ecall\n"
      "sw a0, %0"  // returns a 32-bit value
      : "=m"(ret)
      :
      : "memory");

  return ret;
}
```

---

### 软中断入口程序
- [软中断入口程序](../code/软中断入口程序.md)
：软中断入口程序的主要功能是处理来自用户程序的系统调用请求。首先，通过 smode_trap_vector 保存当前进程的上下文并跳转到 smode_trap_handler。在 smode_trap_handler 中，程序确保处于用户模式，并根据异常类型（如系统调用 ecall）进行相应处理。对于系统调用，handle_syscall 调整程序计数器并调用 do_syscall 执行具体的内核操作。整个过程确保用户程序能够通过软中断与内核进行有效交互。

---

### 系统调用服务程序
- [系统调用服务程序](../code/系统调用服务程序.md)
：Spike 仿真器处理中断服务程序的关键功能是通过一系列函数实现用户程序与内核之间的系统调用交互。首先，do_syscall 根据系统调用号分发请求到相应的处理函数。然后，系统调用的处理过程通过 sys_user_print 等函数将用户请求（如打印字符串）传递到内核并执行。核心流程通过 frontend_syscall 和 htif_syscall 与 Spike 仿真器进行交互，确保系统调用能够在仿真器中得到执行并返回结果。do_tohost_fromhost 负责通过轮询等待仿真器完成任务，最终将控制权返回给用户程序。这些操作保证了仿真环境中系统调用的顺畅执行与正确响应。


---

### 出中断过程
- 在中断执行完成以后，通过重新调用`void switch_to(process* proc)`，利用保存的进程控制块中的trapframe数据结构，恢复用户态进程的上下文。

`void smode_trap_handler(void)`
```cpp

void smode_trap_handler(void) {
  if ((read_csr(sstatus) & SSTATUS_SPP) != 0) panic("usertrap: not from user mode");
  assert(current);
  current->trapframe->epc = read_csr(sepc);

  // 软中断的服务过程
  if (read_csr(scause) == CAUSE_USER_ECALL) {
    handle_syscall(current->trapframe);
  } else {
    sprint("smode_trap_handler(): unexpected scause %p\n", read_csr(scause));
    sprint("            sepc=%p stval=%p\n", read_csr(sepc), read_csr(stval));
    panic( "unexpected exception happened.\n" );
  }

  // 恢复上下文，并回到用户进程
  switch_to(current);
}
```
## 具体实现
- 在handle_syscall函数中可以看到，我们需要利用trapframe中保存的参数，去调用实际的系统调用函数，并且保存返回值，传递系统调用结束后的trapframe回用户进程。
- 返回值的传递，是通过直接用系统调用的返回值，修改trapframe中保存的寄存器实现的。