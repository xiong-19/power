### BRK

##### 一些比较重要的参考

> [linux-glibc内存管理小结2(内存相关系统调用的实现](https://www.cnblogs.com/Five100Miles/p/8481684.html)
>
> [ARM64内存管理十四：brk](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/09/ARM64内存管理十四-brk/)
>
> https://ost.51cto.com/posts/27290

##### set_brk

```c
// 完成bss段的映射; 初始化brk相关变量
static int set_brk(unsigned long start, unsigned long end, int prot)
{
	start = ELF_PAGEALIGN(start);
	end = ELF_PAGEALIGN(end);
	if (end > start) {
		/*
		 * Map the last of the bss segment.
		 * If the header is requesting these pages to be
		 * executable, honour that (ppc32 needs this).
		 */
		int error = vm_brk_flags(start, end - start,
				prot & PROT_EXEC ? VM_EXEC : 0);      < -- -- --||
		if (error)
			return error;
	}
	current->mm->start_brk = current->mm->brk = end;
	return 0;
}
```

```c
// 调用do_brk_flags; 若含有VM_LOCKED标志则调用mm_populate
int vm_brk_flags(unsigned long addr, unsigned long request, unsigned long flags){
    struct mm_struct *mm = current->mm;
    unsigned  long len = PAGE_ALIGN(request);
    LIST_HEAD(uf);
    ret = do_brk_flags(addr, len, flags, &uf)	< -- -- --||
    populate = ((mm->def_flags & VM_LOCKED) != 0);
	if (populate && !ret)
		mm_populate(addr, len);
}
```

```c
// this is really a simplified "do_mmap".  it only handles anonymous maps. 
static int do_brk_flags(unsigned long addr, unsigned long request, unsigned long flags, struct list_head *uf);{
    
    error = get_unmapped_area(file:NULL, addr, len, pgoff:0, flags:MAP_FIXED); // < -- -- --|| 注意此处的MAP_FIXED标志
    error = mlock_future_check(mm, mm->def_flags, len); // 如有VM_LOCKED标志，累加mm->locked_vm
    // 几个检查函数，查询释放与已有的重叠，是则取消映射。是否超越资源限制值，超过最大map数等。
    vma = vma_merge(mm, prev, addr, addr + len, flags,
			anon_vma:NULL, file:NULL, pgoff, policy:NULL, NULL_VM_UFFD_CTX);  // 判断能否与前继和后续合并（判断虚拟地址是否相接，想差很小也不行，调用__vma_adjust合并）
    // http://edsionte.com/techblog/archives/3586 [很详细]
    //		|--> vma_adjust_trans_huge(orig_vma, start, end, adjust_next);
    // 若是不能与已有合并，则创建一个新的
    vma = vm_area_alloc(mm);
    vma_set_anonymous(vma);
    vma_link(mm, vma, prev, rb_link, rb_parent); // 插入到红黑树中
}
```

```C
// Linux内核深度解析mm->get_unmapped_area被设置为了arch_get_unmapped_area(arch_pick_mmap_layout中被设置)
unsigned long get_unmapped_area(struct file *file, unsigned long addr, unsigned long len, unsigned long pgoff, unsigned long flags){
    if (flags & MAP_FIXED)
		return addr;
    find_start_end(addr, flags, &begin, &end); // begin = get_mmap_base(); end=DEFAULT_MAP_WINDOW;
    addr = PAGE_ALIGN(addr);
	vma = find_vma(mm, addr);  // tmp_end > addr || tmp->start < addr (只满足第一点的可)
    if(vma)return addr;
    // 若是没找一个vma或者找到的vma起始地址大于addr+len，直接返回 addr（前提addr存在，其为0则使用第二种方法查找）
    //方法二
    vm_unmapped_area(&info);   // info.end, info.begin由上面得到， 调用unmapped_area_topdown  一个较复杂的方法
}
    
```

#### SYSCALL_DEFINE1(brk, unsigned long, brk)

```c
SYSCALL_DEFINE1(brk, unsigned long, brk){	
	min_brk = mm->start_brk; 
	if (brk < min_brk)  // 直接返回mm->brk
		goto out;  
		
	if (check_data_rlimit(rlimit(RLIMIT_DATA), brk, mm->start_brk,
			      mm->end_data, mm->start_data))
		goto out;  // brk-start_brk  + end_start - start_data < rlimit(RLIMIT——DATA)	
    newbrk = PAGE_ALIGN(brk);
	oldbrk = PAGE_ALIGN(mm->brk);
	if (oldbrk == newbrk)
		goto set_brk;  //   mm->brk = brk; 这说明brk的值并没有页对齐？？；返回brk
	if (brk <= mm->brk) {
		if (!do_munmap(mm, newbrk, oldbrk-newbrk, &uf))  // 小于则释放
			goto set_brk;             // 区间说明：[brk, mm->rak], 即需要将该段区域撤销，可能跨越多个vma区域
		goto out;
	}	
    next = find_vma(mm, oldbrk);  // 是否与已有的mmap映射冲突（在已有vma区域中）
	if (next && newbrk + PAGE_SIZE > vm_start_gap(next)) // newbrk + 4k > vm->start
		goto out;
	/* Ok, looks good - let it rip. */
	if (do_brk_flags(oldbrk, newbrk-oldbrk, 0, &uf) < 0) // [mm->brk, brk]之间申请
		goto out;
    set_brk: // 设置brk，若是有VM_LOCKED标记则立即分配内存
        mm->brk = brk;
        populate = newbrk > oldbrk && (mm->def_flags & VM_LOCKED) != 0;
        up_write(&mm->mmap_sem);
        userfaultfd_unmap_complete(mm, &uf);
        if (populate)
            mm_populate(oldbrk, newbrk - oldbrk);
        return brk;
}
```

```c
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len,
	      struct list_head *uf){
	if ((offset_in_page(start)) || start > TASK_SIZE || len > TASK_SIZE-start)
		return -EINVAL; // 第一个判断以页为单位会报错？？    
    len = PAGE_ALIGN(len);
    vma = find_vma(mm, start);  
    if（start > vma->start）error = __split_vma(mm, vma, start, 0);  //区间说明：[vma->start, start, vma->end], start处于一个vma的中间, 不关心end
    end = start + len;
    last = find_vma(mm, end);
    if(end>last->vm_start)error= __split_vma(mm, last, end, 1)  // 区间说明：[vma->start, end, vma->end] end处于一个vma中间，不关心start
    // 经过两次__split_vma后可能会分割出下图的多个vma
    // 从下面的new开始往上走，依次撤销locked_vm的记录
    if(mm->loaked_vm){
        ……
        while(tmp->start < end){
        	if (tmp->vm_flags & VM_LOCKED) {
				mm->locked_vm -= vma_pages(tmp);
				munlock_vma_pages_all(tmp);
			}
            tmp = tmp->next
        }
    }
    	/*
	 * Remove the vma's, and unmap the actual pages
	 */
    // vma, prev是下图中，最下面两个区域
	detach_vmas_to_be_unmapped(mm, vma, prev, end);  // 从vma开始到end之间的vma都从rb树中去除
	unmap_region(mm, vma, prev, start, end);// 1)收集待移除区域的tlb信息 2)更新内存映射的水位线，mm->hiwater 3) unmap_vmas（对每一将要撤销的VMA）：根据是否为huge page，选择不同的撤销函数。将地址范围的页面的映射关系解除，pud->pmd->pte。清除页表项、移除TLB条目、标记页面为脏页、访问页面等。（普通页面，设备私有页面，交换页面，迁移页面） 4) free_pgtable：释放指定地址空间的页表项。根据释放为大页，采用不同的函数完成处理。
    // 2）完成VMA与物理页之间的映射关系。如反向映射，页面标记，页面添加到合适队列，相关资源被释放。4)释放PTEs及其上级页表结构，如释放不需要的页表
	arch_unmap(mm, vma, start, end);  
	/* Fix up all other VM information */
	remove_vma_list(mm, vma);  // 通过next依次释放vma数据结构,此处应该应该有（第一个区域的next指向最后一个vma，但是没找到此段代码）
}
// maybe：new_below分割的是指定addr的上方或者下方。__split_vma将vma在addr位置分割，被分割是那一部分由new_below决定
```

```c
// 对于do_munmap来说，有两种可能的调用；
// 分割的vma，分割的地址，地址向上还是地址向下
int __split_vma(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long addr, int new_below){
    new = vm_area_dup(vma); // 分配一个新的vma，*new = *vma;
    if (new_below)
		new->vm_end = addr;
	else {
		new->vm_start = addr;
		new->vm_pgoff += ((addr - vma->vm_start) >> PAGE_SHIFT);
	}
    err = vma_dup_policy(vma, new); // 复制vma->policy
    err = anon_vma_clone(new, vma); //将vma的每一个AVC指向的AV与新建的AVC，new，建立关联
    if(new_below)err = vma_adjust(vma, addr, vma->vm_end, vma->vm_pgoff +
			((addr - new->vm_start) >> PAGE_SHIFT), new);
    else err = vma_adjust(vma, vma->vm_start, addr, vma->vm_pgoff, new);
}
// vma_adjust的内容很复杂：
// 简单来说vma->start = start; vma->end = end;
```

<img src="http://xwf-img.oss-cn-beijing.aliyuncs.com/typora-img/image-20240410144932126.png" alt="image-20240410144932126" style="zoom: 50%;" />




#### brk的核心函数

+ 需要搞清楚虚拟空间的分配到底是哪个部分分配模块完成的
+ 虚实空间映射的具体映射过程

##### 一些需要注意的点：

+ 匿名映射
  + 在更新vma的vm_start, vm_end, vm_pgoff成员前vma必须使anon_vma_interval_tree_pre_update_vma移除相关anon_vma 
  + 在更新完vma后需要使用anon_vma_interval_tree_post_update_vma重新添加至树中 



##### merge：

| ![img](http://xwf-img.oss-cn-beijing.aliyuncs.com/typora-img/v2-c036ae212e51da04b20ec13a095fd6d4_r.jpg) | ![img](http://xwf-img.oss-cn-beijing.aliyuncs.com/typora-img/v2-c3310e7989f5112eaf54aa33fa61fd54_1440w.webp) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |



##### brk使用到的重要FLAG

+ VMA的flags(`奔跑吧，linux内核 P199`)
+ mmap的flags(`奔跑吧，linux内核 P223`)
+ 建议建议虚拟空间访问方式(`Linux内核深度解析 p128`) 如MADV_HUGEPAGE

##### 重要数据结构

```c
struct mm_struct{
	unsigned long start_brk, brk;
	unsigned long def_flags;  // 通过设置def_flags，可以为新创建的内存区域指定默认的权限核属性。仅影响新创建的，并可通过其他机制进一步设置与调整。
    unsigned long locked_vm; // 记录内存中被锁定的虚存大小，以页为单位。
}
```

##### some 

+ 在vma_merge是这种合并方式会对大页产生影响吗（不能给其映射大页）？还是分配时会合理得分配在得地址相连？（没有对齐吗？）
+ do_brk_flags中的vma_adjust_trans_huge(orig_vma, start, end, adjust_next)函数到底干了什么。

#### 匿名映射简要说明

> [linux内存源码分析 - 内存回收(匿名页反向映射) - tolimit - 博客园 (cnblogs.com)](https://www.cnblogs.com/tolimit/p/5398552.html)

设定：

+ 几乎所有匿名线性区都有anon_vma(除了两个相邻且特征相同的匿名区会使用同一个anon_vma)
+ 对于匿名映射区，VMA->vm_pgoff是VMA开始地址对应的虚拟页框号。

结构：

+ anon_vma的`rb_root`:将不同进程的AVC添加进来；`root`：？？管理所有anon_vma还是特定的

+ anon_vma_chain的`same_vma`: 加入其所属的VMA的anon_vma_chain中。`rb`：加入其他进程或者本进程的vma的anon_vma的红黑树中。

  + page的`mapping`：依据低位标志位，分别指向`anon_vma`与`address_space`。`index`保存此匿名页在VMA中以页为单位的偏移量。

    > page->mapping = (void *)&anon_vma + 1; 由anon_vma 2字节对齐的原因，+1使其低位为1
    >
    > struct anon_vma * anon_vma = (struct anon_vma *)(page->mapping - 1);

流程分析：

+ 创建VMA：初始VMA的`anon_vma, anon_vma_chain`为空，`vm_pgoff=vm_start>>12`

+ 初次访问时，匿名区的设置`do_anonymous_page`: 1) 若能与前一个vma的anon_vma合并则使用前一个的anon_vma, 不能则创建一个。2）创建一个AVC，分别指向anon_vma，vma，挂入vma的链中。3）创建PTE。4）设置`page（此函数中获取的）`的`mapping`, `index=(addr-vma->start)>>12 + vma_>vm_pgoff` 5) PTE写入页表

<img src="http://xwf-img.oss-cn-beijing.aliyuncs.com/typora-img/image-20240411125002766.png" alt="image-20240411125002766" style="zoom: 50%;" />

<center><font color='red' size=2 >图same_vma处小有问题</font></center>

+ fork（`dup_mm`）: 遍历父进程的vma，为每一个父进程的匿名线性区建立子进程对应的vma，anon_vma，一个或者多个anon_vma_chain

  1）分配一个vma，*vma = *oldvma。初始化子进程vma的anon_vma_chain为空。2）`copy_page_range`： 将父进程的页表复制到子进程。若此vma可写，则父子进程的页表项都设置为可读。3）`anon_vma_fork`:

  ​		(1)父进程没有anon_vma直接退出，（2）遍历父进程的anon_vma_chain, 每一次遍历都创建一个avc（<font color='#ff0000'>什么情况下VMA会有多个AVC？fork是父进程只有一个AVC，但是子进程会有两个AVC。第二个AVC的作用，当子进程访问未建立映射的虚拟地址时，可以将新建立映射的page，通过anon_vma，anon_vma_chain反向到子进程的vma。而第一个AVC保存子进程建立前已创建映射的page，可以反向映射到使用此页的子进程VMA</font>。）

	> anon_vma = pavc->anon_vma;    // pavc：父进程avc
    > anon_vma_chain_link(dst, avc, anon_vma);  //avc为新建， dst为子进程	

​			（3）分配一个`anon_vma, anon_vma_alloc`

```
 anon_vma->root = pvma->anon_vma->root; //pvma父进程vma
 vma->anon_vma = anon_vma;
 anon_vma_chain_link(vma, avc, anon_vma);
```

​		<img src="http://xwf-img.oss-cn-beijing.aliyuncs.com/typora-img/image-20240411133936298.png" alt="image-20240411133936298" style="zoom: 35%;" />

+ 反向映射：(1)page可以通过mapping找到anon_vma (2)由rb_root可以找到父与子进程的AVC，以此找到vma（3）由vma可以找到mm 

  > 注意：未创建映射，代表anon_vma其实并未建立

<font color='#990000' size=4>注：父进程在子进程fork完成映射的页也可以访问到子进程的VMA。所以反向映射需要对遍历到的VMA属于的MM的页表检查，是否有此映射页。</font>

> method: 有page的 index = (addr - vma-<start)>>12 + vm_pgoff  (此页面在第多少个虚拟页)
>
> page代表的线性地址 = vma->start + (page->index - vma-> vm_pgoff) << 12

以此地址查询进程的页表的表项，在检查表项对应的物理页与此物理页是否相同，达到反向映射判断目的。`page_check_address`

<font color='red'>注：以上为堆栈与私有匿名映射，共享匿名映射采用基于文件页的反向映射</font>

文件页的反向映射：1）没有地址空间映射的文件页：只存在与文件的address_space中，进程通过read/write等系统调用读写这些文件页（`不需要反向映射,回收时有page->mapping找到address_space，从其中删除即可`）  2）有地址映射的文件页：文件页存在于文件的address_space， 同时被映射到了一个或者多个进程的虚拟空间。`因此需要反向映射，address_space的i_map红黑树，存放了所有映射此文件的进程的VMA。放需要修改页表时，由i_map访问即可（遍历红黑树时，需判断进程是否映射了此页）`

### MMAP

+ 文件映射：磁盘文件映射到进程的虚拟地址空间
+ 匿名映射：初始化为全0的内存空间
+ 私有映射：多进程数据共享，修改不反映到磁盘实际文件，为COW的机制
+ 共享映射：多进程共享数据，修改`反映到磁盘具体文件？？`

#### SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len, unsigned long, prot, unsigned long, flags, unsigned long, fd, unsigned long, pgoff)

```c
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len, unsigned long, prot, unsigned long, flags, unsigned long, fd, unsigned long, pgoff){
	return ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff);
}
```

```c
unsigned long ksys_mmap_pgoff(unsigned long addr, unsigned long len,
			      unsigned long prot, unsigned long flags,
			      unsigned long fd, unsigned long pgoff){
    if(!(flags&MAP_ANONYMOUS){ //不是匿名页
        audit_mmap_fd(fd, flags) //current->audit_context, 审计子系统信息的记录
        file = fget(fd);
        if (is_file_hugepages(file))  // 判断file->f_op是否为hugetlbfs_file_operations或者shm_file_operations_huge
			len = ALIGN(len, huge_page_size(hstate_file(file))); //看inode上的设置
    }
     else if(flags & MAP_HUGETLB){  // 是匿名页，且flags有巨页标记
        hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK); // 右移26位，然后去低6位
     	len = ALIGN(len, huge_page_size(hs));
        file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,  // <-- -- --||
				VM_NORESERVE,
				&user, HUGETLB_ANONHUGE_INODE,
				(flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK); //设置并返回一个hugetlbfs文件指针
    }
    retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);  // <-- -- --||
}
```

```c
struct file *hugetlb_file_setup(const char *name, size_t size,
				vm_flags_t acctflag, struct user_struct **user,
				int creat_flags, int page_size_log){
	hstate_idx = get_hstate_idx(page_size_log); // 页面大小对数值确定一个索引
    mnt = hugetlbfs_vfsmount[hstate_idx]; //获取对应的大页文件系统的挂载点
    inode = hugetlbfs_get_inode(mnt->mnt_sb, NULL, S_IFREG | S_IRWXUGO, 0); // 创建一个inode，(并关联到指定挂载点mnt上,没太看到)。1）inode->i_mapping->a_ops = &hugetlbfs_aops; 2)			inode->i_op = &hugetlbfs_inode_operations;inode->i_fop = &hugetlbfs_file_operations;
    hugetlb_reserve_pages(inode, from:0,
			to:size >> huge_page_shift(hstate_inode(inode)), vma:NULL,
			vm_flags:acctflag))  // 根据size与页面大小，预留所需的大页，但是vm_flags==VM_NORESERVE则直接返回不预留
    file = alloc_file_pseudo(inode, mnt, name, O_RDWR,
					&hugetlbfs_file_operations);
}
```

```c
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long pgoff){
    security__mmap_file()	// 1)其中调用的call_int_hook(mmap_file, ...)：在特定时间触发时，遍历调用所有对此事件感兴趣的回调函数，对事件的执行过程发出干预。mmap_file:事件标识符。不同的事件标识符对应不同的钩子函数集。具体操作：依次对每一个注册的钩子函数执行【属于security_hook_heads】，钩子函数可能完成检查修改。钩子函数返回非0则立即停止（拒绝此次操作），为0继续。
    ret = do_mmap_pgoff(file, addr, len, prot, flag, pgoff,
				    &populate, &uf);   // <-- -- --||
    if(populate)mm_populate(ret, populate);
}
```

```c
do_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot, unsigned long flags,
	unsigned long pgoff, unsigned long *populate,
	struct list_head *uf)
{
	return do_mmap(file, addr, len, prot, flags, vm_flags:0, pgoff, populate, uf);
}
```

```c
unsigned long do_mmap(struct file *file, unsigned long addr,
			unsigned long len, unsigned long prot,
			unsigned long flags, vm_flags_t vm_flags,
			unsigned long pgoff, unsigned long *populate,
			struct list_head *uf){
    if (!(flags & MAP_FIXED))
		addr = round_hint_to_min(addr);  //确保地址不小于一个定值(4096), 并页对齐
    len = PAGE_ALIGN(len);
    addr = get_unmapped_area(file, addr, len, pgoff, flags);   <-- -- --||
    if (flags & MAP_FIXED_NOREPLACE) {
		struct vm_area_struct *vma = find_vma(mm, addr);
		if (vma && vma->vm_start < addr + len)
			return -EEXIST;
	} // 返回地址是否已经映射
    if(file){
        if(!file_mmap(file, inode, pgoff, len))return; // 检查映射的位置是不是超过文件的最大映射大小。
        。。。。。// 检查文件权限，属性和请求信息，来决定是否允许进行映射，并设置相应的映射属性
    }
    else{
        swicth(flags & MAP_TYPE){ //一个标志集合，可能出现的标志
            case MAP_SHARED: pgoff = 0;
            case MAP_PRIVATE: pgoff = addr >>PAGE_SHIFT; // 为anon_vma而设置
        }
    }
    // 若是有MAP_NORESERVE, 则这次映射使用的内存不计入系统的统计，页不会为该映射区域预留物理内存
    if (flags & MAP_NORESERVE) {
        /* We honor MAP_NORESERVE if allowed to overcommit */
        if (sysctl_overcommit_memory != OVERCOMMIT_NEVER)
            vm_flags |= VM_NORESERVE;
        /* hugetlb applies strict overcommit unless MAP_NORESERVE */
        if (file && is_file_hugepages(file)) // 判断是否为巨页：判断文件的f_op是否为hugetlbfs_file_operations，或者shm_file_operations_huge
            vm_flags |= VM_NORESERVE;
    }
    addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);  // <-- -- --||
	if (!IS_ERR_VALUE(addr) &&
	    ((vm_flags & VM_LOCKED) ||
	     (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))
		*populate = len;
	return addr;
}
```

> MAP_FIXED_NOREPLACE: 当讲文件或其他对象的内容映射到指定的虚拟地址时，仅当改地址区域未被映射时才能进行映射。若指定区域已存在映射，则返回错误（与MAP_FIXED相识的是都是提供精确的虚拟地址作为映射目标，而不是让系统自动选择一个合适的地址）

```c
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
		unsigned long pgoff, unsigned long flags){
	get_area = current->mm->get_unmapped_area;  // 为arch_get_unmapped_area
    if (file) {
		if (file->f_op->get_unmapped_area)
			get_area = file->f_op->get_unmapped_area;  // 特定文件系统相关，如ext4文件系统是，thp_get_unmapped_area
	} else if (flags & MAP_SHARED) {
		/*
		 * mmap_region() will call shmem_zero_setup() to create a file,
		 * so use shmem's get_unmapped_area in case it can be huge.
		 * do_mmap_pgoff() will clear pgoff, so match alignment.
		 */
		pgoff = 0;
		get_area = shmem_get_unmapped_area;  // 匿名映射是通过tmpfs创建的匿名文件
	}
	addr = get_area(file, addr, len, pgoff, flags);
    // 对于不同的映射类型选择不同的映射方式，匿名私有，文件映射，匿名共享映射
    // 三种不同的get_area最终都会调用mm->get_unmapped_area,即arch_get_unmapped_area
    // 如ext4的thp_get_unmapped_area:为透明大页分配一块为映射的内存区域。1）首先检查当前文件是否支持直接访问应用程序（Direct Access，DAX）且启用CONFIG_FS_DAX_PMD，满足则尝试透明大页的映射。否者调用current->mm->get_unmapped_area。
    //  shmem_get_unmapped_area：1）先调用current->mm->get_unmapped_area得到一个地址 2）然后根据一些条件决定是否将其扩展为巨页，并最终返回地址。若是扩展为巨页则再调用一次current->mm->get_unmapped_area（指定的地址不变，但是长度为大页长度），并且得到的地址页发生了调整。（HPAGE_PMD_SIZE）
    //current->mm->get_unmapped_area即arch_get_unmapped_area的内容在上面。
    error = security_mmap_addr(addr); // 调用call_int_hook(mmap_addr, 0, addr);
}
```

> 注意：thp_get_unmapped_area调用_ _thp_get_unmapped_area时，指定了透明大页的大小，为PMD_SIZE。而 _____ _thp_get_unmapped_area在进行一些地址处理后(len+size:PMD_SIZE)仍然调用了current->mm->get_unmapped_area

```c
// 主要作用是负责实现用户空间对内存区域的映射操作
unsigned long mmap_region(struct file *file, unsigned long addr,
		unsigned long len, vm_flags_t vm_flags, unsigned long pgoff,
		struct list_head *uf){
	if(!may_expand_vm(mm, vm_flags, len>>PAGE_SHIFT){ //判断当前total_vm + len>>PAGE_SHIFT是否大于资源限制量。
        nr_pages = count_vma_pages_range(mm, addr, addr + len); // 在mm中搜索，与[addr, addr+len]此虚拟地址存在重合的VMA中，重合的页面数
		if (!may_expand_vm(mm, vm_flags,
					(len >> PAGE_SHIFT) - nr_pages))
			return -ENOMEM;
    }
    /* Clear old maps */
    // 查询红黑树中是可以插入[addr, addr+len]的位置， 若是某个已经存在的vma->end > addr && vma->start< end 返回-ENOMEM, [(addr), start, (addr), (addr+len), end, (addr+len)],
    //   正常情况下返回 0
	while (find_vma_links(mm, addr, addr + len, &prev, &rb_link, &rb_parent)) {
		if (do_munmap(mm, addr, len, uf))
			return -ENOMEM;
	}
    // accountable_mapping检查是否是huge_tlb的大页，是则返回0，否者检查是否为VM_write（其实不止）   
    if (accountable_mapping(file, vm_flags)) {
        charged = len >> PAGE_SHIFT;
        if (security_vm_enough_memory_mm(mm, charged))
           return -ENOMEM;
       vm_flags |= VM_ACCOUNT;
    }
    vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
                    NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX);  // 见前面
   	if (vma)goto out;
    vma = vm_area_alloc(mm);
    vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_flags = vm_flags;
	vma->vm_page_prot = vm_get_page_prot(vm_flags);  // index=vm_flags (VM_READ|VM_WRITE|VM_EXEC|VM_SHARED)， 使用index作为protection_map的下标，得到一个值，然后与，与架构相关函数得到的值相与，最后过滤（一些值，其实是不做处理的），得到页面保护属性
	vma->vm_pgoff = pgoff;
    if(file){
        // 若是vm_flags有VM_DENYWRITE标志则，则调用函数使文件拒绝写访问。
        // 若是共享，则调用函数，将使用file->f_mapping判断使文件映射可写。
    	vma->vm_file = get_file(file);
		error = call_mmap(file, vma); // file->f_op->mmap(file, vma); 对于ext4文件系统其为：ext4_file_mmap:1)标记文件被访问2)设置文件的vm_ops与vm_flags
        addr = vma->vm_start;
		vm_flags = vma->vm_flags;
    }
    else if(vm_flags&VM_SHARED)error = shmem_zero_setup(vma); 
    else vma_set_anonymous(vma);    // vma->vm_ops = NULL;
    vma_link(mm, vma, prev, rb_link, rb_parent);
    file = vma->vm_file;
}
```

```c
shmem_zero_setup:
	file = shmem_kernel_file_setup("dev/zero", size, vma->vm_flags);
	vma->vm_file = file;
	vma->vm_ops = &shmem_vm_ops;
	if (IS_ENABLED(CONFIG_TRANSPARENT_HUGE_PAGECACHE) &&((vma->vm_start + ~HPAGE_PMD_MASK) & HPAGE_PMD_MASK) <(vma->vm_end & HPAGE_PMD_MASK))khugepaged_enter(vma, vma->vm_flags); // 判断是否启动了透明大页，并检查当前区域是否可以使用透明大页，满足则将其加入透明大页的管理中。（以mm的结构体管理的）list_add_tail(&mm_slot->mm_node, &khugepaged_scan.mm_head);
```


```c
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len,
	      struct list_head *uf){
    
}
```

#### munmap

#### some qustion

+ <font color='#123456'>系统调用的宏展开到底是怎么展开的？？</font>——《奔跑吧linux内核 p211》

