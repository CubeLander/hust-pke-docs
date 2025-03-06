### **🔍 `exec()` 需要清理哪些资源？保留哪些资源？**
`exec()` 的核心目标是 **用新的可执行文件替换当前进程的代码段、数据段和堆栈，同时尽可能保持进程的其他环境不变**。这意味着：
- **必须清理的资源**：地址空间、信号处理程序等。
- **必须保留的资源**：文件描述符（除 `FD_CLOEXEC`）、PID、父进程关系等。

---

## **🌟 `exec()` 具体清理哪些资源？**
### **📌 1. 进程地址空间（必须清理 ✅）**
**`exec()` 需要完全清理旧的进程地址空间，并加载新的 ELF 可执行文件。**
```c
// 释放进程的地址空间
exec_cleanup_memory(current);
```
**要清理的内容：**
- **代码段（`.text`）**
- **数据段（`.data/.bss`）**
- **堆（`brk`）**
- **栈**
- **映射的文件（`mmap`）**

✅ **可复用 `free_process()` 的 `exec_cleanup_memory()` 部分**：
```c
void exec_cleanup_memory(struct task_struct *task) {
    if (task->mm) {
        struct mm_struct *mm = task->mm;
        struct vm_area_struct *vma = mm->mmap;
        while (vma) {
            struct vm_area_struct *next = vma->vm_next;
            free_page(vma->vm_start);
            kfree(vma);
            vma = next;
        }
        kfree(mm);
        task->mm = NULL;
    }
}
```
**🔸 但不能释放 `task_struct`，进程本体仍然存在！**

---

### **📌 2. 信号处理程序（必须清理 ✅）**
- `exec()` **必须重置所有 `sigaction` 处理程序**（防止 `exec()` 之后执行旧的信号处理逻辑）。
- 但是 **信号屏蔽集（`sigprocmask`）会保留**！

✅ **要清理的：**
```c
// 重置所有信号处理程序
for (int i = 0; i < MAX_SIGNAL; i++) {
    task->sigaction[i].sa_handler = SIG_DFL;
}
```
✅ **要保留的（不能清理）：**
```c
// 进程的信号屏蔽集仍然有效
// sigprocmask 继承，不需要清理
```
---

### **📌 3. 文件描述符（部分清理 ✅）**
- **`exec()` 必须关闭 `FD_CLOEXEC` 标记的文件**。
- **但普通文件描述符必须保留**（标准输入/输出、管道等）。

✅ **要清理的：**
```c
// 关闭 `FD_CLOEXEC` 标记的文件
for (int fd = 0; fd < MAX_FD; fd++) {
    if (task->files->fd_table[fd] && is_cloexec(fd)) {
        filp_close(task->files->fd_table[fd]);
        task->files->fd_table[fd] = NULL;
    }
}
```
✅ **要保留的：**
```c
// 普通文件描述符（无 `FD_CLOEXEC`）仍然保留
```
---

### **📌 4. 进程 ID（必须保留 ✅）**
- `exec()` **不会改变进程 ID**，即：
  - `getpid()` **在 `exec()` 之后仍然返回相同的 PID**。
  - `getppid()` **父进程 ID 也不会改变**。

✅ **要保留的（不能清理）：**
```c
// 进程 ID 和父进程关系仍然不变
task->pid = old_pid;
task->parent = old_parent;
```
---

### **📌 5. 进程的 `wait_queue_t`（必须保留 ✅）**
- 进程 `exec()` 之后仍然可以被 `wait()` 回收，因此 **不能清理 `wait_queue_t`**。
- 但 **进程处于 `TASK_RUNNING`，不会阻塞 `wait()`**。

✅ **要保留的（不能清理）：**
```c
// 进程 `wait_queue_t` 不能被清理
```
---

### **📌 6. 进程调度信息（必须保留 ✅）**
- 进程的 **优先级（priority）** 不能被修改。
- 进程的 **时间片** 不能被清空。

✅ **要保留的（不能清理）：**
```c
// 进程调度参数仍然有效
task->priority = old_priority;
task->time_slice = old_time_slice;
```
---

### **📌 7. 进程间通信（部分清理 ✅）**
- **共享内存（`shm`）仍然有效**（不能清理）。
- **管道 `pipe` 仍然有效**（不能清理）。
- **消息队列 `mq` 仍然有效**（不能清理）。

✅ **要保留的（不能清理）：**
```c
// 共享内存、管道、消息队列仍然保留
```
---

## **🔥 `exec()` 资源清理代码**
综合以上，完整的 `exec_cleanup()` 代码如下：
```c
void exec_cleanup(struct task_struct *task) {
    // 1. 释放进程地址空间（代码段、数据段、堆、栈）
    exec_cleanup_memory(task);

    // 2. 重置所有信号处理程序（但保留信号屏蔽集）
    for (int i = 0; i < MAX_SIGNAL; i++) {
        task->sigaction[i].sa_handler = SIG_DFL;
    }

    // 3. 关闭 `FD_CLOEXEC` 文件描述符，但保留普通 FD
    for (int fd = 0; fd < MAX_FD; fd++) {
        if (task->files->fd_table[fd] && is_cloexec(fd)) {
            filp_close(task->files->fd_table[fd]);
            task->files->fd_table[fd] = NULL;
        }
    }

    // 4. 保留进程 ID，不释放 `task_struct`
    task->pid = task->pid;
    task->parent = task->parent;

    // 5. 进程 `wait_queue_t` 不能清理（允许 `wait()`）
    // 6. 进程调度信息仍然有效
    task->priority = task->priority;
    task->time_slice = task->time_slice;

    // 7. 共享内存、管道、消息队列仍然有效
}
```
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