# lab1_challenge2 打印异常代码行

## 引用
[lab1_challenge1 打印用户程序调用栈](lab/lab1_challenge1.md)

[ELF文件如何组织调试信息](../doc/ELF文件如何组织调试信息.md)

[make_addr_line源代码分析](../code/make_addr_line.md)

[硬中断的响应过程](../doc/硬件对硬中断的响应过程.md)

[硬中断处理程序](../code/硬中断处理程序.md)

## 实验目标
修改内核（包括machine文件夹下）的代码，使得用户程序在发生异常时，内核能够输出触发异常的用户程序的源文件名和对应代码行。

示例程序和对应输出如下：
```c
int main(void) {
  printu("Going to hack the system by running privilege instructions.\n");
  // we are now in U(user)-mode, but the "csrw" instruction requires M-mode privilege.
  // Attempting to execute such instruction will raise illegal instruction exception.
  asm volatile("csrw sscratch, 0");
  exit(0);
}
/*
...
Switch to user mode...
Going to hack the system by running privilege instructions.
Runtime error at user/app_errorline.c:13
  asm volatile("csrw sscratch, 0");
Illegal instruction!
System is shutting down with exit code -1.
...
*/
```

## 主要内容
lab1_challenge2的分支变化

打印异常代码行的原理

## lab1_challenge2的分支变化
### `elf.c`
```cpp
void read_uleb128(uint64 *out, char **off);
void read_sleb128(int64 *out, char **off);
void read_uint64(uint64 *out, char **off);
void read_uint32(uint32 *out, char **off);
void read_uint16(uint16 *out, char **off);
void make_addr_line(elf_ctx *ctx, char *debug_line, uint64 length);
```
关键新增函数是make_addr_line，它实现了从elf解析debugline段，从而提取目录表、文件表和行表的过程。
### `elf.h`
```c
// elf section header
typedef struct elf_sect_header_t{
    uint32 name;
    uint32 type;
    uint64 flags;
    uint64 addr;
    uint64 offset;
    uint64 size;
    uint32 link;
    uint32 info;
    uint64 addralign;
    uint64 entsize;
} elf_sect_header;

// compilation units header (in debug line section)
typedef struct __attribute__((packed)) {
    uint32 length;
    uint16 version;
    uint32 header_length;
    uint8 min_instruction_length;
    uint8 default_is_stmt;
    int8 line_base;
    uint8 line_range;
    uint8 opcode_base;
    uint8 std_opcode_lengths[12];
} debug_header;
```

### process.h
```c
// code file struct, including directory index and file name char pointer
typedef struct {
    uint64 dir; char *file;
} code_file;

// address-line number-file name table
typedef struct {
    uint64 addr, line, file;
} addr_line;

// the extremely simple definition of process, used for begining labs of PKE
typedef struct process_t {
  // pointing to the stack used in trap handling.
  uint64 kstack;
  // trapframe storing the context of a (User mode) process.
  trapframe* trapframe;

  // added @lab1_challenge2
  char *debugline;
  char **dir;
  code_file *file;
  addr_line *line;
  int line_ind;
}process;
```

## 打印异常代码的实现原理

### elf文件中的调试信息是怎么组织的
[ELF文件如何组织调试信息](../doc/ELF文件如何组织调试信息.md)

在 ELF 文件中，调试信息通过 `.debug_*` 节段（如 `.debug_info`、`.debug_abbrev`、`.debug_line` 等）来组织和存储。DWARF 格式被广泛使用来表示这些信息，帮助调试器将二进制代码映射回源代码。调试信息包含了源代码的行号、符号表、变量和类型信息，但**不包含源代码的实际文本**。这些信息帮助开发者在调试时查看源代码的变量值、函数调用栈、代码行等。

具体二进制组织方式，见函数`make_addr_line`中的解析过程。

## 实现
### 读取.debug_line ELF节
[lab1_challenge1 打印用户程序调用栈](lab/lab1_challenge1.md)

在读取调试信息的过程中加入对`.debug_line` ELF节的解析过程。
解析ELF信息的方法，见`lab1_challenge1`

### 重新组织make_addr_line的代码
见 [make_addr_line源代码分析](../code/make_addr_line.md)

由于作者水平有限，函数原变量名和内部结构很难看懂，于是重新做了一番研究。
### 全局数据结构
在`elf.c`中声明了全局debugline缓冲区，从中分配`dir`表、`file`表和`line`表.
### 从epc到代码行的解析过程
在mtrap.c中新建一个硬中断服务程序error_printer，读取process中存放的文件名表dir，文件表file和行表line成员变量，进行epc到代码行的定位过程。

查找行表（递增），从而根据映射关系定位目标文件路径在`dir`表和`file`表中的序号，从而还原源文件的路径。

根据存放的路径重新打开文件，通过读取换行符\n定位目标行的代码文本。
