# 我们的linux系统总设计

## PMM PhysicalMemoryManager 物理页分配器
在启动时计算和分配，物理页元数据表。
支持分配单页和释放单页，和引用计数
简单使用链表


## VMM VirtualMemoryManager 虚拟页分配器
与进程控制块中的mm_struct互动。



### 通用内存分配
- **kmalloc(size, flags)** - 分配连续的物理内存
- **kfree(ptr)** - 释放通过kmalloc分配的内存



## Pagetable 页表