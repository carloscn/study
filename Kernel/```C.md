```C


#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int main(int argc, char *argv[])
{
    pid_t pid = fork();
    u_int16_t count = 0;

    while(1) {
        if (0 == pid) {
            debug_log("Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("I'm Child process\n");
            // just exit child process, and parent process don't call waitpid(),
            // which to create zombie process
            sleep(1);
            count ++;
            if (count > 10) {
                debug_log("child will exit.\n");
                exit(0);
            }
        } else if (pid > 0) {
            // debug_log("-------------------> wait for child exiting.\n");
            // WNOHANG is non block mode.
            // WUNTRACED is block mode.
            // mask the next line code, the zombie process will be generated.
            // pid_t t_pid = waitpid(pid, NULL, WUNTRACED);
            debug_log("-------------------> Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("-------------------> I'm Parent process!\n");
            sleep(2);
        } else {
            debug_log("fork() failed \n");
        }
    }

    return 0;
}

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


