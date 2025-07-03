
:one:冷启动开销

1. 现有的工作：将内存存储至文件中，然后从文件中恢复
2. 为了减小存储恢复的开销，利用硬件(英特尔特有的硬件)进行（解）压缩

> Sabre: Hardware-Accelerated Snapshot Compression for Serverless MicroVMs
>
> Live demonstration: An fpga based hardware compression accelerator for hadoop system.（对于某些应用程序的压缩需求，使用fpga完成数据压缩）
>
> Base-delta-immediate compression: Practical data compression for on-chip caches.（设计了硬件，压缩的第二级cache，增加了cache的利用率）
>
> Sabre: Hardware-Accelerated Snapshot Compression for Serverless MicroVMs 内存快照原文

**点：**

+ 页面换出换入是否可以采用压缩的操作（硬件完成），进一步增加交换空间的大小。或者减小换出需要的传输数据量，减小带宽需求。

+ 冷启动：是否可以添加一些启发式的方法，选择性将内存存储至文件。

+ 内核中是否存在某些任务需要大量计算，可有fpga等硬件加速完成。

+ 压缩KASAN+一些其他机制，优化KASAN的内存占用

  


:two: 内存：

1. 对于分层的内存系统，同时在快内存与慢内存保留某个页面，避免导致的抖动等问题

> Nomad: Non-Exclusive Memory Tiering via Transactional Page Migration

2. 同时支持页面粒度与对象大小粒度访问远端内存

> A Tale of Two Paths: Toward a Hybrid Data Plane for Efficient Far-Memory Application

3. 认为快内存在某些情况下不一定快于慢内存，通过监控内存带宽来均衡快慢内存

> Tiered Memory Management: Access Latency is the Key

**点：**

+ 对于多进程的情况，小内存开销的情况，可能会出现远端访问，是否可以通过某些机制检测会被高频访问的特色页面，然后拷贝到多个numa节点
+ 设计一个库，提供用户空间的定义特殊变量，对象粒度的多备份变量，内核负责通过插桩或者其他形式同步这些变量的值。（对象大小粒度）
+ 



**点：**

+ 反复执行程序，分析程序的执行情况（对于某些可能反复执行的不大的程序），以此定制预取策略
+ 内核的**idle机制**感觉是不是又机会调整一下，默认机制较简单，但是如何测出优势呢
+ 英特尔的一个用户空间中断的硬件，完成用户空间驱动中断处理
+ 针对kasan的细粒度处理计算



工具：

+ 死锁分析工具
+ 并发检查工具





#### Point



`页面分成管理` `影子页面使得迁移时的非阻塞行为（异步）`

`细粒度的二进制插桩`  

>  Harvesting Memory-bound CPU Stall Cycles in Software with MSH

`分页与细粒度的对象操作优化（页面只换或者页面移动细粒度的）`



`全局变量细粒度移动规整到一个页面（编译器是不是做过这个工作了）`



`分布式存储的一致性问题`

:star::star:`NUMA节点的数据副本设计,针对mpi程序，识别高热被跨节点使用的页面然后复制多份`



fpga帮助页面迁徙，页面迁移时的影子页面，使得迁移过程的可访问性



页面迁移时的，增量迁移流程（影子页面），先有迁移是否参考了页面的性质（可执行，只读）



:star: 机器学习用于内核调度（加上蒸馏等乱其八糟的内核，提出一个评估调度性能的平台，或者接口库）

:star:  idle能耗问题机器学习判断是否需要进入深度睡眠

:star: 监控访问字节粒度的数据热度监控

:star: 内存管理粒度

:star: 对于反复执行的程序的页面超预取（反复执行程序的加速，如服务器程序）

:star:  :star: 用户空间的通信机制想时钟机制那样将部分系统调用直接映射到用户空间

