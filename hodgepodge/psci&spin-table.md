

#### 启动部分

+ soc启动的一般会从片内的rom， 也叫bootrom开始执行第一条指令，这个地址是系统默认的启动地址，会在bootrom中由芯片厂家固化一段启动代码来加载启动bootloader到片内的sram。

+ 启动完成后的bootloader除了做一些硬件初始化之外做的最重要的事情是初始化ddr,因为sram的空间比较小所以需要初始化拥有大内存 ddr,最后会从网络/usb下载或从存储设备分区上加载内核到ddr某个地址，为内核传递参数之后
+ 跳转到内核，就进入了操作系统内核的世界。



![cf430616e8672f9f723a83bdd6e2991d](/home/xiongwanfu/桌面/markdown_fig/cf430616e8672f9f723a83bdd6e2991d.png)



> secondary cpu会为内核准备一块**内存**，用于填写secondary cpu的**入口地址**。
>
> uboot负责将这块内存地址写入device tree，内核完成初始化后。将内核的入口地址写入此内存

```assembly
// uboot内容
#if defined(CONFIG_ARMV8_SPIN_TABLE) && !defined(CONFIG_SPL_BUILD)
	branch_if_master x0, x1, master_cpu                  
	b	spin_table_secondary_jump                    // 其他核
	…
master_cpu:                                                  
	bl	_main   //主核
```

```assembly
ENTRY(spin_table_secondary_jump)
.globl spin_table_reserve_begin
spin_table_reserve_begin:
0:	wfe                                           
	ldr	x0, spin_table_cpu_release_addr      
	cbz	x0, 0b                                
	br	x0                                   
.globl spin_table_cpu_release_addr                    
	.align	3
spin_table_cpu_release_addr:
	.quad	0
.globl spin_table_reserve_end
spin_table_reserve_end:
ENDPROC(spin_table_secondary_jump)
```

​	注意：spin_table_cpu_release_addr将由uboot传递给内核。

​	**具体方式**：将`spin_table_cpu_release_addr`写入设备树的``cpu-release-addr`中。(当所有的CPU的`enable-method`为`spin-table`)



#### SPIN-TABLE

**主核**

![c4933e536ca3f5a683344d985449f5a9](/home/xiongwanfu/桌面/markdown_fig/c4933e536ca3f5a683344d985449f5a9.png)

​	完成执行环境的准备后：

+ 非主核CPU分配idle
+ 每个CPU分配hotplug

==主核如何唤醒其他核心的hotplug？==

```c
// cpu_ops的设置， init_cpu_ops(int cpu)
const struct cpu_operations smp_spin_table_ops = {
	.name		= "spin-table",
	.cpu_init	= smp_spin_table_cpu_init,
	.cpu_prepare	= smp_spin_table_cpu_prepare,
	.cpu_boot	= smp_spin_table_cpu_boot,
}
// 在setup_arch->smp_init_cpus->cpu_ops[cpu]->cpu_init(cpu)
// 完成设备树的"cpu-release-addr"写入cpu_release_addr[cpu]

// kernel_init->smp_prepare_cps->err = cpu_ops[cpu]->cpu_prepare(cpu);
// 将secondary_holding_pen写入cpu_release_addr[cpu]中

[**] uboot中的跳转位置已经被设置
secondary_holding_pen:
主要是判断secondary_holding_pen_release值是否与当前CPU的值相等，是则跳转secondary_startup
设置sp_el0为其idle后，跳转到secondary_start_kernel
然后进入idle    
```

> secondary_holding_pen 一段汇编 。

```c
// smp_init中
for_each_present_cpu(cpu) {
	if (num_online_cpus() >= setup_max_cpus)
		break;
	if (!cpu_online(cpu))
		cpu_up(cpu); |--> return do_cpu_up(cpu, CPUHP_ONLINE);
}

static int do_cpu_up(unsigned int cpu, enum cpuhp_state target){
    ...
    _cpu_up(cpu, 0, target); |--> cpuhp_kick_ap_work(cpu) // 唤醒CPU初始状态st的thread线程
}
// 该线程完成其他核的OFFLINE到BRINGUP_CPU

```

```c
// 达到BRINGUP状态，主核运行该状态的回调函数
static int bringup_cpu(unsigned int cpu){
    ...
    ret = __cpu_up(cpu, idle); |--> ret = boot_secondary(cpu, idle);
}

