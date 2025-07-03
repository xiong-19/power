> [1] [linux内核源码分析之proc文件系统（一）_proc文件系统 内核处理-CSDN博客](https://blog.csdn.net/WANGYONGZIXUE/article/details/123744116)
>
> [2] [linux内核源码分析之proc文件系统（二）_linux proc函数用法-CSDN博客](https://blog.csdn.net/WANGYONGZIXUE/article/details/123746390)
>
> [3] [linux内核源码分析之proc文件系统（三）-CSDN博客](https://blog.csdn.net/WANGYONGZIXUE/article/details/123747091)
>
> [4] https://blog.csdn.net/a29562268/article/details/127041409



```c
// 相关数据结构
struct pid_entry {
	const char *name;
	unsigned int len;
	umode_t mode;
	const struct inode_operations *iop;
	const struct file_operations *fop;
	union proc_op op;
};
ONE("syscall",   S_IRUSR, proc_pid_syscall),

```



```c
start_kernel(void) --> proc_root_init()    
    
void __init proc_root_init(void)
{
	proc_init_kmemcache();  // 创建proc_inode_cachep, pde_opener_cache, proc_dir_entry_cache
	set_proc_pid_nlink();  // 计算pid_entry硬链接表的数量
	proc_self_init();
	proc_thread_self_init();
	proc_symlink("mounts", NULL, "self/mounts"); // 创建/proc/self/mounts, 连接到/proc/mounts

	proc_net_init();  // 创建/proc/self/net，链接到/proc/net，父节点为NULL
	proc_mkdir("fs", NULL);  // 创建/proc/fs，父节点为NULL
	proc_mkdir("driver", NULL);
	proc_create_mount_point("fs/nfsd"); // 创建/proc/fs/nfsd/，父节点为NULL，没有指定inode操作
    
#if defined(CONFIG_SUN_OPENPROMFS) || defined(CONFIG_SUN_OPENPROMFS_MODULE)
	/* just give it a mountpoint */
	proc_create_mount_point("openprom");
#endif
	proc_tty_init();
	proc_mkdir("bus", NULL);
	proc_sys_init();

	register_filesystem(&proc_fs_type);  // // 注册文件系统，它保存在file_systems的链表
}

```

