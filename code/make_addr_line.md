# make_addr_line

## 对代码的重新组织
不改变函数功能，仅仅增加其可读性
```c
/*
 * analyzis the data in the debug_line section
 *
 * the function needs 3 parameters: elf context, data in the debug_line section
 * and length of debug_line section
 *
 * make 3 arrays:
 * "process->dir" stores all directory paths of code files
 * "process->file" stores all code file names of code files and their directory path index of array "dir"
 * "process->line" stores all relationships map instruction addresses to code line numbers
 * and their code file name index of array "file"
 */
void make_addr_line(elf_ctx *ctx, char *debug_line, uint64 length)
{
  // 在debug_line之后为pDir,、pFile、和pLine分配空间。
  // 当然这些空间也可以作为独立的结构，预先在内存中分配好
  // 这么写可读性比较差，篇幅关系懒得改了
  char **pDir = (char **)((((uint64)debug_line + length + 7) >> 3) << 3);
  code_file *pFile = (code_file *)(pDir + 64);
  addr_line *pLine = (addr_line *)(pFile + 64);

  int directory_count = 0;
  int file_count = 0;
  int line_count = 0;
  char *debugline_offset = debug_line;
  while (debugline_offset < debug_line + length)
  {
    // 在编译的过程中，每一个编译单元，都会在debugline中形成它自身的文件名-目录表。
    // 所以说，在我们保存的文件名总表和目录总表中，会有重复的文件名或目录。
    // 这里的dir_base就记录了当前编译单元的目录，在目录总表pDir下的开始位置。
    // file_base同理记录了当前编译单元的文件，在文件总表pFile下的开始位置。
    int dir_base = directory_count;
    int file_base = file_count;

    // 从debug_line中解析debug_header
    debug_header *dh = (debug_header *)debugline_offset;
    debugline_offset += sizeof(debug_header);

    // 从debugline中解析所有directory_name， 直到出现两个'\0'作为终止记号
    while (*debugline_offset != 0)
    {
      pDir[directory_count++] = debugline_offset;
      while (*debugline_offset != 0)
        debugline_offset++;
      debugline_offset++;
    }
    debugline_offset++;

    while (*debugline_offset != 0)
    {
      // 为每个文件确认与文件名的对应关系
      pFile[file_count].file = debugline_offset;
      while (*debugline_offset != 0)
        debugline_offset++;
      debugline_offset++;

      // 这里的dir记录的是文件在当前编译单元中，对应的目录序号(从1开始)。所以说在保存它与目录总表的对应关系时，需要做一个序号转换(从dir_base开始)
      uint64 dir;
      read_uleb128(&dir, &debugline_offset);
      // sprint("%s.dirvalue = %d;\n", pFile[file_count].file, dir);
      // sprint("%s.recdir = %d;\n", pFile[file_count].file, dir - 1 + dir_base);
      pFile[file_count++].dir = dir - 1 + dir_base;
      read_uleb128(NULL, &debugline_offset);
      read_uleb128(NULL, &debugline_offset);
    }
    debugline_offset++;

    // 接下来解析的是指令与行的对应关系。
    // 由于时间精力有限，不做更多分析。
    addr_line regs;
    regs.addr = 0;
    regs.file = 1;
    regs.line = 1;
    // simulate the state machine op code
    for (;;)
    {
      uint8 op = *(debugline_offset++);
      switch (op)
      {
      case 0: // Extended Opcodes
        read_uleb128(NULL, &debugline_offset);
        op = *(debugline_offset++);
        switch (op)
        {
        case 1: // DW_LNE_end_sequence
          if (line_count > 0 && pLine[line_count - 1].addr == regs.addr)
            line_count--;
          pLine[line_count] = regs;
          pLine[line_count].file += file_base - 1;
          line_count++;
          goto endop;
        case 2: // DW_LNE_set_address
          read_uint64(&regs.addr, &debugline_offset);
          break;
        // ignore DW_LNE_define_file
        case 4: // DW_LNE_set_discriminator
          read_uleb128(NULL, &debugline_offset);
          break;
        }
        break;
      case 1: // DW_LNS_copy
        if (line_count > 0 && pLine[line_count - 1].addr == regs.addr)
          line_count--;
        pLine[line_count] = regs;
        pLine[line_count].file += file_base - 1;
        line_count++;
        break;
      case 2:
      { // DW_LNS_advance_pc
        uint64 delta;
        read_uleb128(&delta, &debugline_offset);
        regs.addr += delta * dh->min_instruction_length;
        break;
      }
      case 3:
      { // DW_LNS_advance_line
        int64 delta;
        read_sleb128(&delta, &debugline_offset);
        regs.line += delta;
        break;
      }
      case 4: // DW_LNS_set_file
        read_uleb128(&regs.file, &debugline_offset);
        break;
      case 5: // DW_LNS_set_column
        read_uleb128(NULL, &debugline_offset);
        break;
      case 6: // DW_LNS_negate_stmt
      case 7: // DW_LNS_set_basic_block
        break;
      case 8:
      { // DW_LNS_const_add_pc
        int adjust = 255 - dh->opcode_base;
        int delta = (adjust / dh->line_range) * dh->min_instruction_length;
        regs.addr += delta;
        break;
      }
      case 9:
      { // DW_LNS_fixed_advanced_pc
        uint16 delta;
        read_uint16(&delta, &debugline_offset);
        regs.addr += delta;
        break;
      }
        // ignore 10, 11 and 12
      default:
      { // Special Opcodes
        int adjust = op - dh->opcode_base;
        int addr_delta = (adjust / dh->line_range) * dh->min_instruction_length;
        int line_delta = dh->line_base + (adjust % dh->line_range);
        regs.addr += addr_delta;
        regs.line += line_delta;
        if (line_count > 0 && pLine[line_count - 1].addr == regs.addr)
          line_count--;
        pLine[line_count] = regs;
        pLine[line_count].file += file_base - 1;
        line_count++;
        break;
      }
      }
    }
  endop:;
  }
  // 在进程控制块中保存解析结果
  process *p = ((elf_info *)ctx->info)->p;
  p->debugline = debug_line;
  p->dir = pDir;
  p->file = pFile;
  p->line = pLine;
  p->line_ind = line_count;

  // for (int i = 0; i < p->line_ind; i++)
  // sprint("%p %d %d\n", p->line[i].addr, p->line[i].line, p->line[i].file);
}


```

