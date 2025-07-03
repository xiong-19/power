### 程序的起始

POT：针对部分指令无法被多个进程共享； PLT: 延迟绑定减小开销

![image-20250323182122346](/home/xiongwanfu/桌面/markdown_fig/image-20250323182122346.png)

先读取前128字节，然后得到文件头，得到程序头，然后读入程序头，然后依据程序头中的信息取出动态连接器，读取动态连接器的文件头，更新mm，设置mmap的起点与方向，设置栈顶。

循环取出每一个LOAD段，设置brk位置，bss依据段的大小获取到位置。

返回到动态连接器的返回地址，延迟绑定动态连接的重定位函数，

然后调用_dl_init中的init函数，进入__start运行一些init_arraay中的函数,然后运行main函数，最后exit



#### 进程：

idle线程，除了第一个的其他的idle都是被fork出来的；

复制：依据各种标志位决定复制复制哪些内容，结构体复制后唤醒唤醒。

调度方面：虚拟时间，update_curr

cgroup由bandwith的剩余时间决定是否被调度

而负载均衡由进程运行的时间，计算出进程的衰减时间，队列的衰减时间以此参与负载均衡。

> 这三个值都会在schedule中出现。

#### 时钟中断：

更新rq的 clock与clock_task; 触发负载均衡；更新当前实体的vruntime与cfs_rq的min_vruntime;

检测是否需要调度：超过理论运行时间；大于最小节点的虚拟时间； 虚拟时间大于理论运行时间。

![81d77977f44e285e13ce67729d55f138](/home/xiongwanfu/桌面/markdown_fig/81d77977f44e285e13ce67729d55f138.png)

![6336665b15681e9d9d722e83386b5fc9](/home/xiongwanfu/桌面/markdown_fig/6336665b15681e9d9d722e83386b5fc9.png)

![image-20241224224147413](/home/xiongwanfu/桌面/markdown_fig/image-20241224224147413.png)





#### cgroup

+ `task_group`本身支持`cfs组调度`和`rt组调度`

+ `task_struct（代表进程）`和`task_group（代表进程组）`中分别包含`sched_entity`，进而来参与调度

内核中的全局结构`root_task_group`为所有task_gruop的根节点，以它构建树状结构。

task_group为每一个进程创建一个se与cfs_rq;

#### cgroup权重

+ tg->shares    -->由sched_group_set_share可以设置
+ cal_group_share: 涉及到load的值

![image-20250320110110744](/home/xiongwanfu/桌面/markdown_fig/image-20250320110110744-17424397059031.png)

+ 需要注意的是reweight_entity中，se->load->weight被设置为了share的值

#### cfs_bandwidth

```c
struct cfs_bandwidth{
    u64 quota, period, runtime; // 限额与周期, 限额剩余时间
    struct hrtimer period_timer, slack_timer; //重新填充运行时间，延迟定时器任务出列时将剩余运行时间返回全局池
}
struct cfs_rq{
    u64 runtime_expires;  // 周期计时器剩余时间
	s64 runtime_remaining; // 剩余的运行时间
}
```

![img](/home/xiongwanfu/桌面/markdown_fig/1771657-20200310214212348-1048763810.png)

##### init_cfs_bandwidth

```c
void init_cfs_bandwidth(struct cfs_bandwidth *cfs_b)
{
	cfs_b->runtime = 0;
	cfs_b->quota = RUNTIME_INF;
	cfs_b->period = ns_to_ktime(default_cfs_period());

	hrtimer_init(&cfs_b->period_timer, CLOCK_MONOTONIC, HRTIMER_MODE_ABS_PINNED);
	cfs_b->period_timer.function = sched_cfs_period_timer;
	hrtimer_init(&cfs_b->slack_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
	cfs_b->slack_timer.function = sched_cfs_slack_timer;
	cfs_b->distribute_running = 0;
}
```

enqueue_entity->check_enqueue_throttle-> .....  -> start_cfs_bandwitch 启动period_timer

