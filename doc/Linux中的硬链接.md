在 Linux 的 VFS（Virtual File System）中，**硬链接的创建通过增加指向同一 inode 的目录项实现**，其核心流程涉及 VFS 抽象层与具体文件系统的协作。以下是详细实现机制：

---

### 一、硬链接的本质
1. **共享 inode**  
   硬链接是文件系统中多个目录项（dentry）指向**同一个 inode**，通过 `inode->i_nlink` 统计引用数。
2. **限制条件**  
   - **不能跨文件系统**：inode 编号仅在同一个文件系统内有效。
   - **目录硬链接受限**：避免环路（仅超级用户可通过 `mknod` 创建）。

---

### 二、VFS 层实现流程
#### 1. 系统调用入口  
用户态通过 `link()` 或 `linkat()` 触发，内核进入 `do_linkat()` 函数：
```c
SYSCALL_DEFINE5(linkat, ...) {
    return do_linkat(olddfd, oldname, newdfd, newname, flags);
}
```

#### 2. VFS 通用检查  
- **路径解析**：通过 `user_path_at()` 解析源文件路径，获取 `struct path`。
- **合法性校验**：
  - 源文件存在且非目录（除非特权用户）。
  - 目标文件不存在。
  - 源与目标在同一文件系统（防止跨设备链接）。

#### 3. 调用具体文件系统操作  
通过 `vfs_link()` 调用具体文件系统（如 ext4）的 `link` 方法：
```c
int vfs_link(struct dentry *old_dentry, struct inode *dir, 
             struct dentry *new_dentry) {
    // 调用文件系统实现的 link 方法
    return dir->i_op->link(old_dentry, dir, new_dentry);
}
```

---

### 三、具体文件系统实现（以 ext4 为例）
#### 1. 添加目录项  
在目标目录中创建新目录项（dentry），指向源文件的 inode：
```c
const struct inode_operations ext4_dir_inode_operations = {
    .link = ext4_link,  // ext4 的 link 方法
};

static int ext4_link(struct dentry *old_dentry, struct inode *dir,
                     struct dentry *new_dentry) {
    struct inode *inode = d_inode(old_dentry);
    // 在目录中添加新条目
    err = ext4_add_entry(..., new_dentry, inode);
    if (!err) {
        inode_inc_link_count(inode);  // 增加链接计数
        d_instantiate(new_dentry, inode);
    }
    return err;
}
```

#### 2. 更新 inode 引用计数  
`inode_inc_link_count()` 增加 `i_nlink`，并标记 inode 为脏（需写回磁盘）：
```c
void inode_inc_link_count(struct inode *inode) {
    inode->i_nlink++;
    mark_inode_dirty(inode);
}
```

#### 3. 日志与磁盘同步  
对于日志型文件系统（如 ext4），通过事务确保原子性：
```c
handle_t *handle = ext4_journal_start(dir, EXT4_HT_DIR, credits);
// 在事务中执行添加目录项和更新 inode
ext4_journal_stop(handle);
```

---

### 四、关键数据结构与函数
| **结构/函数**          | **作用**                                 |
|-------------------------|----------------------------------------|
| `struct inode_operations` | 定义文件系统 inode 操作（含 `link` 方法） |
| `d_instantiate()`       | 将 dentry 与 inode 关联                |
| `mark_inode_dirty()`    | 标记 inode 为脏，触发写回              |

---

### 五、错误处理与边界条件
1. **EXDEV 错误**  
   若源与目标不在同一设备，返回 `-EXDEV`（通过 `old_path.mnt != new_path.mnt` 检测）。
2. **权限检查**  
   - 目标目录需有写权限（`inode_permission(dir, MAY_WRITE)`）。
   - 源文件不可被标记为不可链接（如某些特殊文件系统限制）。
3. **资源竞争**  
   使用 `dentry` 缓存锁（`d_lock`）防止并发修改导致状态不一致。

---

### 六、性能优化
1. **目录项缓存**  
   新创建的 dentry 加入缓存（`dentry_cache`），加速后续路径查找。
2. **日志批处理**  
   在 ext4 中，多个操作合并到单个事务，减少磁盘 I/O。

---

### 总结
VFS 通过抽象层将硬链接的通用逻辑（路径解析、权限校验）与具体文件系统的实现（目录项操作、inode 管理）解耦。**核心步骤**包括：
1. 系统调用触发 VFS 层检查。
2. 调用具体文件系统的 `link` 方法添加目录项。
3. 更新 inode 引用计数并同步至磁盘。  
硬链接通过共享 inode 实现多路径访问同一数据，是文件系统高效管理文件的重要机制。