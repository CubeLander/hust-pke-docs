# lab1_challenge1 打印函数调用栈
## 实验目标
- 理解操作系统读取elf文件的方法
- 实现打印函数调用栈的系统调用功能，和对应的用户接口


## 主要内容
- elf文件模块的数据结构
- 系统读取elf文件，并切换到用户进程的实现过程
- 内存栈与函数栈帧的生命周期
- 实验目标的具体实现


## 引用文档
- [ELF文件概述](../doc/ELF文件概述.md)
- [ELF文件头解析](../doc/ELF文件头解析.md)
- [ELF函数返回值解析](../doc/ELF函数返回值解析.md)
- [ELF程序段头解析](../doc/ELF程序段头解析.md)
- [数据段、堆栈和内存](../doc/数据段、堆栈和内存.md)
- [lab1的内存管理漏洞](../doc/lab1的内存管理漏洞.md)
- [lab1是如何避免用户进程写入内核的内存区域的](../doc/lab1是如何避免用户进程写入内核的内存区域的.md)
- [RISC-V通用寄存器概述](../doc/RISC-V通用寄存器概述.md)
- [RISC-V函数帧解析](../doc/RISC-V函数帧解析.md)

## elf模块的数据结构
- [ELF文件概述](../doc/ELF文件概述.md)
### elf_ctx
```cpp
typedef struct elf_ctx_t {
  void *info;
  elf_header ehdr;
} elf_ctx;
```


### elf_info
- 保存spike文件指针，和进程控制块的位置
```cpp
typedef struct elf_info_t {
  spike_file_t *f;
  process *p;
} elf_info;
```

### elf_header ELF文件头结构体
- [ELF文件头解析](../doc/ELF文件头解析.md)
```cpp
typedef struct elf_header_t {
  uint32 magic;        /* Magic Number (ELF文件标识符，固定值0x7f 45 4c 46) */
  uint8 elf[12];       /* ELF标识符（包括架构类型、字节序等，12字节） */
  uint16 type;         /* 文件类型（如EXEC表示可执行文件） */
  uint16 machine;      /* 目标架构（如RISC-V表示RISC-V架构） */
  uint32 version;      /* 文件版本（通常为1，表示当前版本） */
  uint64 entry;        /* 程序入口地址（程序开始执行的位置） */
  uint64 phoff;        /* 程序头表在文件中的偏移量 */
  uint64 shoff;        /* 节头表在文件中的偏移量 */
  uint32 flags;        /* 处理器特定的标志位（表示特定架构的处理器特性） */
  uint16 ehsize;       /* ELF文件头大小（通常为64字节） */
  uint16 phentsize;    /* 每个程序头表项的大小（通常为56字节） */
  uint16 phnum;        /* 程序头表项的数量（例如4个程序头项） */
  uint16 shentsize;    /* 每个节头表项的大小（通常为64字节） */
  uint16 shnum;        /* 节头表项的数量（例如19个节头项） */
  uint16 shstrndx;     /* 节头字符串表的索引（指向节名称的字符串表） */
} elf_header;

```
### elf_prog_header ELF程序段头结构体
```cpp
// ELF程序段头结构体
typedef struct elf_prog_header_t {
  uint32 type;   /* 段类型（如代码段、数据段等） */
  uint32 flags;  /* 段的标志位（表示该段的特性，例如可读、可写、可执行） */
  uint64 off;    /* 段在文件中的偏移量（表示该段在ELF文件中的位置） */
  uint64 vaddr;  /* 段的虚拟地址（程序加载到内存中的位置） */
  uint64 paddr;  /* 段的物理地址（通常与vaddr相同，但对于某些硬件可能不同） */
  uint64 filesz; /* 段在文件中的大小（文件中的字节数） */
  uint64 memsz;  /* 段在内存中的大小（内存中的字节数） */
  uint64 align;  /* 段的对齐要求（通常是2的幂，用于内存对齐） */
} elf_prog_header;
```


