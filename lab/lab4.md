# lab4 实验4：文件系统

## 文件系统概述

[【精华】文件系统：一个自顶而下的理解方式（原创）](../doc/文件系统.md)

[File System Implementation in RISC-V Proxy Kernel overview(Claude)](../doc/File_System_Overview.md)

[VFS Layer Communication in PKE(Claude)](../doc/VFS_Layer_Communication_in_PKE.md)

[文件系统如何工作(Deepseek)](../doc/文件系统的工作方式.md)

[文件系统使用dentry和vinode的设计范式(Deepseek)](../doc/文件系统设计范式.md)

[vinode与rfs_inode协作的设计模式(Deepseek)](../doc/vinode与rfs_inode的设计模式.md)

## 对源代码的优化
rfs_r1block和rfs_w1block形式上都是rfs私有方法，所以说可以在rfs.c中做静态声明。同时添加了静态函数`static struct rfs_dinode*  rfs_alloc_dinode(struct rfs_device * rdev, int* inum);`



## 新建的对象和数据结构

### hash_table 哈希表对象

在util中，新增了哈希表的声明和方法：hash_table.c, hash_table.h
事实上这个“哈希表”的名字并不准确，它的实现其实是一个用key作为关键字的唯一集合，内部数据结构是链表。

这个hash_table的特点是，体现c++的原始多态特性，能够自定义保存对象和比较函数。

### vinode VFS中的文件项

- 以`rfs`文件系统为例，`vinode`是基于`rfs_dinode`创建的。
- `dinode`是文件元信息在存储设备中的持久存储。
- `vinode`是`dinode`文件元信息的缓存，也是系统-->`vfs`-->`rfs`实际操作文件的接口。
- 从`dinode`写到`vinode`的过程，只在文件创建和打开时进行。
- 从`vinode`写回`dinode`的过程`rfs_write_dinode`，只在文件创建和写回时进行。

```c
// abstract vfs inode
struct vinode {
  int inum;                  // inode number of the disk inode
  // 只被存在disk inode的文件系统使用，记录文件在fs中的位置
  // 对于不存在disk inode的文件系统，会另外想办法决定vinode的实际文件位置
  int ref;                   // reference count
  int size;                  // size of the file (in bytes)
  int type;                  // one of FILE_I, DIR_I
  int nlinks;                // number of hard links to this file
  int blocks;                // number of blocks
  int addrs[DIRECT_BLKNUM];  // direct blocks
  void *i_fs_info;           
  // filesystem-specific info (see s_fs_info)
  // 在这里存放hostfs的文件描述符
  struct super_block *sb;          
  // super block of the vfs inode， 确定了vinode对应的具体文件系统
  const struct vinode_ops *i_ops;  
  // 操作这个vinode对应的方法，由具体文件系统提供
};

```

### dentry VFS中的目录项
- 一个dentry对应一个vinode
- 由于硬链接的存在，一个vinode可能同时被多个dentry引用
- 对于普通文件和目录文件，都用dentry管理
- 同时被目录树（通过parent字段）和哈希表dentry_hash_table存储

```c
// vfs abstract dentry
struct dentry {
  char name[MAX_DENTRY_NAME_LEN];	// 文件或目录名字
  int d_ref;
  struct vinode *dentry_inode;	// 文件的实际物理位置
  struct dentry *parent;
  struct super_block *sb;		// 记录文件系统
};
```

### proc_file_management 维护进程打开的所有文件
- 属于进程控制块的成员变量。
- 记录进程的pwd，和所有打开的文件
```c
// data structure that manages all openned files in a PCB
typedef struct proc_file_management_t {
  struct dentry *cwd;  // vfs dentry of current working directory
  struct file opened_files[MAX_FILES];  // opened files array
  int nfiles;  // the number of files opened by a process
} proc_file_management;
```

### struct file 打开的文件的控制块
```c
// data structure of an openned file
struct file {
  int status;
  int readable;
  int writable;
  int offset;
  struct dentry *f_dentry;
};

```

