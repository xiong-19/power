#### 内核启动 :happy:

+ 机器上电，将PC设置为0（arm上此设置），此时MMU未开启。对于嵌入式设备来说，物理地址0为NOR闪存。

![img](../markdown_fig/3bcab1d0e360fb13e3e4a79a0597fb63.png)

 +  设备上电或复位后，处理器首先从内部不可修改的 **BootROM** 开始执行。会检测启动介质（比如SD卡、eMMC、SPI Flash等）的状态。将启动介质中的Miniloader（SPL:uboot的前部分）加载到内存中（可能是内部**SRAM**）；

> 0地址为映射到了芯片内部的BootROM区域

+ 执行Miniloader：完成初始化DDR控制器；加载并启动第二阶段引导程序（U-Boot）

 +  执行uboot：负责后续的硬件初始化、设备树(Device Tree)的加载与传递、并最终加载Linux内核镜像与DTB。

uboot的两种启动方式：见[启动](../psci&spin-table.md)

##### 内核开始

```c
入口 __head:
	直接跳转到stext
stext：
	1. 将引导参数保存至boot_args变量中;
	2. 初始化页表
```