## make_addr_line函数详细解析

这个 `make_addr_line` 函数是用来解析 ELF 文件中的 `debug_line` 节段，提取源代码的行号信息，并根据调试信息将机器指令地址与源代码行号进行映射。具体来说，它会将每个源代码文件的路径、文件名以及每个指令地址对应的行号信息保存在 `process` 结构体中。

### 函数详细解析：

#### 1. **参数说明**：
   - `elf_ctx *ctx`：ELF 文件的上下文，通常包含有关 ELF 文件的信息。
   - `char *debug_line`：指向 `debug_line` 节段的指针，`debug_line` 存储了调试信息中的行号映射数据。
   - `uint64 length`：`debug_line` 节段的长度，指示要读取的字节数。

#### 2. **主要目标**：
   - 函数通过解析 `debug_line` 节段，将源代码文件的路径和文件名存储在 `process->dir` 和 `process->file` 数组中。
   - 同时，函数将指令地址和源代码行号的映射关系存储在 `process->line` 数组中，以便在调试过程中能够通过地址查找到源代码行号。

#### 3. **函数结构与流程**：

   1. **初始化数组**：
      - `process->dir` 用来存储目录路径。
      - `process->file` 用来存储文件名及其对应的目录路径索引。
      - `process->line` 用来存储地址与源代码行号的映射关系。

   2. **目录和文件路径的解析**：
      - 通过 `off` 指针读取 `debug_line` 节段中的目录路径和文件名信息。
      - 每个 `CU`（编译单元）内的目录和文件路径以 null 字符 `\0` 结尾，`process->dir` 和 `process->file` 数组会分别保存这些路径和文件名。

   3. **行号信息的解析**：
      - 通过 `debug_header` 结构体读取 `debug_line` 节段的头部信息，然后解析每个编译单元中的行号信息。
      - 使用状态机模拟调试器操作，根据不同的操作码（opcode）调整程序计数器（`regs.addr`）和行号（`regs.line`）。

   4. **状态机与操作码**：
      - 函数中使用了多种调试操作码（如 `DW_LNE_end_sequence`，`DW_LNS_advance_pc`，`DW_LNS_advance_line` 等）来解析行号与机器指令的关系。
      - 每个操作码执行不同的操作，比如更新程序计数器、行号、文件等信息。
      - 对于某些操作（如 `DW_LNE_set_address`），通过读取 ULEB128 值或固定值，更新地址信息。

   5. **更新行号信息**：
      - 根据当前地址与源代码的行号，更新 `process->line` 数组中的对应项。

#### 4. **细节说明**：
   - `read_uleb128`：函数用于解码 ULEB128 编码的数据，通常用于解析调试信息中的整数值。
   - `read_sleb128`：用于解码带符号的 ULEB128 编码（即 SLEB128），用于解析带符号的增量值（例如行号的变化）。
   - **扩展操作码（Extended Opcodes）**：这些操作码通常是特殊指令，处理特定的调试任务，如结束序列、设置地址等。

#### 5. **操作码解析**：
   - `DW_LNE_end_sequence`：表示当前的序列结束，更新行号映射，并可能删除最后一条记录。
   - `DW_LNE_set_address`：设置当前的指令地址。
   - `DW_LNS_copy`：复制当前的行号信息。
   - `DW_LNS_advance_pc`：根据增量更新程序计数器（指令地址）。
   - `DW_LNS_advance_line`：根据增量更新行号。
   - `DW_LNS_set_file`：设置当前源文件。

#### 6. **结束处理**：
   - 解析完所有编译单元后，函数会完成 `process->line` 数组的填充，记录每个地址对应的文件和行号信息。

### 总结：
- **目标**：函数的主要目的是解析调试信息中的 `debug_line` 节段，将指令地址与源代码的行号进行映射，并记录源代码的目录路径和文件名。
- **数据结构**：通过 `process->dir`、`process->file` 和 `process->line` 来分别保存目录、文件和行号信息。
- **操作码解析**：使用了一系列的调试操作码（如 `DW_LNS_copy`、`DW_LNE_end_sequence` 等）来处理行号和指令地址的关系。

该函数对于调试器或其他需要调试信息的工具非常重要，它能够将机器代码中的指令地址与源代码行号进行映射，帮助开发者更好地理解程序的执行过程。