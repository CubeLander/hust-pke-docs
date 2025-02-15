# 华中科技大学 操作系统实验文档
## 一、前言

本文档旨在继承和发扬开源精神，借助社区智慧，通过一系列实践探索为开源事业贡献微薄之力，希望能够为后续的研究与开发提供参考与借鉴。

需要特别说明的是，文档中部分内容由 ChatGPT 自动生成，尚未经过充分推敲和严格验证，可能存在不准确或不完善之处。对此，敬请读者谅解，并欢迎大家提出宝贵意见，共同改进和完善。

文档中使用vscode的搜索和c语言扩展功能，从而实现源代码中对关键字的准确查找。故在引用函数和汇编符号时均省略目录和文件位置，希望读者自行确认。

## 二、实验概述

- [实验概述](lab/实验概述.md)


## 三、参考资料

- [riscv-software-src/riscv-pk: RISC-V Proxy Kernel](https://github.com/riscv-software-src/riscv-pk.git)
- [MrShawCode/riscv-pke: RISC-V Proxy Kernel for Education](https://github.com/MrShawCode/riscv-pke)
- [华中科技大学操作系统实验（riscv-pke）文档 - Gitee.com](https://gitee.com/hustos/pke-doc/tree/master)

- **riscv-gnu-toolchain百度网盘源**：[riscv-gnu-toolchain](https://pan.baidu.com/s/1Z9xKV_UY2Li_SxYrbJT5Zw?pwd=cpbf)

## 四、环境搭建（Docker+vscode）
- **调试好上手即用的x64 ubuntu 24.04 LTS镜像**：
```
  docker pull crpi-x7y7w4q8rsfacqq9.cn-shanghai.personal.cr.aliyuncs.com/cubelander-images/x86-pke:latest
```
- [环境配置](lab/环境配置.md)

## 五、实验过程

- [lab1_1 系统调用](lab/lab1_1.md)
- [lab1_2 硬件中断](lab/lab1_2.md)
- [lab1_3 外部中断](lab/lab1_3.md)
