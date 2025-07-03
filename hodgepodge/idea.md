#### 1. 隔离进程

```c
// 唤醒进程
wake_up_process // 该步不能确定唤醒到哪个CPU上， 直接抄取部分代码。

正常状态 就绪队列 + TASK_RUNNING
TASK_INIERRUPTIBLE: 一般等待资源或者信号唤醒。而UNINTERRUPTIBLE不响应信号唤醒。
```

```c
sleeping_task = current;
set_current_state(TASK_INTERRUPTIBLE);
schedule();
func1();
/* Rest of the code ... */
/*
如果 schedule() 是被一个状态为TASK_RUNNING 的进程调度，那么 schedule() 将调度另外一个进程占用 CPU；如果 schedule() 是被一个状态为 TASK_INTERRUPTIBLE 或 TASK_UNINTERRUPTIBLE 的进程调度，那么还有一个附加的步骤将被执行：当前执行的进程在另外一个进程被调度之前会被从运行队列中移出，这将导致正在运行的那个进程进入睡眠，因为 它已经不在运行队列中了。

*/
wake_up_process(sleeping_task);
//在调用了 wake_up_process() 以后，这个睡眠进程的状态会被设置为 TASK_RUNNING，而且调度器会把它加入到运行队列中去。
```

```c
/* ‘q’是我们希望睡眠的等待队列 */
DECLARE_WAITQUEUE(wait,current);
add_wait_queue(q, &wait);
set_current_state(TASK_INTERRUPTIBLE);
/* 或 TASK_INTERRUPTIBLE */
while(!condition) /* ‘condition’ 是等待的条件 */
schedule();
set_current_state(TASK_RUNNING);
remove_wait_queue(q, &wait);
//上面的操作，使得进程通过下面的一系列步骤安全地将自己加入到一个等待队列中进行睡眠：首先调用 DECLARE_WAITQUEUE () 创建一个等待队列的项，然后调用 add_wait_queue() 把自己加入到等待队列中，并且将进程的状态设置为TASK_INTERRUPTIBLE 或者 TASK_INTERRUPTIBLE。然后循环检查条件是否为真：如果是的话就没有必要睡眠，如果条件不为真，就调用 schedule()。当进程 检查的条件满足后，进程又将自己设置为 TASK_RUNNING 并调用 remove_wait_queue() 将自己移出等待队列。
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/8a260e5856f342dd9316b756554db597.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpZWJhb19oYW4=,size_16,color_FFFFFF,t_70#pic_center)

```
// 直接迁移到目标CPU上运行
if (likely(cpu_active(dest_cpu))) {
    struct migration_arg arg = { p, dest_cpu };
    raw_spin_unlock_irqrestore(&p->pi_lock, flags);
    stop_one_cpu(task_cpu(p), migration_cpu_stop, &arg);
    return;
}
// 移动过去了，那么如何保持在该CPU上运行而不再转移到其他CPU上
```

```
__schedule 分析，当preempt=False时
	1) 若是state != running, [1] 有信号待处理则， state = running;[2]没有则将进程移出当前的CPU， 且 p->on_rq = 0
```

```c
移动进程选择：1)迁移 2) activate_task 
activate_task:
	enqueue_task:
		1. enqueue_entity(cfs_rq, se, flags);
			1. 更新cfs_rq
            2. 更新负载均衡
            3. se->on_rq = 1
            4. 插入红黑数中
		2. cfs->h_nr_running++ (对与每个调度实体都进行，与组调度相关)
        3. rq->nr_running++    
```

```c
移除，添加需要考虑的问题：
1) 红黑树节点问题
2) p->on_rq
3) se->on_rq
4) p->state  // p->on_rq = TASK_ON_RQ_QUEUED
```



#### 2. fork进程

```c
// SIGCHLD:进子程终止后发送SIGCHLD信号通知父进程。为子进城建立一个基于父进程的完整副本。
SYSCALL_DEFINE0(fork)
{
	return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}
// 父进程阻塞，直到子进程调用exit或者execve为止。VFORK：父进程被挂起，直到子进程释放资源。VM：执行相同的地址空间。
SYSCALL_DEFINE0(vfork)
{
	return _do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
			0, NULL, NULL, 0);
}
SYSCALL_DEFINE%(clone, ...){
	return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
	return _do_fork(flags|CLONE_VM|CLONE_UNTRACED, (unsigned long)fn,
		(unsigned long)arg, NULL, NULL, 0);
}


fork()函数的细节内容分析：
    1.	子进程的tsk->stack为重新分配的，只是内容与父进程相同。
    2.	sched_fork: 将state设置为TASK_NEW;prio设置为父进程的normal_prio。包含调度相关的数据结构的初始化。
    3.	copy_files: 依据CLONE_FILES（为0则重新创建）, 决定是否将子进程的p->files设置为新的数据结构。
    4.	copy_fs: CLONE_FS, p->fs; copy_sighand: CLONE_SIGHAND, p->sighand；copy_signal, CLONE_THREAD, p->signal; copy_mm, CLONE_VM, p->mm, p->active_mm; copy_io, CLONE_IO, task->io_context【CONFIG_BLOCK配置开启才有效】
    5.	copy_thread: 通过参数设置新线程的初始状态。区分内核线程还是用户进程在这里完成。新进程的PC值被设置为了 ret_from_fork, 【x0, 即返回值被设置为了0】。
    6. 分配PID， p->pid = pid_nr(pid)
    7. CLONE_THREAD: 决定是否加入父进程的进程组，不加入则自己是自己组的leader, p->group_leader = p; 
	8. CLONE_PARENT | CLONE_THREAD, p->real_parent = current->real_parent; 
    	// 但是都会在新的数据结构中复制原数据结构的内容	

```

#### some problem

+ 如何获取子进程的当前堆栈信息？若是直接将用户空间堆栈的内容给予内核空间会产生安全问题吗？（用户空间不可信任的堆栈信息？或者只使用部分堆栈中的信息。）
+ 父进程留在内核空间，而子进程返回用户空间。但是使用`SIGCHLD`, 需要对信号处理函数进行修改。
+ **线程的管理，可以查看 irq_thread函数的执行。**
+ **工作队列的bound内核线程不会迁移查看相关设置方法**





子进程死亡需要通知父进程，所以可以利用这一点：

+ 子进程回到用户空间，而父进程留在用户空间
+ 问题：fork函数会直接唤醒子进程，使得子进程的运用CPU未知（还有个问题是如何迁移进程，以及进程间通信）
+ 而留在内核空间的父进程在什么CPU，没有影响（问题：用户空间的子进程如何向用户空间的父进程发送信息）。

