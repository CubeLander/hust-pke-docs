# lab2的内核初始化改动

## s_start
在s_start之前的过程没有发生变化。
```c
int s_start(void) {
  sprint("Enter supervisor mode...\n");
  //  satp是(S-mode)下控制页表的核心寄存器，用于地址变换和权限控制
  //    存储分页模式MODE(bare/sv39/sv48)
  //    区分不同进程的地址空间ASID
  //    存储页表基址PPN
  write_csr(satp, 0);

  // 初始化物理页资源表
  //    确定核心占用的内存区域首尾 g_kernel_start, g_kernel_end
  //    确定所有可用的物理页资源首尾 free_mem_start_addr, free_mem_end_addr
  //    以此创建空闲物理页表 create_freepage_list(free_mem_start_addr, free_mem_end_addr);
  //
  // 空闲物理页资源表的数据结构
  //    通过全局变量 static list_node g_free_mem_list; 管理所有空闲的物理页
  //    list_node->next 指向下一个空闲的物理页基址，最后一个节点指向空指针。
  //    释放物理页：在物理页开头创建链表节点，并插入物理页链表头部。
  //    分配物理页：分配链表中第一个节点，并更新物理页链表。
  pmm_init();

  // 参见kernel.lds.
  // 初始化内核空间的内存地址映射
  //    将code and text segment映射为虚实地址相同的 读/执行权限页
  //    将内核剩下的段映射为虚实地址相同 的读/写页
  //    分配一个空闲页用作全局页目录 g_kernel_pagetable(指向对应的内存页地址)
  kern_vm_init();

  //  写入satp寄存器并刷新tlb缓存
  //    从这里开始，所有内存访问都通过MMU进行虚实转换
  enable_paging();

  sprint("kernel page table is on \n");
  load_user_program(&user_app);
  sprint("Switch to user mode...\n");
  switch_to(&user_app);
  return 0;
}
```

## load_user_program
```c
****void load_user_program(process *proc) {
  sprint("User application is loading.\n");

  // 为进程控制块的各个成员指针分配物理内存
  proc->trapframe = (trapframe *)alloc_page();    memset(proc->trapframe, 0, sizeof(trapframe));
  proc->pagetable = (pagetable_t)alloc_page();    memset((void *)proc->pagetable, 0, PGSIZE);

  // 内核栈是自上而下增长的，所以说起始位置是页的高地址（左闭右开）
  proc->kstack = (uint64)alloc_page() + PGSIZE;
  uint64 user_stack_bottom = (uint64)alloc_page();

  // USER_STACK_TOP = 0x7ffff000, defined in kernel/memlayout.h
  proc->trapframe->regs.sp = USER_STACK_TOP;  //virtual address of user stack top

  sprint("user frame 0x%lx, user stack 0x%lx, user kstack 0x%lx \n", proc->trapframe,
         proc->trapframe->regs.sp, proc->kstack);

  load_bincode_from_host_elf(proc);

  // 为用户栈创建地址映射
  user_vm_map((pagetable_t)proc->pagetable, USER_STACK_TOP - PGSIZE, PGSIZE, user_stack_bottom,
         prot_to_type(PROT_WRITE | PROT_READ, 1));

  // 为中断上下文创建地址映射
  user_vm_map((pagetable_t)proc->pagetable, (uint64)proc->trapframe, PGSIZE, (uint64)proc->trapframe,
         prot_to_type(PROT_WRITE | PROT_READ, 0));

  // 因为用户模式触发中断时，使用的stvec=smode_trap_vector仍然是虚拟地址，所以说要把虚拟中断入口地址-->物理中断入口地址的虚实映射关系，也加入到用户模式的页表当中，才能让软中断成功跳转到正确的中断入口向量地址。
  user_vm_map((pagetable_t)proc->pagetable, (uint64)trap_sec_start, PGSIZE, (uint64)trap_sec_start,
         prot_to_type(PROT_READ | PROT_EXEC, 0));
}
```

## elf_alloc_mb
我改进了代码，让elf_alloc_mb支持多个物理页：
```c
static void *elf_alloc_mb(elf_ctx *ctx, uint64 elf_pa, uint64 elf_va, uint64 size) {
  elf_info *msg = (elf_info *)ctx->info;

  // 计算需要多少页
  uint64 num_pages = (size + PGSIZE - 1) / PGSIZE; // 向上取整
  void *first_pa = NULL;

  for (uint64 i = 0; i < num_pages; i++)
  {
    void *pa = alloc_page();
    if (pa == 0)
      panic("uvmalloc mem alloc failed\n");

    memset((void *)pa, 0, PGSIZE);

    // 记录第一个分配的物理页
    if (i == 0)
      first_pa = pa;

    // 映射虚拟地址到物理地址
    user_vm_map((pagetable_t)msg->p->pagetable, elf_va + i * PGSIZE, PGSIZE, (uint64)pa,
                prot_to_type(PROT_WRITE | PROT_READ | PROT_EXEC, 1));
  }

    return first_pa; // 返回第一个物理页的地址
}
```

- pmm_init()
- kern_vm_init()