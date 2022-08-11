```C
//kernel/fork.c
long do_fork(unsigned long clone_flags,
             unsigned long stack_start,
             struct pt_regs *regs,
             unsigned long stack_size,
             int __user *parent_tidptr,
             int __user *child_tidptr)
```

`asmlinkage void __exception asm_do_IRQ(unsigned int irq, struct pt_regs *regs)`

```assembly
.macro	irq_handler
ldr_l	x1, handle_arch_irq @ ------  (1)
mov	x0, sp
irq_stack_entry  @ ------  (2)
blr	x1     @ ------ (3)
irq_stack_exit   @ ------ (4)
.endm
```
(1) 将handle地址保存在x1
(2)  切换栈，也就是将svc栈切换程irq栈. 在此之前，SP还是EL1_SP，在此函数中，将EL1_SP保存，再将IRQ栈的地址写入到SP寄存器
(3) 执行中断处理函数
(4) 恢复EL1_SP(svc栈)


```assembly
/*
 * x19 should be preserved between irq_stack_entry and
 * irq_stack_exit.
 */
.macro	irq_stack_exit
//x19保存着svc mode下的栈，也就是EL1_SP
	mov	sp, x19     
.endm

```

1.在驱动中定义一个static struct fasync_struct *async;
2.在fasync系统调用中注册fasync_helper(fd, filp, mode, &async);
3.在中断服务程序（顶半部、底半部都可以）发出信号kill_fasync(&async, SIGIO, POLL_IN);
4.在用户应用程序中用signal注册一个响应SIGIO的回调函数signal(SIGIO, sig_handler);
5.通过fcntl(fd, F_SETOWN, getpid())将将进程pid传入内核
6.通过fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | FASYNC)设置异步通知


[^1]:[]()
[^2]:[]()
[^3]:[]()
[^4]:[]()
[^5]:[]()
[^6]:[]()
[^7]:[]()
[^8]:[]()
[^9]:[]()
[^10]:[]()
[^11]:[]()
[^12]:[]()
[^13]:[]()
[^14]:[]()
[^15]:[]()
[^16]:[]()
[^17]:[]()
[^18]:[]()
[^19]:[]()
[^20]:[]()

内核中提供了等待事件`wait_event()`宏（以及它的几个变种），可用于实现简单的进程休眠，等待直至某个条件成立，主要包括如下几个定义：

```C
<binfmts.h>
struct linux_binfmt {
    struct linux_binfmt * next;
    struct module *module;
    int (*load_binary)(struct linux_binprm *, struct pt_regs * regs);
    int (*load_shlib)(struct file *);
    int (*core_dump)(long signr, struct pt_regs * regs, struct file * file);
    unsigned long min_coredump; /* minimal dump size */
};
```

| 架构   | 实现                                                         |
| ------ | ------------------------------------------------------------ |
| arm    | [arch/arm/kernel/sys_arm.c, line 254](http://lxr.free-electrons.com/2.4.31/source/arch/arm/kernel/sys_arm.c?v=#L254) |
| i386   | [arch/i386/kernel/process.c, line 737](http://lxr.free-electrons.com/source/arch/i386/kernel/process.c?v=2.4.31#L737) |
| x86_64 | [arch/x86_64/kernel/process.c, line 728](http://lxr.free-electrons.com/source/arch/x86_64/kernel/process.c?v=2.4.31#L728) |



