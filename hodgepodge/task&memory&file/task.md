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
入口 __head:  xo=FDT blob     arc/arm64/kernel/head.S
	直接跳转到stext
stext：
	1. 将引导参数保存至boot_args变量中; x1到x4保存到 boot_args变量中;
	2. el2_setup: 
		（1）依据启动的等级，同时是否支持虚拟化扩展决定是否在el2执行内核。否则el1执行内核。（2）el2包含物理定时器，gicv3的设置，处理器ID，调试，性能监控相关寄存器，是否支持大页映射。
                       
	3. 初始化页表
```

> VHE: KVM(一个内核模块)将内核变为虚拟机监控程序。
>
> ```c
> // QEMU创建一个KVM虚拟机，与虚拟机的交互方式如下：
> fd=open("/dev/kvm", O_RDWR);
> wmfd = ioctl(fd, KVM_CREATE_NM, 0);
> vcpu_fd = ioctl(vmfd, KVM_CREATE_VCPU, 0); // 每个虚拟机处理器是一个线程(一个kvm_vcpu结构体);
> 
> ioctl(vcpu_id, KVM_RUN, 0); // QEMU进程经入内核，KVM模块获取到对应 vCPU 的上下文，KVM 模块准备虚拟 CPU 状态。 KVM执行KVM_RUN由el1->el2（首先保存保存所有寄存器，然后将寄存器的值设置为客户操作系统的寄存器的值，由el2到el1执行客户操作系统）
> 
> // 若是支持VHE：陷入内核后便是el2, 只需要el2->el1
> 
> 注：
>     1. HCR_EL2中，hypervisor明确设置了应当拦截guest操作中的哪些操作或者异常。当硬件检测到时会将异常自动转移至el2,而不是guset的el1或者el0(硬件标记了当前为guest环境)；
> 	2. 无论当前的vcpu进程运行与el0或者el1,当中断或者异常事件发生时，(1)硬件处理:保存guset的信息如pc,异常信息；进入el2；(2):跳转至el2的异常向量，将通用寄存器等保存至vCPU内存结构；(3)运行vmexit处理；(4)返回至vCPU的函数，退出guest状态，进一步处理中断或者异常；(5)完成中断或者异常处理后，恢复guest上下文(eret完成)(vmentry);
> 	*. 异常处理内容根据需要可可能(1)通过vmentry转到guest中执行；(2)通过异常返回机制返回至宿主内核执行的el1；
>         但是所有的处理化处理都是在el2的hypervisor处理完成的。
>         
> ```



```c
4. __primary_switch:
	(1) 将init_task写入sp_el0, 并设置init的栈;sp=intit_task->stack + THREAD_SIZE - PT_REGS_SIZE + S_STACKFRAME;
	(2) 将init_task的__per_cpu_offset[cpu]写入tpidr_el1;
	(3) bl	start_kernel;
```

```c
cgroup_init_early:
	init_cgroup_root(): // 设置了cgroup_root的cgroup变量的各个变量，主要是初始化各个链表。包含各个资源子系统的链表。 
						// init_task.cgroups = &init_css_set
						// 初始化各个资源子系统

```

