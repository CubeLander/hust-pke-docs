# lab3_challenge1 wait

## 实验要求
为用户程序实现wait系统调用
wait系统调用需要实现的功能：
    - 当pid为-1时，父进程等待任意一个子进程退出，然后返回子进程的pid；
    - 如果父进程没有子进程，直接返回-1;
    - 当pid大于0时，父进程等待pid指定的子进程退出，然后返回子进程的pid；
    - 对于pid不符合要求，或者不是该进程的子进程，返回-1;

同时补充do_fork函数，实现其中子进程数据段的复制。
实验不能轮询等待，需要将父进程存入某个阻塞队列，等待信号。

## 实现原理
为每个子进程分配一个信号量：当父进程创建一个子进程时，为子进程分配一个信号量，并将其保存在子进程的控制块中。
父进程等待子进程：父进程通过 wait(pid) 系统调用查找指定的子进程，并执行 sem_P 操作等待子进程完成。父进程会阻塞，直到子进程执行完毕。
子进程终止时通知父进程：子进程终止时，通过 sem_V 操作释放父进程的信号量，从而使父进程继续执行。


wait(-1)的实现中，子进程在退出时，通过指向父进程的指针做`V(sem_parent)`，这样任意的子进程退出事件都可以唤醒父进程。
但是父进程仍然需要遍历一遍自己的子进程列表，才能查找那个终止的子进程，并返回pid。

## 实现

在进程数据结构中增加一个信号量index，并在初始化过程中获取。

```c
typedef struct process_t {
...
  int sem_index;
...

}process;
```

### alloc_process
```c
process *alloc_process() {
...
	ps->sem_index = sem_new(0,ps->pid);
...
}
```

### free_process
在这里我们需要销毁进程对应的信号量，但是由于代理内核结构简单，还不需要考虑内存泄漏，故没有实现。

## sys_user_wait
```c
int sys_user_wait(int pid) {
  int hartid = read_tp();
  int child_found_flag = 0;
  if (pid == -1) {
		sem_P(current[hartid]->sem_index);
  }
  if (0 < pid && pid < NPROC) {
    process *p = &procs[pid];
    if (p->parent != current[hartid]) {
      return -1;
    } else {
			sem_P(p->sem_index);
      return pid;
    }
  }
  return -1;
}

```

## sys_user_exit


