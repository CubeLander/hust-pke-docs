# lab4_3 硬链接

## 实验目标



## 参考资料



## 实验原理
[Linux中的硬链接(Deepseek)](../doc/硬链接.md)




## 源代码的变化
增加了用户接口link和unlink
相应地，在rfs中的vinode接口中，增加了rfs_link和rfs_unlink
```c
/**** vinode inteface ****/
const struct vinode_ops rfs_i_ops = {
    .viop_read = rfs_read,
    .viop_write = rfs_write,
    .viop_create = rfs_create,
    .viop_lseek = rfs_lseek,
    .viop_disk_stat = rfs_disk_stat,
    .viop_link = rfs_link,
    .viop_unlink = rfs_unlink,
    .viop_lookup = rfs_lookup,

    .viop_readdir = rfs_readdir,
    .viop_mkdir = rfs_mkdir,

    .viop_write_back_vinode = rfs_write_back_vinode,

    .viop_hook_opendir = rfs_hook_opendir,
    .viop_hook_closedir = rfs_hook_closedir,
};
```

## link的调用关系
link(path1,path2)
- 虚实地址转换
- do_link(path1, path2)

do_link(path1, path2)
- vfs_link(oldpath, newpath)

vfs_link(oldpath, newpath)
- 先查询路径有效
- 给新路径分配dentry
- 设置dentry到old_vinode的对应关系
- 调用viop_link创建实际硬链接

rfs_link(parent_vinode, new_dentry, old_vinode)
- 所以说，在其中我们只需要登记目录项就可以了。
- 需要注意处理一下返回值的传递。


## 相关源代码参考

### 

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


### rfs_create
```c
//
// create a file with "sub_dentry->name" at directory "parent" in rfs.
// return the vfs inode of the file being created.
//
struct vinode *rfs_create(struct vinode *parent, struct dentry *sub_dentry) {
  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  int free_inum = 0;
	struct rfs_dinode *free_dinode = rfs_alloc_dinode(rdev,&free_inum);

  // initialize the states of the file being created

  /* ===== 阶段4：初始化磁盘inode ===== */
  // 设置新文件元数据（原TODO部分）
  free_dinode->size = 0;        // 初始文件大小为0字节
  free_dinode->type = R_FILE;    // 标记为普通文件类型（需确认RFS_FILE定义）
  free_dinode->nlinks = 1;      // 初始链接数（父目录引用）
  free_dinode->blocks = 1;      // 占用块数（即将分配第一个数据块）

  // DO NOT REMOVE ANY CODE BELOW.
  // allocate a free block for the file
  free_dinode->addrs[0] = rfs_alloc_block(parent->sb);

  // **  write the disk inode of file being created to disk
  rfs_write_dinode(rdev, free_dinode, free_inum);
  free_page(free_dinode);

  // ** build vfs inode according to dinode
  struct vinode *new_vinode = rfs_alloc_vinode(parent->sb);
  new_vinode->inum = free_inum;
  rfs_update_vinode(new_vinode);

  // ** append the new file as a direntry to its parent dir
  int result = rfs_add_direntry(parent, sub_dentry->name, free_inum);
  if (result == -1) {
    sprint("rfs_create: rfs_add_direntry failed");
    return NULL;
  }

  return new_vinode;
}
```