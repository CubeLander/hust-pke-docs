# lab2_challenge3 多核内存管理

## 改动
### elf.c
注释掉了加载调试信息的代码（因为会造成两个核请求一个文件，产生死锁）
### kernel.c
```c
void load_user_program(process *proc)
...
user_heap_init(proc);
...
```
### pmm.c
在alloc_page中加入信号量sync_barrier，避免两个核同时操作物理内存池

### vmm.c

在malloc函数中，增加如果找不到合适的内存块，就在空闲链表之后继续延长新的页的功能。

### user_lib.c
将naive_malloc直接定义为请求一个4000大小的内存块。