# lab4_challenge1 相对路径

## 实验目标
实现形如：`./file`、`./dir/file`以及`../dir/file`的路径形式

## 实验内容

在user_lib.c中，增加了`SYS_user_rcwd`和`SYS_user_ccwd`的系统调用，我们需要在内核中实现这两个系统调用。

对cwd的维护，事实上可以在process中进行，然后进程向下传递系统调用时，都传递拼接过的绝对路径。这样我们就不需要修改vfs的代码解析方法了。

理论上来说需要将字符串传回给用户，但是由于proxy kernel的极简设定，就直接在内核中打印了。

cwd的dentry和初始化已经在proc_file_manager中实现了。

我们不为`../`和`./`做形式上的dentry目录项，因为可以在路径解析的时候特殊处理，并且为它们做目录项缓存很占地方。
