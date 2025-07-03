#### 1. 基本信息

​	目前 Linux 支持64种信号。信号分为非实时信号(不可靠信号)和实时信号（可靠信号）两种类型，对应于 Linux 的信号值为 1-31 和 32-64。非实时信号多次发送进程只能接送一次； 而实时信号支持队列，信号不会丢失。

```c
struct task_struct {
 ...
 int sigpending;  // 是否有信号待处理。1或者0。
 ...
 struct signal_struct *sig;
 sigset_t blocked;	// 屏蔽的信号。
 struct sigpending pending;  // 进程接收到的信号队列。
 struct sighand_struct		*sighand;
 ...
}

struct signal_struct {
	struct sigpending shared_pending; 
};

struct sighand_struct {
	atomic_t		count;
	struct k_sigaction	action[_NSIG];
	spinlock_t		siglock;
	wait_queue_head_t	signalfd_wqh;
};

typedef void (*__sighandler_t)(int);
struct sigaction {	
	__sighandler_t sa_handler;
	unsigned long sa_flags;
	sigset_t sa_mask;
};
struct k_sigaction {
	struct sigaction sa;
};


struct sigqueue {
	struct sigqueue *next;
	siginfo_t info;
};
struct sigpending {
	struct sigqueue *head, **tail;
	sigset_t signal;
};
// 当进程接收到一个信号后，便需要将信号添加到pending队列中。

```
![img](https://pic1.zhimg.com/80/v2-2ea6064531fc93880eb6ac6b213307e0_1440w.webp)

#### 2. 信号的发生

​	信号的生成可由`内核`产生，也可以由`用户 `产生。

+ 程序除0产生异常生成`SIGFPE`信号, 非法访问内存产生`SIGBUS`信号。
+ 终端：`kill -9 [pid]`; 进程： `kill, raise函数`。

```c
int kill(pid_t pid, int sig);

系统调用：
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct siginfo info;
	
	info.si_signo = sig;
	info.si_errno = 0;
	info.si_code = SI_USER;
	info.si_pid = task_tgid_vnr(current);
	info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
	return kill_something_info(sig, &info, pid);
}
```

```c
// pid > 0, 发送信号给pid对应进程。
// pid = 0, 发送给本进程组中的所有进程
// pid = -1, 发送给除init外的其他进程
// pid < -1, 发送给进程组id为 -pid 的所有进程
static int kill_something_info(int sig, struct siginfo *info, pid_t pid){
	if(pid > 0)ret = kill_pid_info(sig, info, find_vpid(pid));
	if(pid != -1)ret = __kill_pgrp_info(sig, info, pid?find_vpid(-pid):task_pgrp(current));
	else {
		for_each_process(p){
			err = group_send_sig_info(sig, info, p, PIDTYPE_MAX);
		}
	} 
}
```

```c
int kill_pid_info(int sig, struct siginfo *info, struct pid *pid)
{
	for(;;){
        if (p)
			error = group_send_sig_info(sig, info, p, PIDTYPE_TGID);
    }
}
group_send_sig_info
    |--> ret = do_send_sig_info(sig, info, p, type);
		|--> if (lock_task_sighand(p, &flags)) {
				ret = send_signal(sig, info, p, type);
				unlock_task_sighand(p, &flags);
			}

```

```c
static int __send_signal(int sig, struct siginfo *info, struct task_struct *t,
			enum pid_type type, int from_ancestor_ns)
{
    prepare_signal(sig, p, force);
    /* 是否信号可以被传输：
       对于特殊的信号处理：如当前进程将要退出而信号为SIGKILL; 若信号为停止类的信号，则清空所有队列的所有SIGCONT信号；信号为SIGCONT，则清空所有STOP类信号
    */
	pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
    if (legacy_queue(pending, sig))
        goto ret;  // 若信号为不可靠信号，且已经在padding队列中了，则忽略这个信号
    if (sig < SIGRTMIN)
		override_rlimit = (is_si_special(info) || info->si_code >= 0);
	else
		override_rlimit = 0;
    struct sigqueue *q;
	q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit); // 分配一个
    if(q){
        list_add_tail(&q->list, &pending->list); // 加入pending队列中
        // 核心的一点kthreadd_done
        q->info.si_signo = sig;
    }
    signalfd_notify(t, sig);  // 用于在有信号发生时，通知那些通过 signalfd 注册了信号监听的进程，唤醒他们，使他们可以读取并处理这些信号。
    	|--> if(waitqueue_active(&tsk->sighand->signalfd_wqh))
            	wake_up(&tsk->sighand->signalfd_wqh)		// 队列是否不为空
            
    sigaddset(&pending->signal, sig);  // 设在对应标志位，表示信号被接收到。
    complete_signal(sig, t, type);  
    /* 决定哪个线程应该处理接收到的信号，并采取一些措施。
       在一些情况下是不需要唤醒。信号传输进程的，而在一些情况下需要将信号传输的进程唤醒。
       对于致命的信号，则应该唤醒。
    */.

}
```

#### 3. 内核的处理过程

​	信号屏蔽字保存在`task_struct`的`blocked`结构中。

```c
ret_to_user:
	|-> work_pending
        |-> do_notify_resume
        

static void do_signal(struct pt_regs *regs){
    bool syscall = in_syscall(regs); // 是否在系统调用中； 即resgs->syscallno值
    if(syscall)do something; // 作一些处理
        
    struct ksignal ksig;
    if(get_signal(&ksig)){
        ...
        handle_signal(&kskthreadd_doneig, regs);
    }    
}
```

```c
bool get_signal(struct ksignal *ksig){
    struct sighand_struct *sighand = current->sighand;
	struct signal_struct *signal = current->signal;
    
    if((sig->flags & SIGNAL_GROUP_EXIT) || (sig->group_exit_task != NULL);){
        ksig->info.si_signo = signr = SIGKILL;
		sigdelset(&current->pending.signal, SIGKILL);
    }
    /* 从队列中移除并处理信号，包括同步信号和异步信号。根据信号处理器（SIG_IGN、SIG_DFL或用户自定义）来决定如何响应信号。如果信号被忽略，继续处理下一个；如果使用默认处理器且信号应停止进程，执行停止操作。 */
    for(::){
        struct k_sigaction *ka;
        
        // 先判断同步信号，再处理普通信号
        
        signr = dequeue_synchronous_signal(&ksig->info); //从当前的带处理信号队列中查找并处理同步信号。 1) 在current->pending中查找是否存在同步信号（pending->sig[0]帮助判断）， 同步信号包含6个。没有则直接返回。 2)遍历pending->list, 找到同步信号的sigqueue q; 3) 找到其后是否有找到q, q->info.si_signo相同的信号，有跳转到5 4)将q->pending->signal置位当前信号 5)将q从队列删除
		if (!signr)
			signr = dequeue_signal(current, &current->blocked, &ksig->info);
       	/*
			signr = __dequeue_signal(&tsk->pending, mask, info, &resched_timer);       		 |--> int sig = next_signal(pending, mask);
			|--> if (sig) collect_signal(sig, pending, info, resched_timer);
       					|--> 找到一个q, q->info.si_signo == sig, 并将其从队列中删去， 将q->info赋予info, pending->signal中删去
        */
        if(!signr)break; // 当信号处理完，则退出循环
        
        ka = &sighand->action[signr-1];
        if (ka->sa.sa_handler == SIG_IGN) /* Do nothing.  */
			continue;
		if (ka->sa.sa_handler != SIG_DFL) {
			/* Run the handler.  */
			ksig->ka = *ka;

			if (ka->sa.sa_flags & SA_ONESHOT)
					ka->sa.sa_handler = SIG_DFL;
			break; /* will return non-zero "signr" value */
		}
        // 其后还有一些特殊信号的处理
    }
    ksig->sig = signr;
    return ksig->sig > 0;
}
```

![image-20240817161505764](/home/xiongwanfu/桌面/markdown_fig/image-20240817161505764.png)

```c
static void handle_signal(struct ksignal *ksig, struct pt_regs *regs){
    if (is_compat_task()) {
        if (ksig->ka.sa.sa_flags & SA_SIGINFO)
            ret = compat_setup_rt_frame(usig, ksig, oldset, regs);
        else
            ret = compat_setup_frame(usig, ksig, oldset, regs);
    } else {
        ret = setup_rt_frame(usig, ksig, oldset, regs); // 64位调用此函数
        /* 
        	1. get_sigframe得到regs中的sp， sp -= rt_sigframe的大小
        	2. 将sa_restore按照函数栈的规则放入frame->pretcode中。当sa_handler执行完成后, 弹出frame, 便跳转回sa_restore中。 【setup_sigframe】
        	3. 将regs->ip设置为sa_handler, regs->sp = frame的地址 【setup_return】
        */
    }

}
```

![img](https://static001.geekbang.org/resource/image/3d/fb/3dcb3366b11a3594b00805896b7731fb.png)

#### 4. 信号注册

```c
// 库函数
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);

int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
// 调用do_sigaction, rt_sigaction
```

![set-sa](https://imgconvert.csdnimg.cn/aHR0cHM6Ly93c2ctYmxvZ3MtcGljLm9zcy1jbi1iZWlqaW5nLmFsaXl1bmNzLmNvbS94ZW5vbWFpL3NldC1zYS5wbmc?x-oss-process=image/format,png)

#### 5 . 参考

> [1] https://ty-chen.github.io/linux-kernel-signal/
>
> [2] https://blog.csdn.net/qq_22654551/article/details/107416813
>
> [3] https://kernel.meizu.com/2016/07/04/linux-signal.html/  ==魅族==  :star2: :star2: :star2: :star2: :star2:

#### some record

1. 在内核线程中注册信号处理函数：在内核线程代码中，使用`signal()`或`sigaction()`函数注册一个信号处理函数，用于处理接收到的信号。
2. 在用户空间程序中发送信号：用户空间程序可以使用`kill()`系统调用向内核线程发送信号，使用内核线程的进程ID（PID）作为目标。例如，`kill(threakthreadd_doned_pid, SIGUSR1)`会向指定的内核线程发送SIGUSR1信号。
3. 在信号处理函数中唤醒内核线程：在信号处理函数中调用适当的唤醒函数，以唤醒处于睡眠状态的内核线程。具体的唤醒函数取决于您的内核线程实现和需求，可以使用`wake_up_process()`等函数。



信号处理时机

1. 同步处理：某些信号（如SIGSEGV、SIGFPE等）会立即被内核处理，即中断当前的执行流程，并在内核空间中执行相应的信号处理程序。这是因为这些信号通常表示了一些严重的错误或异常情况，需要立即处理。
2. 异步处理：对于大多数信号，内核不会立即处理，而是将其标记为待处理状态，并在合适的时机进行处理。合适的时机包括但不限于以下情况：
   - 调用系统调用：当进程进行系统调用时，内核会检查是否有待处理的信号需要处理。如果有，内核会在系统调用返回后先处理信号，然后再继续执行进程的代码。
   - 返回用户空间：当一个内核线程完成其任务并准备返回用户空间时，内核会检查是否有待处理的信号需要处理。如果有，内核会在返回用户空间前先处理信号。
   - 定时处理：内核会周期性地检查是否有待处理的信号需要处理。这样可以确保即使在进程没有调用系统调用或返回用户空间的情况下，信号也能被及时处理。



​	当时钟中断发生时，内核会停止当前正在执行的任务，并根据中断处理程序（IRQ）的设置进行处理。其中，一个重要的处理是检查是否有待处理的信号需要被处理。

​	在这个时钟中断处理程序中，内核会检查进程的信号屏蔽字（signal mask）和挂起的信号队列。如果有一个信号不在屏蔽字中，并且进程有对该信号的注册处理函数，那么内核会调用相应的信号处理函数来处理该信号。
