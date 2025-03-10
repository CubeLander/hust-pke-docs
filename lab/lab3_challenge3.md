# lab3_challenge3 写时拷贝

## 什么是写时拷贝技术
堆栈、和数据段，都是写时拷贝的。

## 写时拷贝技术的实现
当fork的时候，父子进程都丢失对页面的写权限
在写的时候，触发页错误，在其中做数据的复制和重新映射
系统调用返回的时候会重新做写动作，然后操作都一样。

## Q&A
### 系统如何区分页面错误中的写时拷贝和非法访问？

## 实现
主要我们需要修改fork()的代码，和操作系统处理页面错误的机制。

在内核页表中维护全局引用计数，使用sv39页表方案的保留字。
因为所有物理页都在内核页表中记录，所以说引用计数只需要考虑用户的引用。

对页错误服务程序做调整，能够识别写时拷贝和非法访问。

由于引用计数的实现，需要建一个全局的物理页描述表，做一个大工程，所以就省略了。
