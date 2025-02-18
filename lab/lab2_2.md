# lab2_2 简单内存分配

## 实验目标
完成两个用户态系统调用naive_malloc()和naive_free()， 其中naive_malloc和两个函数的系统调用用户接口已经实现。
```c

struct my_structure {
  char c;
  int n;
};

int main(void) {
    struct my_structure* s = (struct my_structure*)naive_malloc();
    s->c = 'a';
    s->n = 1;

    printu("s: %lx, {%c %d}\n", s, s->c, s->n);
    naive_free(s);

    exit(0);
}
```

## lab2_2源代码变化
### syscall.c
```c

//
// maybe, the simplest implementation of malloc in the world ... added @lab2_2
//
uint64 sys_user_allocate_page() {
  void* pa = alloc_page();
  uint64 va = g_ufree_page;
  g_ufree_page += PGSIZE;
  user_vm_map((pagetable_t)current->pagetable, va, PGSIZE, (uint64)pa,
         prot_to_type(PROT_WRITE | PROT_READ, 1));

  return va;
}

//
// reclaim a page, indicated by "va". added @lab2_2
//
uint64 sys_user_free_page(uint64 va) {
  user_vm_unmap((pagetable_t)current->pagetable, va, PGSIZE, 1);
  return 0;
}
```
### vmm.c
```c
//
// unmap virtual address [va, va+size] from the user app.
// reclaim the physical pages if free!=0
//
void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free) {
  // TODO (lab2_2): implement user_vm_unmap to disable the mapping of the virtual pages
  // in [va, va+size], and free the corresponding physical pages used by the virtual
  // addresses when if 'free' (the last parameter) is not zero.
  // basic idea here is to first locate the PTEs of the virtual pages, and then reclaim
  // (use free_page() defined in pmm.c) the physical pages. lastly, invalidate the PTEs.
  // as naive_free reclaims only one page at a time, you only need to consider one page
  // to make user/app_naive_malloc to behave correctly.
  panic( "You have to implement user_vm_unmap to free pages using naive_free in lab2_2.\n" );

}

```

## 虚拟内存分配过程（只分配一个页）
首先找到一个空闲的物理页

然后在用户页表中创建多级页表，和最终的页表项，建立虚拟地址到物理页之间的对应关系

## 虚拟内存回收过程

首先确定与释放的虚拟内存空间对应的全部页表项
如果free=1，释放所有对应的物理页，并收回物理页内存池
然后修改页表项，置valid位为零。
我们不需要释放中间的页表。