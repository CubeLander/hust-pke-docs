在Linux RISC-V架构中，硬件中断处理的相关代码分布在几个关键位置。以下是查找和理解这些代码的指南：

## 1. 中断初始化和入口点

主要文件：
- `arch/riscv/kernel/irq.c` - 中断控制器初始化和通用中断处理
- `arch/riscv/kernel/entry.S` - 异常和中断的底层入口点

## 2. 中断向量和处理函数

- `arch/riscv/kernel/trap.c` - 包含`do_IRQ()`函数和异常处理程序
- `arch/riscv/kernel/traps.c` - 异常处理初始化和通用异常处理函数

## 3. 平台特定中断控制器驱动

- `drivers/irqchip/` - 包含各种中断控制器的驱动程序
- `drivers/irqchip/irq-riscv-*.c` - RISC-V特定的中断控制器实现
- `drivers/irqchip/irq-sifive-plic.c` - SiFive PLIC（平台级中断控制器）的驱动

## 4. 内存映射I/O (MMIO) 相关代码

- `arch/riscv/mm/init.c` - 内存和I/O映射初始化
- `arch/riscv/include/asm/io.h` - 定义MMIO访问的基本函数
- `drivers/base/iomap.c` - 通用I/O映射函数实现

## 5. 设备树解析（用于MMIO地址分配）

- `drivers/of/address.c` - 从设备树中提取MMIO地址范围
- `arch/riscv/kernel/setup.c` - RISC-V特定的设备树设置

## 如何追踪中断处理流程

1. 从启动流程开始：
   - `start_kernel()` → `init_IRQ()`（在`init/main.c`中）
   - RISC-V特定初始化：`riscv_init_irq()`（在`arch/riscv/kernel/irq.c`中）

2. 研究中断入口点：
   - 查看`arch/riscv/kernel/entry.S`中的`handle_exception`和`handle_irq_mode_[x]`函数

3. 跟踪中断处理函数：
   - 从`do_IRQ()`（在`arch/riscv/kernel/trap.c`中）开始
   - 了解通用中断处理框架（在`kernel/irq/handle.c`中的`generic_handle_irq()`）

4. 查看特定设备驱动：
   - 如键盘：`drivers/input/keyboard/`
   - 网络：`drivers/net/`
   - 存储：`drivers/block/`或`drivers/scsi/`

## 搜索技巧

在Linux源代码中查找中断相关代码时，可以使用以下关键词：
- `handle_irq`
- `do_IRQ`
- `request_irq`
- `irq_set_handler`
- `readl`/`writel`（用于MMIO访问）
- `ioremap`（用于设置MMIO映射）
- `mcause`（RISC-V特定中断原因寄存器）
- `CSR_`（控制状态寄存器宏）

通过grep命令搜索，例如：
```bash
grep -r "do_IRQ" arch/riscv/
```

或使用GitHub/GitLab等代码查看工具的搜索功能来定位这些函数和它们的调用关系。