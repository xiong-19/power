![image-20250706212345512](../markdown_fig/image-20250706212345512.png)

![kernel启动流程-head.S的执行_2.总体流程_kernel head.s-CSDN博客](../markdown_fig/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBASFplcm8uY2hlbg==,size_20,color_FFFFFF,t_70,g_se,x_16.png)

> vmlinux.lds: 中可以查看各段的地址(如**_text,** **idmap_pg_dir**, **swaooer_pg_dir**)
>
> vmlinux.lds.S: 查看设置



#### 内核启动 :+1:

+ `__create_page_tables`

> 注意：取址为相对取址，所以一开始取的地址就是变量所在的物理地址

```
(1) 清空init_pg_dir到init_pg_end
(2) 将物理地址的__idmap_text_start到__idmap_text_end区域映射到对应的物理地址（以此可以发生函数跳转，相对跳转）在idmap_pg_dir中。
(3) 将_start与_end的物理地址映射到其对应的虚拟地址地址(KIMAGE_VADDR+_end-_start)
	va = pa+KIMAGE_VADDR
建立两个映射的原因开启MMU时，PC为一个很小的值，采用ttbr0_el1(idmap_pg_dir); ttbr1_el1(init_pg_dir)	
注意：此处为2MB的块映射。

开启mmu：
	bl	__enable_mmu    // 分别设置ttbr0_el1; ttbr1_el1, 设置sctlr_el1寄存器
	ldr	x8, =__primary_switched   // bl	start_kernel
	adrp	x0, __PHYS_OFFSET
	blr	x8
```

+ `start_kernel`

> 首先开启了虚拟映射，但是只有**DTB**的物理地址，而物理地址相关的内容都在**DTB**中，所以得先解析DTB，所以这时候**Fixed map**出现，映射DTB到**fixed map**的区域。而解析的**memory**记录在那儿？**memblock**出现，同时现在只有小范围的物理空间被映射了，那么为需要分配一个页面做一些一些事，memblock便起到了分配空间的作用。

```c
page_address_init; //创建了128个static struct page_address_slot

setup_arch:

		early_fixmap_init；  // 计算得到了FIXADDR_START在页表的pmd表项，并在表项填写了bm_pte的物理地址
// *** 初期只能访问小块区域内存kernel image ***
//  ===fixmap这部分内容主要是用于在MMU enabled ~~ mem_init完成之前阶段内存使用===
/* 	(1)FDT的映射，分配一块固定的FDT虚拟地址用于映射其物理地址使用；
	(2)IOMAP的映射，对于这个阶段外设的使用，这里是有一段地址，分配给到IO使用，与FDT的区别在于，FDT为永久映射，而IOMAP这里是用完擦除；
	(3)paging init的时候使用fixmap的PGD\PMD\PTE等信息，方便建立后续的映射；
*/
            
// FIXMAP: 在Linux内核启动的早期阶段，完整的页表机制尚未建立，动态内存分配器也未初始化。此时，内核需要通过Fixmap（固定映射）机制访问特定的物理内存或硬件寄存器。
// enum fixed_address中记录fixmap的地址区域细分; 包含永久区域用于某个内核模块；临时区域各模块皆可使用。
//	功能用于dtb解析；early console等等
            
        early_ioremap_init；//   将FIXADDR_TOP向下的几个区域的地址写入slot_virt数组中
            
        setup_machine_fdt: //将__fdt_pointer对应的物理地址与虚拟地址FIX_FDT建立映射 
				memblock_reserve: //memblock_add_range(&memblock.reserved, base, size, MAX_NUMNODES, 0);   base为设备树的物理地址，大小为其大小
				early_init_dt_scan； // early_init_dt_scan_memory:扫描dts的内存节点,调用memblock_add(base, size);添加内存节点； memblock_mark_hotplug标记热插拔
            
   		cpu_uninstall_idmap; // 清空恒等映射，ttbr0设置为reserved_pg_dir
            
        arm64_memblock_init(); 
// ===在启动阶段, 内存分配并不需要很复杂,也不必它多考虑回收再利用，所以使用 memblock 来进行内存分配管理。[直到free_initmem]====

//通过memblock_remove将某些memblock_region区域从memblock.memory中移除，这些区域包含了DDR物理地址所不包含的区域，以及内核线性映射区所不能涵盖的区域；同时将某些物理区间添加到memblock.reserved中，这额区间包含dts中预留区域，命令行中通过参数预留的CMA区域，内核的代码段、initrd、页表、数据段等所在区域，crash kernel保留区域以及elf相关区域。

		paging_init();  // 在early_fixmap_init中建立了fixmap的映射，在FIX_PGD的虚拟地址的slot_virt填写了swapper_pg_dir的物理地址: 相当于在FIX_PGD以可以访问到页表(?); 
			map_kernel; // 早期的映射PGD,PUD,PMD占用的内存是连续的,所以需要重新映射。
			// 通过early_pgtable_alloc遍历节点分配页面
```

> ref: https://www.cnblogs.com/zhangzhiwei122/p/16062195.html 【memblock】 memblock有两个memblock_type一个memory与一个reserved，memblock_type为一个结构体包含多个memblock_regios,而memblock_regios包含base, size, flag

| ![img](../markdown_fig/1771657-20190831230833457-192209033-17519028970922.png) | ![img](../markdown_fig/1771657-20190907233854385-1365207283.png) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |

