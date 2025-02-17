# lab1_challenge3 多核启动和运行
由于这个实验需要管理多个进程的内存，所以需要在lab2中实现mmu以后再完成。
## 实验目标
之前的实验都在单核环境下进行。在本次实验中，你需要修改操作系统内核使其支持两核并发运行，并且在每个核上加载一个程序运行，等到两个程序都执行完毕后退出并关闭模拟器。

### 给定应用

#### user/app0.c

```c
#include "user_lib.h"
#include "util/types.h"

int main(void) {
  printu(">>> app0 is expected to be executed by hart0\n");
  exit(0);
}
```

#### user/app1.c

```c
#include "user_lib.h"
#include "util/types.h"

int main(void) {
  printu(">>> app1 is expected to be executed by hart1\n");
  exit(0);
}
```

在本次实验中，给定两个简单的用户程序，每个程序会输出一句话。你需要让每个核分别加载一个程序，并能够正确运行，输出相应内容然后退出。