### elf_status
- [ELF函数返回值解析](../doc/ELF函数返回值解析.md)
```cpp
typedef enum elf_status_t {
  EL_OK = 0,

  EL_EIO,
  EL_ENOMEM,
  EL_NOTELF,
  EL_ERR,

} elf_status;


```
## lab1源代码中加载elf模块的过程

### 解析过程源代码
`s_start();`-->  `load_user_program(&user_app);`-->`load_bincode_from_host_elf(proc);`-->`elf_init(&elfloader, &info);`-->`elf_load(&elfloader);`-->`p->trapframe->epc = elfloader.ehdr.entry;`
其中，在`load_bincode_from_host_elf(proc);`声明了两个关键局部变量：
```cpp
  //elf loading. elf_ctx is defined in kernel/elf.h, used to track the loading process.
  elf_ctx elfloader;
  // elf_info is defined above, used to tie the elf file and its corresponding process.
  elf_info info;
```
这两个变量贯穿了从spike打开到关闭用户程序文件，整个解析elf的过程。
#### void load_bincode_from_host_elf(process *p)
```cpp
//
// load the elf of user application, by using the spike file interface.
//
void load_bincode_from_host_elf(process *p) {
  arg_buf arg_bug_msg;
  // retrieve command line arguements
  size_t argc = parse_args(&arg_bug_msg);
  if (!argc) panic("You need to specify the application program!\n");
  sprint("Application: %s\n", arg_bug_msg.argv[0]);

  //elf loading. elf_ctx is defined in kernel/elf.h, used to track the loading process.
  elf_ctx elfloader;
  // elf_info is defined above, used to tie the elf file and its corresponding process.
  elf_info info;

  info.f = spike_file_open(arg_bug_msg.argv[0], O_RDONLY, 0);
  info.p = p;
  // IS_ERR_VALUE is a macro defined in spike_interface/spike_htif.h
  if (IS_ERR_VALUE(info.f)) panic("Fail on openning the input application program.\n");

  // init elfloader context. elf_init() is defined above.
  if (elf_init(&elfloader, &info) != EL_OK)
    panic("fail to init elfloader.\n");

  // load elf. elf_load() is defined above.
  if (elf_load(&elfloader) != EL_OK) panic("Fail on loading elf.\n");

  // entry (virtual, also physical in lab1_x) address
  p->trapframe->epc = elfloader.ehdr.entry;

  // close the host spike file
  spike_file_close( info.f );

  sprint("Application program entry point (virtual address): 0x%lx\n", p->trapframe->epc);
}
```

#### elf_status elf_init(elf_ctx *ctx, void *info)
- 从elf源文件中读取elf文件头数据结构
```cpp
//
// init elf_ctx, a data structure that loads the elf.
//
elf_status elf_init(elf_ctx *ctx, void *info) {
  ctx->info = info;

  // load the elf header
  if (elf_fpread(ctx, &ctx->ehdr, sizeof(ctx->ehdr), 0) != sizeof(ctx->ehdr)) return EL_EIO;

  // check the signature (magic value) of the elf
  if (ctx->ehdr.magic != ELF_MAGIC) return EL_NOTELF;

  return EL_OK;
}
```

#### static uint64 elf_fpread(elf_ctx *ctx, void *dest, uint64 nb, uint64 offset)
- 从ELF文件的offset字节开始，读nb个字节到内存中dest指针指向的位置
```cpp
//
// actual file reading, using the spike file interface.
//
static uint64 elf_fpread(elf_ctx *ctx, void *dest, uint64 nb, uint64 offset) {
  elf_info *msg = (elf_info *)(ctx->info);
  // call spike file utility to load the content of elf file into memory.
  // spike_file_pread will read the elf file (msg->f) from offset to memory (indicated by
  // *dest) for nb bytes.
  return spike_file_pread(msg->f, dest, nb, offset);
}
```

