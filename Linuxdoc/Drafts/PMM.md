Linux内核的物理页分配器提供了一系列接口，以下是一些核心接口的声明：

## 页面分配接口

```c
// 分配单个页面
struct page *alloc_page(gfp_t gfp_mask);

// 分配2^order个连续页面
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);

// 分配指定页面数的连续物理内存并返回其虚拟地址
void *__get_free_pages(gfp_t gfp_mask, unsigned int order);

// 分配单个页面并返回其虚拟地址
void *__get_free_page(gfp_t gfp_mask);

// 分配小内存块
void *kmalloc(size_t size, gfp_t flags);
```

## 页面释放接口

```c
// 释放由alloc_page(s)分配的页面
void __free_page(struct page *page);
void __free_pages(struct page *page, unsigned int order);

// 释放由__get_free_page(s)分配的内存
void free_page(unsigned long addr);
void free_pages(unsigned long addr, unsigned int order);

// 释放kmalloc分配的内存
void kfree(const void *);
```

## 页面预留接口

```c
// 预留特定区域的物理内存
int reserve_pages(phys_addr_t start, phys_addr_t end);

// 释放预留的物理内存
void free_reserved_area(void *start, void *end, int poison, const char *s);
```

## 内存信息查询接口

```c
// 获取内存区域信息
unsigned long nr_free_pages(void);
int nr_free_pages_zone(int zone);

// 获取整体内存使用信息
void si_meminfo(struct sysinfo *val);
```

## 内存分配控制接口

```c
// 设置内存分配掩码
gfp_t gfp_allowed_mask;

// 内存分配失败处理
void *memalloc_noio_save(void);
void memalloc_noio_restore(void *);
```

在Linux中，物理页分配的GFP（Get Free Page）标志控制分配行为，常见的有：

- `GFP_KERNEL`：可能休眠的内核空间分配
- `GFP_ATOMIC`：不可休眠的中断上下文分配
- `GFP_USER`：用户空间分配
- `GFP_HIGHUSER`：用户空间分配，使用高区内存
- `__GFP_ZERO`：分配后填充零值

Linux使用伙伴系统（buddy system）来管理物理页，并结合slab分配器处理小内存块分配。内核不直接使用引用计数来管理物理页面的生命周期，而是使用更复杂的页面管理结构（`struct page`），其中包含引用计数字段`_refcount`来跟踪页面的使用情况，防止过早释放。