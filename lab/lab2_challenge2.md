# lab2_challenge2 堆空间管理

## 实验内容
修改内核的代码，使得应用程序的malloc能够在一个物理页中分配，并对各申请块进行合理的管理

学校的要求很笼统，只要求malloc能够分配一个物理页下的资源。

测试程序的要求如下：
```c
char *m = (char *)better_malloc(100);
char *p = (char *)better_malloc(50);
better_free((void *)m);
char *n = (char *)better_malloc(50);
// 这些代码实现之后，m和n的地址相同。
```
（待填坑）
## 具体实现

### 初始化用户程序堆空间

### malloc的实现

### free的实现
我们采用双向链表来管理堆内的空间：

