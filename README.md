# 华中科技大学 操作系统实验文档

## 一、前言

本文档旨在记录我在完成 [华中科技大学 操作系统课程设计（Gitee）](https://gitee.com/hustos/pke-doc/tree/master)，基于 [课程源代码仓库(Github)](https://github.com/MrShawCode/riscv-pke) 学习时，所遇到的困难及其解决方案，特别是在开发基于RISC-V代理内核（Proxy Kernel）的操作系统过程中积累的具体经验。文档内容不仅包括对原文档的补充，还针对具体问题提供了详细的解决方案，旨在为后续同学的学习提供帮助和参考。

开源精神是本项目的核心，借助社区智慧和实践探索，希望能为开源事业贡献微薄之力。文档中的参考资料由ChatGPT/Deepseek自动基于网上现有的资源整合生成，经过作者审阅编写，其中可能存在不准确或不完善之处，欢迎读者提出宝贵意见，改进这个项目。




## 二、实验概述


本课程项目旨在基于 [RISC-V 工具链](https://github.com/riscv-collab/riscv-gnu-toolchain) 设计并实现一个代理内核（Proxy Kernel，PKE）操作系统（相当于虚拟机），旨在通过一系列实验任务，帮助学生深入理解操作系统原理及其与硬件的协同工作。通过实现一个简化的代理内核，学生将聚焦于操作系统核心功能的实现，如内存管理、进程调度和中断处理，避免过多的硬件相关问题，从而集中精力掌握系统的核心概念和技术。

本实验采用的代理内核架构，区别于传统的宏内核和微内核，重点在于通过简化的设计满足给定应用的需求，系统规模随着应用需求的变化而变化。这种方法不仅极大降低了实验的复杂性，也保留了操作系统设计的完整性，既能管理处理器和内存，又能通过与主机的通信来完成硬件功能，从而为后续的软硬协同设计打下基础。

在完成操作系统部分的实验后，系统能力培养部分将引导学生在 FPGA 开发板上部署 RISC-V 软核，并扩展代理内核以实现设备管理和文件访问等更复杂功能。整个实验流程循序渐进，从基础实验到挑战实验，旨在帮助学生深入理解操作系统和计算机系统的整体架构，并为开发和验证现代计算机系统提供实践经验。

[原实验概述](doc/实验概述.md)




## 三、环境搭建（Docker+vscode）
### 调试好上手即用的x64 ubuntu 24.04 LTS镜像
```
  docker pull crpi-x7y7w4q8rsfacqq9.cn-shanghai.personal.cr.aliyuncs.com/cubelander-images/x86-pke:latest
```
[配置过程](lab/环境配置.md)

### 调试工具的安装和使用

[调试工具安装配置过程](lab/调试工具.md)


## 四、实验内容

### lab1

[lab1_1 系统调用](lab/lab1_1.md)

[lab1_2 硬件中断](lab/lab1_2.md)

[lab1_3 外部中断](lab/lab1_3.md)

[lab1_challenge1 打印用户程序调用栈](lab/lab1_challenge1.md)


[lab1_challenge2 打印异常代码行](lab/lab1_challenge2.md)

[lab1_challenge3 多核启动和运行](lab/lab1_challenge3.md)

### lab2

[lab2 内存管理基础知识](lab/lab2.md)

[lab2_1 虚实地址转换](lab/lab2_1.md)

[lab2_2 简单内存分配](lab/lab2_2.md)

[lab2_3 缺页异常](lab/lab2_2.md)

[lab2_challenge1 复杂缺页异常](lab/lab2_challenge1.md)

[lab2_challenge2 堆空间管理](lab/lab2_challenge2.md)

[lab2_challenge3 多核内存管理](lab/lab2_challenge3.md)


### lab3
[lab3 进程调度基础知识](lab/lab3.md)

[lab3_1 fork](lab/lab3_1.md)

[lab3_2 yield](lab/lab3_2.md)

[lab3_3 循环轮转调度](lab/lab3_3.md)

[lab3_challenge1 进程等待和数据段复制](lab/lab3_challenge1.md)

[lab3_challenge2 实现信号量](lab/lab3_challenge2.md)

[lab3_challenge3 写时复制](lab/lab3_challenge3.md)

### lab4

[lab4 文件系统基础知识](lab/lab4.md)

[lab4_1 相对路径](lab/lab4_1.md)

[lab4_2 目录文件](lab/lab4_2.md)

## 五、附录

[riscv-software-src/riscv-isa-sim](https://github.com/riscv-software-src/riscv-isa-sim)

[riscv-gnu-toolchain 百度网盘zip](https://pan.baidu.com/s/1Z9xKV_UY2Li_SxYrbJT5Zw?pwd=cpbf)