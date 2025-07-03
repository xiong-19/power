```c
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;
	};
} spinlock_t;


typedef struct raw_spinlock {
	arch_spinlock_t raw_lock;
} raw_spinlock_t;


typedef struct {
	union {
		u32 slock;
		struct __raw_tickets {
#ifdef __ARMEB__
			u16 next;
			u16 owner;
#else
			u16 owner;
			u16 next;
#endif
		} tickets;
	};
} arch_spinlock_t;

static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}

static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

```c
/*
	约定：
	1）当读取值为1时，内核开始工作，工作完成将值修改为 0
	2）当读值为0时，将值+2, 用户空间可传递数据， 传递完成 -1
*/
int signal;
int tmp;
int newval;

// 内核部分
// 循环原子读，直到读到1
while(1){
	asm volatile(
		"1:	ldrex	%0, [%4186112]\n"
		"	strex	%1, %0, [%4186112]\n"
		"	teq	%1, #0\n"
		"	bne	1b"
		: "=&r" (signal), "=&r" (tmp)
		:
		: "cc");
	if(sinal)break;
    schedule();
}

// 原子减1
asm volatile(
    "1:	ldrex	%0, [%4186112]\n"
    "   sub %2, %0, #1\n"
    "	strex	%1, %2, [%4186112]\n"
    "	teq	%1, #0\n"
    "	bne	1b"
    : "=&r" (signal), "=&r" (tmp), "=&r"(new)
    :
    : "cc");


// 用户空间部分
// 循环原子读，直到读到0, +2
int signal;
int tmp;
int newval;


/*
分析：
	读得不为0：
		此间不被其他进入，则正常写回，且经判断不为0, 重新读值
		此间被其他进入，写回异常，重新读值
	读得为0：
		此间不被其他进入，则正常+2, 且写回，由于读值为0,最下面判断不生效
		此间被其他进入，则虽然+2, 但是写回异常，重新读值
可能问题：1. 其他进程的反复读值，导致+2写不进去？？ 2. 体系结果的优化，可能产生的影响未使用到屏障？？（ 是否需要添加"dmb     ish"指令）

*/
asm volatile(
    "0: wfe"
    "1:	ldrex	%0, [%4186112]\n"
    "   teq %0, #0\n"
    "   bne 2f\n"
    "   add %2, %0, #2\n"
    "2:	strex	%1, %2, [%4186112]\n"
    "	teq	%1, #0\n"
    "	bne	1b"
    "   teq %0, #0\n"
    "   bne 1b\n"
    : "=&r" (signal), "=&r" (tmp), "=&r"(newval)
    :
    : "cc");


asm volatile(
    "1:	ldrex	%0, [%4186112]\n"
    "   sub %2, %0, #1\n"
    "	strex	%1, %2, [%4186112]\n"
    "	teq	%1, #0\n"
    "	bne	1b"
    : "=&r" (signal), "=&r" (tmp), "=&r"(new)
    :
    : "cc");



```



````c
	__asm__ __volatile__(
"1:	ldrex	%0, [%3]\n"
"	add	%1, %0, %4\n"
"	strex	%2, %1, [%3]\n"
"	teq	%2, #0\n"
"	bne	1b"
	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
	: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
	: "cc");



#define wfe()		asm volatile("wfe" : : : "memory")  // 内核进程核不能睡眠

#define smp_mb() barrier();
#define barrier() __asm__ __volatile__("": : :"memory")

// 内核中部分
	__asm__ __volatile__(
"1:	ldrex	%0, [%3]\n"
"	add	%1, %0, %4\n"
"	strex	%2, %1, [%3]\n"
"	teq	%2, #0\n"
"	bne	1b"
	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
	: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
	: "cc");

u64 owner;
u64 * owneraddr = 0x400000 - 2*4096;  // 4186112
u64 * nextaddr = 0x400000 - 2*4096+4；//4186116
    
	__asm__ __volatile__(
"1:	ldrex	%0, [#4186116]\n"
"	add	%1, %0, #1\n"
"	strex	%2, %1, [#4186116]\n"
"	teq	%2, #0\n"
"	bne	1b"
	: "=&r" (old)
	: 
	: "cc");
while(owner != *nextaddr){
    asm volatile("wfe" : : : "memory")
    owner = * owneraddr;
}
__asm__ __volatile__("": : :"memory")
````

