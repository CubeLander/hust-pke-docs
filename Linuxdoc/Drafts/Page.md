Linux内核使用`struct page`结构体来表示物理页的元数据。这个结构体在Linux内核中至关重要，它为每个物理页面维护状态信息。以下是`struct page`的核心结构（基于较新的Linux内核版本）：

```c
struct page {
    unsigned long flags;     /* 原子标志，页面状态标记 */
    
    /* 
     * 五种不同使用情况的共用体:
     * 1. 匿名映射时的映射信息
     * 2. 文件映射时的地址空间
     * 3. slab分配器使用信息
     * 4. 页表页使用信息
     * 5. 移动页使用信息
     */
    union {
        struct {
            struct list_head lru;  /* 用于页面回收的LRU链表 */
            struct address_space *mapping;  /* 所属的地址空间 */
            pgoff_t index;         /* 在映射中的索引 */
            unsigned long private; /* 私有数据指针 */
        };
        struct {
            struct list_head slab_list; /* slab分配器链表 */
            struct kmem_cache *slab_cache; /* slab所属缓存 */
            void *freelist;        /* 空闲对象链表 */
        };
        struct {
            struct page_pool *pp;  /* 页池指针 */
            struct device *dev;    /* 设备结构 */
        };
        /* ... 其他专用类型 ... */
    };

    union {
        atomic_t _mapcount;    /* 页表映射计数 */
        unsigned int active;   /* SLAB活跃对象 */
        int units;             /* 设备中的单元 */
    };
    
    atomic_t _refcount;    /* 引用计数，跟踪页面使用者数量 */
    
    /* 用于复合页(compound page)的字段 */
    struct page *first_page;  /* 指向复合页首页的指针 */
    unsigned long compound_head; /* 复合页首页指针编码，复合页中所有页面共享 */
    
    /* 用于页表、设备I/O和文件页的其他字段... */
};
```

这个结构体的特点和使用方式：

1. **内存高效**：尽管看起来很复杂，但Linux内核采用了许多技巧使每个`struct page`尽可能小，因为系统中通常有数百万个页面。

2. **多功能设计**：使用联合体（union）以不同方式重用相同的内存空间，根据页面当前的用途适配不同场景。

3. **引用计数**：`_refcount`字段跟踪页面被使用的次数，当计数归零时页面可以被释放。

4. **映射计数**：`_mapcount`跟踪页面被映射到页表中的次数。

5. **页面状态**：`flags`字段包含页面状态标志，如`PG_locked`（页面锁定）、`PG_dirty`（页面脏）等。

6. **复合页支持**：系统可以将多个连续页面组合成一个"复合页"以支持大页（huge pages）等功能。

7. **内存压缩**：在64位架构中，通过各种巧妙的位操作和指针压缩技术减小结构体大小。

Linux内核中，这个结构体会被组织到全局数组`mem_map`中，或者在NUMA系统中分布在每个内存节点的`node_mem_map`数组中，允许通过物理页号快速索引到对应的`struct page`结构。