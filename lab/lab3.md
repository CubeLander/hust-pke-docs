# lab3 实验3 进程管理

## lab3在lab2基础上的变化
### process.h
#### 新的全局声明
```c
// riscv-pke kernel supports at most 32 processes
#define NPROC 32
// maximum number of pages in a process's heap
#define MAX_HEAP_PAGES 32
```
#### 进程控制块process的变化
```c
  heap_block* heap;
  // size_t heap_size;


  // points to a page that contains mapped_regions. below are added @lab3_1
  mapped_region *mapped_info;
  // next free mapped region in mapped_info
  int total_mapped_region;
  // heap management
  process_heap_manager user_heap;

    // process id
  uint64 pid;
  // process status
  int status;
  // parent process
  struct process_t *parent;
  // next queue element
  struct process_t *queue_next;
```
#### 数据结构mapped_region
用来记录所有分配给用户进程的内存区域。
在elf_load中进行写入

```c
// the VM regions mapped to a user process
typedef struct mapped_region {
  uint64 va;       // mapped virtual address
  uint32 npages;   // mapping_info is unused if npages == 0
  uint32 seg_type; // segment type, one of the segment_types
} mapped_region;
```
#### 数据结构process_heap_manager
```c
typedef struct process_heap_manager {
  // points to the last free page in our simple heap.
  uint64 heap_top;
  // points to the bottom of our simple heap.
  uint64 heap_bottom;

  // the address of free pages in the heap
  uint64 free_pages_address[MAX_HEAP_PAGES];
  // the number of free pages in the heap
  uint32 free_pages_count;
}process_heap_manager;

```

### 全局变量
```c
// process pool. added @lab3_1
process procs[NPROC];

// current points to the currently running user-mode application.
process* current[NCPU];
```



### s_start
```c
init_proc_pool()
// 初始化进程池
insert_to_ready_queue( load_user_program() );
schedule();
```

### load_user_program
原本在load_user_program中进行用户程序空间的内存映射。

用户程序的段映射集成到load_bincode_from_host_elf-->elf_load中
用户程序的其他内存分配，trapframe，pagetable，kstack，user_stack, userheap全部集成到alloc_process函数中。

### elf_load
从其中把elf_load_segment拆出来当一个函数，提高了可读性和可维护性。

### insert_to_ready_queue
将初始化好的进程控制块存入就绪进程队列。

### schedule
修改schedule函数，以支持多核的退出。
需要加入一个互斥锁，避免多个核心同时操作ready_queue_head

### alloc_process
为用户程序分配其他所有内存空间。
为什么不能在alloc_process中给用户程序初始化堆？那是因为和do_fork中子进程拷贝父进程堆冲突。
重点是堆内存的分配方式：在初始化的时候没有分配任何页。
也就是说，堆内存的初始化会在第一次malloc请求时执行。
```c
  // initialize the process's heap manager
  ps->user_heap.heap_top = USER_FREE_ADDRESS_START;
  ps->user_heap.heap_bottom = USER_FREE_ADDRESS_START;
  ps->user_heap.free_pages_count = 0;

  // map user heap in userspace
  ps->mapped_info[HEAP_SEGMENT].va = USER_FREE_ADDRESS_START;
  ps->mapped_info[HEAP_SEGMENT].npages = 0;  // no pages are mapped to heap yet.
  ps->mapped_info[HEAP_SEGMENT].seg_type = HEAP_SEGMENT;
```



### vmm.c
需要修改原有的双向链表堆实现，适配进程控制块中新的堆管理方式。
方便起见，我们预先给用户堆分配一个页，从而进行初始化。
这个预先给堆分配一个空页的实现是错的，因为会影响到fork中拷贝父进程的堆。
这一段堆初始化代码需要到第一次malloc时才能执行。
```c
  heap_block *user_heap = (heap_block *)Alloc_page();
  user_heap->size = PGSIZE - sizeof(heap_block);
  user_heap->prev = NULL;
  user_heap->next = NULL;
  user_heap->free = 1; // 初始块是空闲的
   user_vm_map(ps->pagetable, ps->user_heap.heap_bottom, PGSIZE,
              (uint64)user_heap, prot_to_type(PROT_WRITE | PROT_READ, 1)); 
```



不使用free_pages_count字段。
在malloc给用户堆分配新页中，ps->mapped_info[HEAP_SEGMENT].npages++;
在do_fork中，不需要考虑用户堆中的空闲页。

同样地，在malloc给用户栈分配新页时，也需要修改mapped_info。在handle_user_page_fault中，我们需要对于mapped_info做更新