#### schdule

+ `sched_submit_work`:若是状态不为running，则其是否是一个worker, 是则进行处理。然后依据`plug`成员变量，其指向一个队列，保存积累的I/O请求，然后一次性提交。
+ `preempt`++
+ `__schedule`
+ `preempt`--

##### __shedule

> rq队列加减锁，会使preempt的值发生加减

+ 得到rq队列，并依据调度器的特性决定是否取消队列的高精计时器
+ 关中断
+ rq队列加锁，rq队列的时钟进行更新（与调度决策、统计数据相关）
+ 若进程为非TASK_RUNNING，则从rq队列中踢出
+ ==选出一个合适的进程==
+ 在当前rq上记录切入切出的进程信息
+ WRITE_ONCE(rq->membarrier_state, membarrier_state);
+ 释放锁的依赖关系（正常下锁的加锁/释放由同一个进程完成，而进程调度是一个特例）

##### finish_task_switch

+ 获取当前核心的rq队列
+ rq队列解锁
+ 会调用当前rq队列的rq->prev_mm，这个值与`rq`队列对应  :red_circle: :red_circle:





#### note:

