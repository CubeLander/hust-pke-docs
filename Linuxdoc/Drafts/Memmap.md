这是个很好的问题！Linux内核在处理`mem_map`数组大小时采用了灵活的方法，因为系统的物理内存配置在启动时才能确定。

Linux通过以下方式确定和处理`mem_map`的大小：

1. **启动时内存探测**：
   在系统引导过程中，内核会探测可用的物理内存范围，通常通过BIOS/UEFI、设备树或其他平台特定机制获取内存信息。

2. **动态分配**：
   `mem_map`不是静态分配的固定大小数组，而是在内核初始化期间根据探测到的内存大小动态分配。每个物理页面对应一个`struct page`结构体。

3. **基于区域的分配**：
   现代Linux内核实际上按内存区域(memory zones)和节点(nodes)组织`mem_map`，而不是单一的全局数组：
   ```c
   // 对于UMA系统
   mem_map = alloc_bootmem_node(NODE_DATA(0), page_size * max_pages);
   
   // 对于NUMA系统
   NODE_DATA(nid)->node_mem_map = alloc_bootmem_node(...);
   ```

4. **稀疏内存支持**：
   对于有内存空洞或可热插拔内存的系统，Linux使用稀疏内存模型(SPARSEMEM)，将物理地址空间划分为固定大小的"sections"：
   ```c
   struct mem_section {
       struct page *section_mem_map;
       /* 其他字段 */
   };
   ```
   这里只为实际存在的内存部分分配`struct page`数组，而不是为整个物理地址空间分配。

5. **页面偏移计算**：
   系统使用`page_offset_base`和物理页号来计算从物理地址到`struct page`数组元素的映射：
   ```c
   #define __pa_to_page(pa) (pfn_to_page(__pa_to_pfn(pa)))
   #define pfn_to_page(pfn) (_pfn_to_page(pfn))
   ```

6. **内存热插拔**：
   当添加新内存(如在服务器上热插拔内存模块)时，内核会为新页面动态分配`struct page`结构并整合到页面管理系统中。

这种灵活的设计让Linux内核能够适应从嵌入式设备到大型服务器的各种内存配置，并支持动态变化的内存大小。通过稀疏内存模型和分段分配，内核避免了为潜在的巨大物理地址空间预分配`mem_map`，只为实际存在的物理内存分配所需的元数据空间。