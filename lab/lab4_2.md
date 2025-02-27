# lab4_2 目录

## 实验目标
完善对目录文件的操作

## 新特性

### 新增数据结构struct dir
这个struct dir是遍历目录过程中的目录项，用来从系统传回遍历结果。
dir_fd对应struct file中的offset由系统维护，在整个遍历过程中自增，用来决定在struct dir中传回的目录项index。在遍历完目录以后offset不会重置，而是直接释放这个目录句柄。
等待下一次遍历的时候，再临时创建目录文件。

### 新增用户接口
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
//		然后把dentry封装成file，传给上层。
// -> viop_hook_opendir -> rfs_hook_opendir
//		预加载目录数据到内存
#define SYS_user_readdir  (SYS_user_base + 25)
// -> sys_user_readdir()
//		系统调用接口，从目录名得到文件描述符file descriptor
// -> do_readdir 
//		从fd获取struct file *pfile
//		调用vfs_readdir(*pfile,*dir)
// -> vfs_readdir 
//		解析pfile，拿到dentry，并解析出inode
//		调用文件系统接口，执行目录解析动作
// -> viop_readdir -> rfs_readdir
//		将目录内容解析到dir数据结构中
#define SYS_user_mkdir    (SYS_user_base + 26)
// -> sys_user_mkdir()
//		系统调用接口
// -> do_mkdir 
//		调用vfs_mkdir(*pfile,*dir)
// -> vfs_mkdir 
//		首先解析目录字符串，得到父目录项struct dentry *parent
//		alloc_vfs_dentry创建新目录项dentry
//		通过父目录项的方法创建子目录项viop_mkdir(parent->dentry_inode, new_dentry);
// -> viop_mkdir -> rfs_mkdir
//		创建实际的rfs_dinode文件元信息
// 		调用 rfs_update_vinode 映射到 vfs 中的 vinode 
#define SYS_user_closedir (SYS_user_base + 27)
// sys_user_closedir -> do_closedir -> vfs_closedir -> viop_hook_closedir -> rfs_hook_closedir
```
[rfs_hook_opendir](../code/file_system/rfs_hook_opendir.md)

[vfs_mkdir](../code/file_system/vfs_mkdir.md)

[rfs_mkdir](../code/file_system/rfs_mkdir.md)

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
	// 提前把文件硬盘块中的目录项读进内存
    .viop_hook_closedir = rfs_hook_closedir,	// 新增
	
};

```

## 实验目标
补充rfs_readdir下的代码，从打开目录时，执行`rfs_hook_opendir`创建的`dir_cache`中，读出目录项信息，并通过`struct dir*`传回给用户




## 参考代码

### rfs_hook_opendir
```c
//
// when a directory is opened, the contents of the directory file are read
// into the memory for directory read operations
//
int rfs_hook_opendir(struct vinode *dir_vinode, struct dentry *dentry) {
  // allocate space and read the contents of the dir block into memory
  void *pdire = NULL;
  void *previous = NULL;
  struct rfs_device *rdev = rfs_device_list[dir_vinode->sb->s_dev->dev_id];

  // read-in the directory file, store all direntries in dir cache.
  for (int i = dir_vinode->blocks - 1; i >= 0; i--) {
    previous = pdire;
    pdire = alloc_page();

    if (previous != NULL && previous - pdire != RFS_BLKSIZE)
      panic("rfs_hook_opendir: memory discontinuity");

    rfs_r1block(rdev, dir_vinode->addrs[i]);
    memcpy(pdire, rdev->iobuffer, RFS_BLKSIZE);
  }

  // save the pointer to the directory block in the vinode
  struct rfs_dir_cache *dir_cache = (struct rfs_dir_cache *)alloc_page();
  dir_cache->block_count = dir_vinode->blocks;
  dir_cache->dir_base_addr = (struct rfs_direntry *)pdire;

  dir_vinode->i_fs_info = dir_cache;

  return 0;
}

```

### rfs_mkdir
```c

```

### rfs_add_direntry
```c
//
// add a new directory entry to a directory
//
int rfs_add_direntry(struct vinode *dir, const char *name, int inum) {
  if (dir->type != DIR_I) {
    sprint("rfs_add_direntry: not a directory!\n");
    return -1;
  }
	// 直接将子文件目录项的磁盘号附加到目录的数据块后面
	// 这个代码没有考虑到一个目录可以有多个目录块
  int block_index = dir->addrs[dir->size / RFS_BLKSIZE];
	uint64 offset = dir->size % RFS_BLKSIZE;

  struct rfs_device *rdev = rfs_device_list[dir->sb->s_dev->dev_id];

	// 读取实际磁盘块装入缓存，并写入内容
  if (rfs_r1block(rdev, block_index) != 0) {
    sprint("rfs_add_direntry: failed to read block %d!\n", block_index);
    return -1;
  }

  struct rfs_direntry *p_direntry = (struct rfs_direntry *)((uint64)rdev->iobuffer + offset);
  p_direntry->inum = inum;
  strcpy(p_direntry->name, name);

  // write the modified (parent) directory block back to disk
  if (rfs_w1block(rdev, block_index) != 0) {
    sprint("rfs_add_direntry: failed to write block %d!\n", block_index);
    return -1;
  }

  // update its parent dir state
  dir->size += sizeof(struct rfs_direntry);

  // write the parent dir inode back to disk
  if (rfs_write_back_vinode(dir) != 0) {
    sprint("rfs_add_direntry: failed to write back parent dir inode!\n");
    return -1;
  }

  return 0;
}

```