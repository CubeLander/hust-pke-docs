# File System Implementation in RISC-V Proxy Kernel for Education (PKE)

The RISC-V PKE implements a practical file system architecture that provides essential file operations while maintaining the "just-enough" philosophy of a proxy kernel. This implementation includes several key components:

## File System Architecture

The PKE file system is structured with a layered design:

1. **Virtual File System (VFS)** - An abstraction layer that provides a uniform interface for different file system implementations
2. **File System Implementations**:
   - **Host File System (HOSTFS)** - Allows access to files on the host system
   - **RAM Disk File System (RFS)** - An in-memory file system for temporary storage

## Key Components

### VFS Layer (`kernel/vfs.h/c`)

The VFS layer implements:
- Abstract objects like `dentry` (directory entry), `vinode` (virtual inode), and `file`
- File operation interfaces: `open`, `read`, `write`, `lseek`, `close`
- Directory operations: `opendir`, `readdir`, `mkdir`, `closedir`
- Hard link operations: `link`, `unlink`

### RFS Implementation (`kernel/rfs.h/c`)

RFS is a simple file system that:
- Uses a portion of RAM as disk space
- Follows a structure similar to traditional UNIX file systems with superblocks, inodes, and data blocks
- Has a layout with: superblock (1 block), disk inodes (10 blocks), bitmap (1 block), and data blocks (100 blocks)

### HOSTFS Implementation (`kernel/hostfs.h/c`)

HOSTFS provides:
- Access to the host file system through the Spike emulator interface
- Translation between PKE file operations and host system calls

### Device Management (`kernel/ramdev.h/c`)

The device layer handles:
- RAM disk device initialization and management
- Block-level read/write operations

### User Interface (`kernel/proc_file.h/c`)

This component:
- Connects system calls to the VFS layer
- Manages per-process file descriptors and working directories

## File System Operations

The PKE file system supports:
- Basic operations: open, read, write, seek, close
- Directory management: opendir, readdir, mkdir, closedir
- Hard linking capabilities: link, unlink
- File information retrieval: stat, disk_stat

## Hard Link Implementation

The hard link functionality demonstrates the UNIX-like design:
- Links share the same inode
- Each link increments the link count in the inode
- When a link is removed, the link count decreases
- The file data is freed only when the link count reaches zero

This design provides a practical educational example of how file systems work in real operating systems while maintaining the simplicity needed for educational purposes.