dequeue_entity->return_cfs_rq_runtime   ....- > start_cfs_slack_bandwidth 启动slack_timer周期为5ms，将运行队列剩余的时间返回给tg->cfs_b(**在该task_group中平衡各个CPU上的cfs_rq的运行时间`runtime`**)

+ `task_group`针对每个cpu都维护了一个`cfs_rq`，这些`cfs_rq`来共享该`task_group`的限额运行时间

+ period_timer填充后会分配时间给cfs_b中受限的cfs_rq

```c
period_timer:
cfs_b->runtime += cfs_b->quota;
cfs_rq->runtime_remaining += runtime;
/* we check whether we're throttled above */
if (cfs_rq->runtime_remaining > 0)
    unthrottle_cfs_rq(cfs_rq);

slack_timer:
cfs_b->runtime += slack_runtime;
```

> 注意put_prev_entity会检测是否throttle,   cfs_rq->runtime_remaining > 0
>
> enqueue时会调用assign_cfs_rq_runtime分配cfs_b的时间给cfg_rq
>
> update_curr时也会

#### 优先级问题:

>  nice: 对其他进程的友好程度，越不友好越废时间
>
> 0-99实时进程；100-139普通进程；-1 deadline进程
>
> `prio`:调度类考虑的优先级
>
> `static_prio`: 静态优先级，内核不存储nice值，而为static_prio, 不随时间改变
>
> `normal_prio`: 基于`static_prio`计算而，对于普通进程其与`staic_prio`相同，对于实时进程需要重新计算。

​	虚拟时间(虚拟运行时间)：ncie值越小，虚拟时间过得慢

```c
struct load_weight{
	unsigned long weight;
	u32 inv_weight;
}
```



​	nice值[-20,19],没增大1,获取cpu的时间少10%。内核nice为0的权重值为1024,其他由`sched_prio_to_weight`查表获取。

​	$$inv\_weight = 2^{32} / weight $$ 用于计算虚拟时间 $vruntime = exec * nice\_0\_weight / weight$。 exec实际运行时间。



#### 时钟中断：

更新rq的 clock与clock_task; 触发负载均衡；更新当前实体的vruntime与cfs_rq的min_vruntime;

检测是否需要调度：超过理论运行时间；大于最小节点的虚拟时间； 虚拟时间大于理论运行时间。

#### 负债均衡

cpu的计算能力由设备树或者BIOS提供给内核

```c
struct sched_avg {  //实体与CFS就绪队列都包含此结构
	load_sum; 实体：可运行状态下的累计衰减时间； 调队队列：时间x权重
    	
}
对于实体来说load_sum与runnable_load_sum相等都为进程可运行状态的累计衰减时间;util_sum为总算力，需要乘CPU的计算能力。
    
对于队列来说：load_sum, runable_load_sum是所有可运行状态进程的工作负载（时间x权重;util_sum 总算力

实体：load_sum, runable_load_sum都为量化负载， util_avg实际算力
队列： load_sum= sum(se->avg.load_avg) (or running_load_avg); util_avg实际算力
    
xxx_avg都是量化负载;
```

>  衰减系数矩阵runnable_avg_yN_inv; load_avg_yN_org衰减系数

衰减计算分了3级计算方式不同：

__update_load_sum 更新load_sum, load_runnable_sum, util_sum, 依据时间长短的衰减系数

__update_load_avg:



```c
___update_load_sum:
0. 更新 delta = now - sa->last_update_time; sa->last_update_time += delta
1. 计算上一次的sa->load_sum的衰减后值，若这段时间在rq上，则在加上这段时间的衰减值
3. load_sum记录了在rq上的时间，util_sum运行的时间

___update_load_avg
1. sa->load_avg = div_u64(load * sa->load_sum, divider);
2. sa->runnable_avg = div_u64(sa->runnable_sum, divider);
	WRITE_ONCE(sa->util_avg, sa->util_sum / divider);

__update_load_avg_cfs_rq
1.  计算与实体的计算相同, load_sum（load为权重值，runnable为挂载的实体数, running为当前cfs_rq是否被运行）
```

![image-20250409144352642](/home/xiongwanfu/桌面/markdown_fig/image-20250409144352642.png)

