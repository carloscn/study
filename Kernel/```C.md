```C
#define init_waitqueue_head(wq_head)                            \
    do {                                                        \
        static struct lock_class_key __key;                     \
        __init_waitqueue_head((wq_head), #wq_head, &__key);     \
    } while (0)

void __init_waitqueue_head(struct wait_queue_head *wq_head, const char *name, struct lock_class_key *key)
{
    spin_lock_init(&wq_head->lock);
    lockdep_set_class_and_name(&wq_head->lock, key, name);
    INIT_LIST_HEAD(&wq_head->head);
}

#define DECLARE_WAIT_QUEUE_HEAD(name)                       \
    struct wait_queue_head name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {               \
    .lock       = __SPIN_LOCK_UNLOCKED(name.lock),          \
    .head       = { &(name).head, &(name).head } }

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
DELARE_WAIT_QUEUE_HEAD(queue);
DEFINE_WAIT(wait);

while (! condition) {
    prepare_to_wait(&queue, &wait, TASK_INTERRUPTIBLE);
    if (! condition)
        schedule();
    finish_wait(&queue, &wait)
}

```

上述所有形式函数中，`wq_head`是等待队列头（采用”值传递“的方式传输函数），`condition`是任意一个布尔表达式。使用`wait_event`，进程将被置于非中断休眠，而使用`wait_event_interruptible`时，进程可以被信号中断。
另外两个版本`wait_event_timeout`和`wait_event_interruptible_timeout`会使进程只等待限定的时间（以**jiffy**表示，给定时间到期时，宏均会返回0，而无论`condition`为何值）。