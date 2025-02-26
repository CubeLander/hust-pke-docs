# lab3_2 进程yield

## 实现原理

在软中断的handle_mtimer_trap中，为每个进程记录tick_count++
在之后的rrsched()中，进行tick_count的比较和处理。