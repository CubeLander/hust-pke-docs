# 华中科技大学 操作系统实验文档
## 一、 前言

本文档旨在继承和发扬开源精神，借助社区智慧，通过一系列实践探索为开源事业贡献微薄之力，希望能够为后续的研究与开发提供参考与借鉴。

需要特别说明的是，文档中部分内容由 ChatGPT/Deepseek 自动生成，尚未经过充分推敲和严格验证，可能存在不准确或不完善之处。对此，敬请读者谅解，并欢迎大家提出宝贵意见，共同改进和完善。

文档中使用vscode的搜索和c语言扩展功能，从而实现源代码中对关键字的准确查找。因为篇幅有限，故在引用函数和汇编符号时均省略目录和文件位置，希望读者自行搜索和确认。




## 二、实验概述

[实验概述](lab/实验概述.md)


## 三、参考资料

[riscv-software-src/riscv-pk: RISC-V Proxy Kernel](https://github.com/riscv-software-src/riscv-pk.git)

[MrShawCode/riscv-pke: RISC-V Proxy Kernel for Education](https://github.com/MrShawCode/riscv-pke)

[华中科技大学操作系统实验（riscv-pke）文档 - Gitee.com](https://gitee.com/hustos/pke-doc/tree/master)

[riscv-collab/riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain)

[riscv-software-src/riscv-isa-sim](https://github.com/riscv-software-src/riscv-isa-sim)

[百度网盘下载：riscv-gnu-toolchain](https://pan.baidu.com/s/1Z9xKV_UY2Li_SxYrbJT5Zw?pwd=cpbf)


## 四、环境搭建（Docker+vscode）
### 调试好上手即用的x64 ubuntu 24.04 LTS镜像
```
  docker pull crpi-x7y7w4q8rsfacqq9.cn-shanghai.personal.cr.aliyuncs.com/cubelander-images/x86-pke:latest
```
[配置过程](lab/环境配置.md)

### 调试工具的安装和使用

[调试工具安装配置过程](lab/调试工具.md)


## 五、实验内容(按完成顺序)

[lab1_1 系统调用](lab/lab1_1.md)

[lab1_2 硬件中断](lab/lab1_2.md)

[lab1_3 外部中断](lab/lab1_3.md)

[lab1_challenge1 打印用户程序调用栈](lab/lab1_challenge1.md)


[lab1_challenge2 打印异常代码行](lab/lab1_challenge2.md)

[lab1_challenge3 多核启动和运行](lab/lab1_challenge3.md)

[lab2 内存管理基础知识](lab/lab2.md)

[lab2_1 虚实地址转换](lab/lab2_1.md)

[lab2_2 简单内存分配](lab/lab2_2.md)

[lab2_3 缺页异常](lab/lab2_2.md)



[lab2_challenge1 复杂缺页异常](lab/lab2_challenge1.md)


## 六、文档作者的一些碎碎念

在操作系统开发的过程中，作者逐渐意识到，尽管C语言本身并不直接提供类和对象的语法支持，但通过合理的编程思想，依然可以借助面向对象的设计原则来组织代码。例如，可以将全局结构体（如进程控制块、内存管理单元（MMU）等）视为“类”的实例，借助结构体的封装特性，模拟对象的属性和行为。作者发现，这种方法不仅提升了代码的模块化、可扩展性，还使得复杂的操作系统组件更加易于管理和维护。这种思维方式的转变，使得开发过程变得更加清晰，并为后续的系统扩展提供了更加灵活的架构设计。

如果说将面向对象特性引入C源代码中，那么需要使用更现代的C设计模式，如Linux不透明指针。

这个实验的源代码是“纯草台班子”，仅有教学、演示和参考目的，没什么应用价值。