boot_secondary 
|--> cpu_ops[cpu]->cpu_boot(cpu);
//变量secondary_holding_pen_release中写入逻辑cpu的id
|--> sev(); //唤醒其他核
```

![81d77977f44e285e13ce67729d55f138](/home/xiongwanfu/桌面/markdown_fig/81d77977f44e285e13ce67729d55f138.png)









#### PSCI

> 以支持所有cpu相关的电源管理接口，而且由于电源相关操作是系统的关键功能，为了防止其被攻击，该协议将底层相关的实现都放到了secure空间，从而可提高系统的安全性。
>
> 工作于non secure EL1（linux内核）和EL3（bl31）之间的一组电源管理接口，其目的是**让linux实现具体的电源管理策略**，而**由bl31管理底层硬件相关的操作**。
>
> 支持smc与hvc两种值令。

电源状态：4种power domain

+ run：电源和时钟都打开，该domain正常工作。
+ standby：关闭时钟，但电源处于打开状态。执行wfi或wfe指令会进入该状态。
+ retention：它将core的状态，包括调试设置都保存在低功耗结构中。
+ power down：关闭时钟和电源。

##### PSCI驱动设置

start_kernel->setup_arch->psci_dt_init：调用以下三个函数中的一个

```c
static const struct of_device_id psci_of_match[] __initconst = {
	{ .compatible = "arm,psci",	.data = psci_0_1_init},
	{ .compatible = "arm,psci-0.2",	.data = psci_0_2_init},
	{ .compatible = "arm,psci-1.0",	.data = psci_1_0_init},
	{},
};
```

设置psci_ops回调函数。

```c
const struct cpu_operations cpu_psci_ops = {
	.name		= "psci",
	.cpu_init	= cpu_psci_cpu_init,
	.cpu_prepare	= cpu_psci_cpu_prepare,
	.cpu_boot	= cpu_psci_cpu_boot,
#ifdef CONFIG_HOTPLUG_CPU
	.cpu_can_disable = cpu_psci_cpu_can_disable,
	.cpu_disable	= cpu_psci_cpu_disable,
	.cpu_die	= cpu_psci_cpu_die,
	.cpu_kill	= cpu_psci_cpu_kill,
#endif
};
//在cpu_boot中会调用psci驱动中的psci_ops函数 
```

PSCI初始化：

......

##### psci中第二个核心启动

> psci的前两个函数init与prepare都为空。

与SPIN_TABLE的区别：为调用`cpu_psci_cpu_boot`函数![c7c396e47419a93c7fb32b1c69a00189](/home/xiongwanfu/桌面/markdown_fig/c7c396e47419a93c7fb32b1c69a00189.png)

![b84f4c94a6567e108530762be798399b](/home/xiongwanfu/桌面/markdown_fig/b84f4c94a6567e108530762be798399b.png)

>  pwr_domain_on将为secondary cpu设置入口函数，然后为其上电使该cpu跳转到内核入口secondary_entry处开始执行。

```c
int err = psci_ops.cpu_on(cpu_logical_map(cpu), __pa_symbol(secondary_entry));
psci_ops.cpu_on = psci_cpu_on;
// 平台相关回调函数pwr_domain_on将为secondary cpu设置入口函数，然后为其上电使该cpu跳转到内核入口secondary_entry处开始执行。
// 因为可以控制上点的CPU所以不需要，想SPIN_TABLE那样把当前需要唤醒的CPU ID写入一片地址中。
```

![6336665b15681e9d9d722e83386b5fc9](/home/xiongwanfu/桌面/markdown_fig/6336665b15681e9d9d722e83386b5fc9.png)

##### EL3逻辑

+ 首先设置psci平台相关的操作函数
+ smc指令进入EL3异常向量表

```asciiarmor
blr     x15  //跳转到处理函数
b       el3_exit  //从ｅｌ3退出  会ｅｒｅｔ 回到ｅｌ1 （后面会讲到）
```

+ 跳转到psci_smc_handler中，调用`ret = (u_register_t)psci_cpu_on(x1, x2, x3);`

  > 所有的从处理器启动后都会从bl31_warm_entrypoint开始执行，在plat_setup_psci_ops中会设置（每个平台都有自己的启动地址寄存器，通过写这个寄存器来获得上电后执行的指令地址）

  + 上电，通过cpu的编号找到cpu上下文，并设置

```c
write_ctx_reg(state, CTX_ELR_EL3, ep->pc);  
write_ctx_reg(state, CTX_SPSR_EL3, ep->spsr);
```



#### 上下线流程

![image-20241224224147413](/home/xiongwanfu/桌面/markdown_fig/image-20241224224147413.png)



#### 参考

> https://blog.csdn.net/weixin_45264425/article/details/127893776

`echo 0 > /sys/devices/system/cpu/cpu1/online`

思路：向online操作的调用路径

![image-20241225110458446](/home/xiongwanfu/桌面/markdown_fig/image-20241225110458446.png)

先加锁，然后检查是否可以offline，然后直接调用cpu_subsys_offline，然后再释放锁

```
lock_device_hotplug_sysfs()
device_offline()
unlock_device_hotplug()

ret = dev->bus->offline(dev);
if (!ret) {
kobject_uevent(&dev->kobj, KOBJ_OFFLINE);
dev->offline = true;
}

关键他会判断一个hotplug是不是为0
```

```c
cpu_ops[cpu]->cpu_die(cpu);
.cpu_can_disable = cpu_psci_cpu_can_disable,
.cpu_disable	= cpu_psci_cpu_disable,
.cpu_die	= cpu_psci_cpu_die,
.cpu_kill	= cpu_psci_cpu_kill,
```



分析:

:star2:**.disable:**

​	cpu_psci_cpu_disable——查看当前cpu是否为resident_cpu  ==该变量为psci_probe中初始化的==

​	take_cpu_down->`__cpu_disable`  ->op_cpu_disable:  take_cpu_down要求其返回值不小于0

:star2:**can_disbale:**

​	topology_init->cpu_can_disable——设置cpu->hotpluggable标记其是否可以下线

​	cpu->hotpluggable在register_cpu中被调用。注意device_offline中也会调用到这个函数。但是下线函数可以直接绕过这个判断。

​	device_add中也涉及到这个变量，用于判断是否创建device_create_file文件

:star2: **die:**

​	cpu_die：cpu将要关闭前夕。

​	cpu_die_early: 

>  Kill the calling secondary CPU, early in bringup before it is turned online.

​	ipi_cpu_crash_stop：核间中断

:star2:**kill:**

​	__cpu_up->op_cpu_kill——没有该函数则直接返回0

​	takedown_cpu->_cpu_die->op_cpu_kill——等5s，等是不是真的死了



修改：

1. kernel/sysctl.c   2642  and 349 添加了文件处理

2. arch/arm64/kernel/smp_spin_table.c 119 补充了涉及的spin-table函数
3. drivers/base/core.c   2669  下线函数