### rfs_device 实际存储设备



## vfs文件系统

### VFS概述
- PKE需要同时支持两类文件系统：
  - 通过spike模拟器，用hostfs和主机上的文件系统交互
  - 用内存当虚拟磁盘,ramfs文件系统
- 为了同时支持这两种文件系统，所以说在PKE中，引入虚拟文件系统(Virtual File System)

### VFS的初始化
s_start() -> fs_init() -> vfs_init()
- 初始化了全局变量dentry_hash_table和vinode_hash_table

### vfs提供的接口
为用户提供系统调用接口：
```c
// added @lab4_1
#define SYS_user_open (SYS_user_base + 17)
// sys_user_open	系统调用接口
//     |
//     V
// 	do_open		从vfs_open获取文件描述符，并存入进程控制块中
//     |
//     V
// 	vfs_open 		调用下层文件系统，拿到文件操作句柄。
//     |  
//     V
//	alloc_vfs_dentry
//	viop_create 		通过父目录inode提供的方法，创建新的inode
//	viop_hook_open		 实际文件系统打开文件操作

#define SYS_user_read (SYS_user_base + 18)
// sys_user_read	系统调用接口
//     |
//     V
// 	do_read		从进程控制块中获取文件句柄，然后做文件读
//     |
//     V
// 	vfs_read 调用下层文件系统，做实际读操作
//     | 
//     V
//	viop_read 实际文件系统的读操作

#define SYS_user_write (SYS_user_base + 19)
#define SYS_user_lseek (SYS_user_base + 20)
#define SYS_user_stat (SYS_user_base + 21)
#define SYS_user_disk_stat (SYS_user_base + 22)
#define SYS_user_close (SYS_user_base + 23)
```
[vfs_open](../code/file_system/vfs_open.md)









## hostfs文件系统对象

### hostfs文件系统的初始化
fs_init()->register_hostfs()
初始化hostfs的元对象，注册到全局文件系统表fs_list中
初始化hostfs的superblock:`fs_type->get_superblock = hostfs_get_superblock;`
挂载hostfs文件系统作为根目录：`vfs_mount("HOSTDEV", MOUNT_AS_ROOT);`

### hostfs文件系统接口
```c
/**** host-fs vinode interface ****/
const struct vinode_ops hostfs_i_ops = {
    .viop_read = hostfs_read,
    .viop_write = hostfs_write,
    .viop_create = hostfs_create,
    .viop_lseek = hostfs_lseek,
    .viop_lookup = hostfs_lookup,

    .viop_hook_open = hostfs_hook_open,
    .viop_hook_close = hostfs_hook_close,

    .viop_write_back_vinode = hostfs_write_back_vinode,
};
```




## rfs文件系统对象

### rfs的初始化
- 注册设备：register_rfs
- 初始化设备和根节点：[init_rfs_device](../code/file_system/init_rfs_device.md)
- 格式化ramfs文件系统： [rfs_format_dev](../code/file_system/rfs_format_dev.md)
  

- 挂载文件系统: [vfs_mount](../code/file_system/vfs_mount.md)

### rfs文件系统接口
```c
/**** vinode inteface ****/
const struct vinode_ops rfs_i_ops = {
    .viop_read = rfs_read,
    .viop_write = rfs_write,
    .viop_create = rfs_create,
    .viop_lseek = rfs_lseek,
    .viop_disk_stat = rfs_disk_stat,
    .viop_lookup = rfs_lookup,

    .viop_write_back_vinode = rfs_write_back_vinode,
};
```
[rfs_read](../code/file_system/rfs_read.md)

[rfs_write](../code/file_system/rfs_write.md)

[rfs_create](../code/file_system/rfs_create.md)

[rfs_lseek](../code/file_system/rfs_lseek.md)

[rfs_disk_stat](../code/file_system/rfs_disk_stat.md)

[rfs_lookup](../code/file_system/rfs_lookup.md)

[rfs_write_back_vinode](../code/file_system/rfs_write_back_vinode.md)


