# lab4_challenge2 重载执行

## 实验目标

建立`exec(exectuable path)`系统调用

## 实验原理

fork：创建一个新进程，复制当前进程的内存空间、文件描述符、堆栈等，返回值不同：父进程返回新进程的 PID，子进程返回 0。

execve：在当前进程中加载并执行一个新的程序，替换当前进程的内存空间和上下文，但保持进程ID（PID）不变。新程序直接进main入口。

你提到的 重载执行 其实是指在已有进程的上下文中加载新程序。这并不涉及创建一个新进程（这是 fork 的工作），但在实现时，它是替换当前进程的内存和状态。因此，execve 通常会和 fork 配合使用。

## 内容
exec中需要释放内存中所有的旧资源（所有段，包括内核管理的数据结构等）
取消挂起的信号处理程序

解析和加载elf，重新执行加载用户程序并执行的过程

同时pid,ppid,cwd可能被保留

exec对于信号的影响？
在 exec 之前，进程可能通过 sigprocmask 设定了信号屏蔽集（即哪些信号暂时不能被处理）。在 exec 之后，屏蔽集会被保留，所以如果某些信号在 exec 之前被屏蔽，在 exec 之后仍然会被屏蔽。

	作者云，所以说，进程的资源需要通过pcb来统一管理


我们需要拆开解耦host elf的代码。

## 处理流程

### 进程目前的各个资源

#### PCB

注释中只讨论exec有影响的内容。

```c
typedef struct process_t {
  uint64 kstack;
  pagetable_t pagetable;	// 需要手动做map/unmap
  trapframe* trapframe;
  trapframe* ktrapframe;

  // added @lab1_challenge2 这些事实上需要映射elf文件的debug段，最后一块实现
  char *debugline;
  char **dir;
  code_file *file;
  addr_line *line;
  int line_count;

  uint64 user_stack_bottom;
		// 随用户栈变化的，需要重置

  mapped_region *mapped_info;
  	// 需要换掉里面的程序
  int total_mapped_region;
  // 内容有变化？用户的堆栈和代码数据都要重置，堆重置会-1
  process_heap_manager user_heap;
	// 用户堆需要清空
  uint64 pid;
  int status;
  struct process_t *parent;
  struct process_t *queue_next;	
  int tick_count;
  int sem_index;
  // 

  proc_file_management *pfiles;
  // 保留进程的文件资源
  // 局部变量丢了，再用这些文件资源有什么用？
  // 比如说保留与父进程通信的管道（被动），保留从终端的输入。
  // 可以手动设置文件是否被exec继承
  // 另外，在进程终止时，会关闭和释放进程持有的全部文件资源。
  // 进程对于文件的使用不一定是完全互斥的，多个进程可以同时读一个文件。

}process;
```

#### mapped_region
```c
// the VM regions mapped to a user process
typedef struct mapped_region {
  uint64 va;       // mapped virtual address
  uint32 npages;   // mapping_info is unused if npages == 0
  uint32 seg_type; // segment type, one of the segment_types
} mapped_region;

// types of a segment
enum segment_type {
  STACK_SEGMENT = 0,   // runtime stack segmentm from init_user_stack
  // 重置用户栈
  CONTEXT_SEGMENT, // trapframe segment, from alloc_process
  SYSTEM_SEGMENT,  // system segment,from alloc_process
  HEAP_SEGMENT,    // runtime heap segment, from init_user_heap
  // 重置用户堆
  CODE_SEGMENT,    // ELF segmentm from elf_load_segment
  // 重置代码
  DATA_SEGMENT,    // ELF segment, from elf_load_segment
  // 重置全局数据段
};
```

#### process_heap_manager
```c
typedef struct process_heap_manager {
  // points to the last free page in our simple heap.
  uint64 heap_top;
  // points to the bottom of our simple heap.
  uint64 heap_bottom;

  // the address of free pages in the heap
  // uint64 free_pages_address[MAX_HEAP_PAGES];
  // the number of free pages in the heap
  // uint32 free_pages_count;
}process_heap_manager;
```




## exec需要清理哪些资源？

### **🔍 `exec()` 需要清理哪些资源？保留哪些资源？**
`exec()` 的核心目标是 **用新的可执行文件替换当前进程的代码段、数据段和堆栈，同时尽可能保持进程的其他环境不变**。这意味着：
- **必须清理的资源**：地址空间、信号处理程序等。
- **必须保留的资源**：文件描述符（除 `FD_CLOEXEC`）、PID、父进程关系等。

---

## **🎯 总结**
| 资源 | `exec()` 是否清理？ | 备注 |
|------|------------------|------|
| **地址空间（代码/数据/堆/栈）** | ✅ **必须清理** | 旧代码必须释放，加载新 ELF |
| **文件描述符（FD）** | 🔸 **部分清理** | 关闭 `FD_CLOEXEC`，保留普通 FD |
| **信号处理程序** | ✅ **必须清理** | `sigaction` 设为 `SIG_DFL` |
| **信号屏蔽集** | ❌ **保留** | `sigprocmask` 继承 |
| **进程 ID（PID）** | ❌ **保留** | `getpid()` 不变 |
| **父进程 ID（PPID）** | ❌ **保留** | `getppid()` 不变 |
| **`wait_queue_t`** | ❌ **保留** | 允许 `wait()` |
| **进程调度信息** | ❌ **保留** | `priority`、`time_slice` 仍然有效 |
| **共享内存/管道/IPC** | ❌ **保留** | `shm` / `pipe` 仍然有效 |

你现在 `exec()` 里遇到哪些资源释放的问题？🚀🔥



## 进程持有哪些资源？

### 内核态资源

PCB

用户页表

用户mm资源表

等待队列（没有实现）

内核栈

用来实现wait的信号量

内核使用的文件表

在调度器中的调度状态

进程间通信，和网络资源
共享内存（shmget/shmat）。
信号量（sem_t）。
管道（pipe）。
套接字（socket）。

### 用户态资源

数据段

代码段

堆

栈

用户信号量