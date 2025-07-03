
:one:冷启动开销

1. 现有的工作：将内存存储至文件中，然后从文件中恢复
2. 为了减小存储恢复的开销，利用硬件(英特尔特有的硬件)进行（解）压缩

> Sabre: Hardware-Accelerated Snapshot Compression for Serverless MicroVMs 原文
>
> Live demonstration: An fpga based hardware compression accelerator for hadoop system.（对于某些应用程序的压缩需求，使用fpga完成数据压缩）
>
> Base-delta-immediate compression: Practical data compression for on-chip caches.（设计了硬件，压缩的第二级cache，增加了cache的利用率）



**点：**

+ 页面换出换入是否可以采用压缩的操作（硬件完成），进一步增加交换空间的大小。或者减小换出需要的传输数据量，减小带宽需求。

> 有意义吗？

内核中是否存在某些任务需要大量计算，可有fpga等硬件加速完成。

> 是否存在这种需求



+ 压缩（Kernel Address Sanitizer）+一些其他机制，优化KASAN的内存占用（后台线程完成不影响速度）

> 感觉还可以  :star:



+ 反复执行程序，分析程序的执行情况（对于某些可能反复执行的不大的程序），以此定制预取策略

+ 上需：或者直接分配巨页给高频访问的页面，浪费空间（并想办法复用浪费的巨页），增加执行效率  :star:（再添加冷启动）

> 对于某些特殊的应用程序可能比较有效（克服cow的不好之处） :star:



:notebook:

+ 英特尔的一个用户空间中断的硬件

> 用户空间直接完成调度，进程间通信
>
> 是否存在某些中断的常见，直接在用户空间处理中断的内容，避免陷入内核






:two: 内存：

1. 对于分层的内存系统，同时在快内存与慢内存保留某个页面，避免导致的抖动等问题

> Nomad: Non-Exclusive Memory Tiering via Transactional Page Migration

2. 同时支持页面粒度与对象大小粒度访问远端内存

> A Tale of Two Paths: Toward a Hybrid Data Plane for Efficient Far-Memory Application

3. 认为快内存在某些情况下不一定快于慢内存，通过监控内存带宽来均衡快慢内存

> Tiered Memory Management: Access Latency is the Key





**点：**

+ 对于多进程的情况，小内存开销的情况，可能会出现远端访问，是否可以通过某些机制检测会被高频访问的特色页面，然后拷贝到多个numa节点

> 争对只读页面，若是所有页面，这就和cache的作用重复了感觉  :star:

+ ~~设计一个库，提供用户空间的定义特殊变量，对象粒度的多备份变量，内核负责通过插桩或者其他形式同步这些变量的值。（对象大小粒度）~~  :star:

> 有意义吗？



**点：**

+ 内核的**idle机制**感觉是不是有机会调整一下，默认机制较简单，但是如何测出优势呢

  

工具：

+ 死锁分析工具
+ 并发检查工具



ERASAN: Efficient Rust Address Sanitizer

用户空间的ASAN uprobe+其他机制

37，53，56，57



Gwp-asan: Sampling heap memory error detection in-the-wild

SANRAZOR: Reducing redundant sanitizer checks in C/C++ programs

Debloating address sanitizer.
