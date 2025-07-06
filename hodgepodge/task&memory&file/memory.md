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

```c
page_address_init //创建了128个static struct page_address_slot
```



