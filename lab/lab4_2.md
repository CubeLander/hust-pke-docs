# lab4_2 目录

## 实验目标
完善对目录文件的操作

## 新特性
增加了对目录的打开和关闭，以及创建和读取用户接口：
```c
// added @lab4_2
#define SYS_user_opendir  (SYS_user_base + 24)	
// -> sys_user_opendir()
//		系统调用接口，从目录名得到文件描述符file descriptor
// -> do_opendir 
//		调用vfs_opendir()，拿到文件句柄struct file*
//		将文件句柄struct file*装入进程控制块process中
//		封装成文件句柄file descriptor给用户
// -> vfs_opendir 
//		调用lookup_final_dentry()，拿到dentry
// -> viop_hook_opendir
// -> rfs_hook_opendir
#define SYS_user_readdir  (SYS_user_base + 25)
// sys_user_readdir -> do_readdir -> vfs_readdir -> viop_readdir -> rfs_readdir
#define SYS_user_mkdir    (SYS_user_base + 26)
// sys_user_mkdir -> do_mkdir -> vfs_mkdir -> viop_mkdir -> rfs_mkdir
#define SYS_user_closedir (SYS_user_base + 27)
// sys_user_closedir -> do_closedir -> vfs_closedir -> viop_hook_closedir -> rfs_hook_closedir
```

相应的，在ramfs中也增加了对应的方法：
```c
/**** vinode inteface ****/
const struct vinode_ops rfs_i_ops = {
    .viop_read = rfs_read,
    .viop_write = rfs_write,
    .viop_create = rfs_create,
    .viop_lseek = rfs_lseek,
    .viop_disk_stat = rfs_disk_stat,
    .viop_lookup = rfs_lookup,

    .viop_readdir = rfs_readdir,	// 新增
    .viop_mkdir = rfs_mkdir,		// 新增

    .viop_write_back_vinode = rfs_write_back_vinode,

    .viop_hook_opendir = rfs_hook_opendir, 		// 新增
    .viop_hook_closedir = rfs_hook_closedir,	// 新增
};

```