#### elf_status elf_load(elf_ctx *ctx)
- 解析ELF文件头，读取剩余部分
```cpp
elf_status elf_load(elf_ctx *ctx) {
  // 用来存放elf程序段头的临时变量
  elf_prog_header ph_addr;

  // 循环解析elf程序段头
  int i, off;
  for (i = 0, off = ctx->ehdr.phoff; i < ctx->ehdr.phnum; i++, off += sizeof(ph_addr)) {
    // 读取每一个elf程序段头
    if (elf_fpread(ctx, (void *)&ph_addr, sizeof(ph_addr), off) != sizeof(ph_addr)) return EL_EIO;

    if (ph_addr.type != ELF_PROG_LOAD) continue;
    if (ph_addr.memsz < ph_addr.filesz) return EL_ERR;
    if (ph_addr.vaddr + ph_addr.memsz < ph_addr.vaddr) return EL_ERR;

    // 为elf程序段分配内存
    void *dest = elf_alloc_mb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);

    // 将elf程序段写入内存
    if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz)
      return EL_EIO;
  }

  return EL_OK;
}
```

#### static void *elf_alloc_mb(elf_ctx *ctx, uint64 elf_pa, uint64 elf_va, uint64 size)
- 因为目前还没有引入虚拟内存，所以直接分配了物理内存。
```cpp
//
// the implementation of allocater. allocates memory space for later segment loading
//
static void *elf_alloc_mb(elf_ctx *ctx, uint64 elf_pa, uint64 elf_va, uint64 size) {
  // directly returns the virtual address as we are in the Bare mode in lab1_x
  return (void *)elf_va;
}
```

### 上述代码备注
我在阅读上述源代码时，发现为elf文件分配内存的代码直接使用了elf文件内各个段的虚拟地址作为物理地址，写进了内存区域中。
这让我感到非常疑惑：代理内核源代码也同样是作为elf文件，并把各个段的虚拟地址直接当做物理地址载入内存的。如果代理内核各个段的虚拟地址刚好和用户程序的重叠，那么用户程序就有可能直接写入内核的内存区域，从而造成程序崩溃。
- [lab1的内存管理漏洞](../doc/lab1的内存管理漏洞.md)
- [lab1是如何避免用户进程写入内核的内存区域的](../doc/lab1是如何避免用户进程写入内核的内存区域的.md)

## riscv函数帧结构
[RISC-V函数帧解析](../doc/RISC-V函数帧解析.md)
调用者的帧指针存放在当前栈的fp-16位置，当前函数的返回地址存放在fp-8位置。
从中断上下文中的epc，可以推断出触发中断的函数名（见下文）。
从栈帧中存放的，当前函数的返回地址，可以推断出当前函数在调用者程序中的入口位置，从而得到调用者的函数名。
由此可见，递归地解析帧指针fp，就能得到调用序列的帧指针地址。
如此确定和输出整个调用序列。

## 实现

为了便于调试，禁用了 `minit.c` 中的 `timerinit()` 函数。在 `user_lib.c` 中新增了打印寄存器状态的调试函数。

### 从PC地址定位函数名

`symbols` 中的 `value` 字段表示函数入口指令在内存中的位置。当前中断程序计数器（`epc`）与 `symbols` 中各函数入口位置比较，找到最大小于 `epc` 的 `symbol_value`，即为离 `epc` 最近的符号地址，从而获取函数名。获取函数名的过程依赖 ELF 文件，因此需要预先读取 ELF 文件并解析符号到函数名的映射关系。

### 系统调用实现

#### 初始化函数名符号表和字符串表

在 `elf.h` 中新增 `elf_section_header` 和 `elf_symbol` 结构体，用于存储解析后的 ELF 内容。在 `elf_load` 过程中，调用新函数 `load_function_name`，解析并加载函数名对应的 `symbols` 和函数名字符串。设置全局变量存储函数名符号表和字符串表，并在 `syscall.c` 中通过 `extern` 引用，供系统调用使用。

#### 编写系统调用用户接口

在 `user_lib.c` 中增加相应的用户接口函数，并在 `user_lib.h` 中声明。

#### 编写系统调用服务程序

在 `syscall.h` 中增加 `backtrace` 的系统调用号；在 `syscall.c` 中增加 `print_backtrace` 系统调用服务程序，从中断上下文解析调用栈并输出用户程序的调用过程。
