# VFS Layer Communication in PKE

The Virtual File System (VFS) in the PKE kernel acts as an intermediary between user programs and concrete file system implementations (like HOSTFS and RFS). Its primary function is abstracting away the differences between file systems while efficiently passing parameters between layers.

## Parameter Communication

The VFS layer uses several key mechanisms to communicate parameters:

1. **Operation Function Pointers**: The `vinode_ops` structure contains function pointers that implement specific operations. When a user request arrives at the VFS layer, it invokes the appropriate function pointer from the concrete file system, passing standardized parameters.

2. **Abstract Data Structures**: The VFS defines universal structures (`vinode`, `dentry`, `file`) that all file systems implement. These structures contain both common fields and a filesystem-specific info pointer (`i_fs_info`/`s_fs_info`) that allows each file system to store its own private data.

3. **Unified Error Passing**: Return values follow consistent conventions across layers, with negative values indicating errors and specific positive values indicating success or sizes.

## Key Characteristics

1. **Clean Layer Separation**: The VFS maintains clean separation between layers through well-defined interfaces, preventing direct dependence on implementation details of specific file systems.

2. **Object-Oriented Design**: Though implemented in C, the VFS uses an object-oriented approach where operations are associated with objects:
   ```c
   viop_read(node, buf, len, offset) // Calls the appropriate read function for the node
   ```

3. **Path Resolution**: The VFS handles complex path resolution step-by-step, breaking down full paths into components and querying the appropriate file system for each directory entry.

4. **Caching Mechanisms**: The VFS implements hash tables for dentry and vinode caching, storing previously accessed entries to avoid repeated disk accesses.

5. **Reference Counting**: The VFS manages object lifetimes through reference counting, ensuring proper resource management as files are opened, closed, and shared.

This design allows the VFS to efficiently dispatch operations to the correct filesystem implementation while presenting a consistent interface to user programs, regardless of which underlying filesystem actually handles the request.