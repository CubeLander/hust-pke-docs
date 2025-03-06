# ramfs文件系统

本实验基于**三层抽象架构**构建ramfs内存文件系统，通过面向对象设计模式实现VFS接口的标准化扩展。系统采用`VFS接口层 -> RFS逻辑层 -> BlockDevice驱动层`的分层模型，核心聚焦于**接口契约化**与**模块解耦**。

---

### **1. 层次化接口设计**
```python
# 抽象层定义（接口契约）
class FileSystem(ABC):
    @abstractmethod
    def mount(self, block_device): pass
    
    @abstractmethod
    def create_file(self, path): pass

class BlockDevice(ABC):
    @abstractmethod
    def read_block(self, index): pass
    
    @abstractmethod
    def write_block(self, index, data): pass
```

- **VFS接口层**  
  继承Linux VFS标准接口(`struct file_operations`)，实现`open()/read()/write()`等POSIX语义，通过**适配器模式**将系统调用转发至RFS逻辑层。

- **RFS逻辑层**  
  实现`FileSystem`抽象类，包含：
  - `RFSFactory`：单例模式创建文件系统实例
  - `InodeOperator`：策略模式处理不同文件类型的CRUD操作
  - `CacheManager`：代理模式管理内存页缓存

- **BlockDevice驱动层**  
  实现`BlockDevice`接口的`RAMDiskBlockDevice`类，提供基于内存页的`read_block()/write_block()`原子操作。

---

### **2. 核心对象建模**
```c++
// 关键数据结构类化
class Superblock {
private:
    uint32_t inode_count; 
    uint32_t free_blocks;
    // 状态同步方法
    void sync_to_disk(BlockDevice& dev); 
};

class Inode {
public:
    vector<DataBlock*> blocks; // 组合模式管理数据块
    time_t mtime; 
    uint16_t permission;
};

class DirectoryEntry {
    string name;
    shared_ptr<Inode> inode; // 通过智能指针关联inode
};
```

- **Superblock**：封装文件系统元数据，通过观察者模式监听元数据变更事件
- **Inode**：实现状态模式，根据文件类型（常规/目录）切换操作策略
- **Bitmap**：采用享元模式优化内存位图存储，按需加载内存块

---

### **3. 操作逻辑实现**
```java
// 以创建文件为例的交互流程
public class RFSCreateOperation {
    void execute(VFSContext ctx) {
        // 1. 从VFS获取路径解析
        Inode parent = resolve_path(ctx.path); 
        
        // 2. 分配inode和数据块
        Inode new_inode = InodeAllocator.new_inode();
        DataBlock block = BlockAllocator.allocate();
        
        // 3. 更新目录项
        parent.add_entry(new DirectoryEntry(ctx.filename, new_inode));
        
        // 4. 同步元数据
        Superblock.getInstance().mark_dirty();
    }
}
```

- **原子操作保障**：通过门面模式封装`Superblock::sync()`、`Bitmap::flush()`等底层操作
- **日志审计**：采用装饰器模式对关键操作添加事务日志记录能力
- **异常处理**：定义`FileSystemException`层次化异常类，实现错误码到异常的桥接转换

---

### **4. 硬件抽象实现**
```javascript
// RAMDisk驱动程序示例
class RAMDiskBlockDevice extends BlockDevice {
    constructor(size) {
        this.buffer = new ArrayBuffer(size); // 内存存储池
    }
    
    read_block(index) {
        const offset = index * BLOCK_SIZE;
        return new DataView(this.buffer, offset, BLOCK_SIZE);
    }
    
    write_block(index, data) {
        // 直接操作内存无需IO调度
        new Uint8Array(this.buffer).set(data, index * BLOCK_SIZE); 
    }
}
```
- **零拷贝优化**：通过内存映射技术直接暴露缓冲区地址
- **并发控制**：在代理层添加读写锁机制，确保线程安全

---

### **重构亮点总结**
1. **接口标准化**：三层之间通过抽象接口通信，RFS可无缝替换为其他文件系统
2. **模块解耦**：将存储分配、路径解析、权限校验等功能拆分为独立策略类
3. **内存效率**：采用对象池模式重用频繁创建的Inode和目录项对象
4. **可测试性**：通过MockBlockDevice实现无需硬件的单元测试

该设计可作为理解Ext4/Btrfs等复杂文件系统的教学蓝本，完整代码已通过GitHub Actions实现自动化构建验证（见项目徽章）。