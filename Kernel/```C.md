```C
/* 0 */		CALL(sys_restart_syscall)
		CALL(sys_exit)
		CALL(sys_fork)
		CALL(sys_read)
		CALL(sys_write)
/* 5 */		CALL(sys_open)
		CALL(sys_close)
		CALL(sys_ni_syscall)		/* was sys_waitpid */
		CALL(sys_creat)
		CALL(sys_link)
/* 10 */	CALL(sys_unlink)
		CALL(sys_execve)
		CALL(sys_chdir)
		CALL(OBSOLETE(sys_time))	/* used by libc4 */
		CALL(sys_mknod)
/* 15 */	CALL(sys_chmod)
		CALL(sys_lchown16)
		CALL(sys_ni_syscall)		/* was sys_break */
		CALL(sys_ni_syscall)		/* was sys_stat */
		CALL(sys_lseek)
/* 20 */	CALL(sys_getpid)
		CALL(sys_mount)
		CALL(OBSOLETE(sys_oldumount))	/* used by libc4 */
		CALL(sys_setuid16)
		CALL(sys_getuid16)
/* 25 */	CALL(OBSOLETE(sys_stime))
		CALL(sys_ptrace)
		CALL(OBSOLETE(sys_alarm))	/* used by libc4 */
		CALL(sys_ni_syscall)		/* was sys_fstat */
		CALL(sys_pause)
/* 30 */	CALL(OBSOLETE(sys_utime))	/* used by libc4 */
		CALL(sys_ni_syscall)		/* was sys_stty */
		CALL(sys_ni_syscall)		/* was sys_getty */
		CALL(sys_access)
		CALL(sys_nice)
/* 35 */	CALL(sys_ni_syscall)		/* was sys_ftime */
		CALL(sys_sync)
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
int usb_hub_init(void)
{
	if (usb_register(&hub_driver) < 0) {
		printk(KERN_ERR "%s: can't register hub driver\n",
			usbcore_name);
		return -1;
	}

	/*
	 * The workqueue needs to be freezable to avoid interfering with
	 * USB-PERSIST port handover. Otherwise it might see that a full-speed
	 * device was gone before the EHCI controller had handed its port
	 * over to the companion full-speed controller.
	 */
	hub_wq = alloc_workqueue("usb_hub_wq", WQ_FREEZABLE, 0);
	if (hub_wq)
		return 0;

	/* Fall through if kernel_thread failed */
	usb_deregister(&hub_driver);
	pr_err("%s: can't allocate workqueue for usb hub\n", usbcore_name);

	return -1;
}
```


