# lab2_3 缺页异常
## 实验内容
在PKE操作系统内核中完善用户态栈空间的管理，使得它能够正确处理用户进程的“压栈”请求。


## lab2_3的源代码改动
### strap.c
```c
//
// the page fault handler. added @lab2_3. parameters:
// sepc: the pc when fault happens;
// stval: the virtual address that causes pagefault when being accessed.
//
void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval) {
  sprint("handle_page_fault: %lx\n", stval);
  switch (mcause) {
    case CAUSE_STORE_PAGE_FAULT:
      // TODO (lab2_3): implement the operations that solve the page fault to
      // dynamically increase application stack.
      // hint: first allocate a new physical page, and then, maps the new page to the
      // virtual address that causes the page fault.
      panic( "You need to implement the operations that actually handle the page fault in lab2_3.\n" );

      break;
    default:
      sprint("unknown page fault.\n");
      break;
  }
}

void smode_trap_handler(void) {
    ...
    case CAUSE_STORE_PAGE_FAULT:
    case CAUSE_LOAD_PAGE_FAULT:
      // the address of missing page is stored in stval
      // call handle_user_page_fault to process page faults
      handle_user_page_fault(cause, read_csr(sepc), read_csr(stval));
      break;
    ...

}
```

## 实现方法
首先判断异常是缺页异常

然后判断缺页异常是由用户栈操作引起的
    如何判定这个异常的虚拟内存请求来源于用户栈？
    通过输入的参数stval（存放的是发生缺页异常时，程序想要访问的逻辑地址）判断缺页的逻辑地址，小于USER_STACK_TOP，并大于USER_STACK_BOTTOM

最后我们为异常地址stval分配物理页，扩充用户栈。
分配物理页的实现过程，参考elf_alloc_mb