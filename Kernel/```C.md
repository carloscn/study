```C
#include <linux/kernel.h>
#include <linux/errno.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/miscdevice.h>
#include <asm/io.h>
#include <linux/interrupt.h>
#include <linux/irq.h>
#include <linux/gpio.h>
#include <mach/regs-gpio.h>
#include <asm-generic/siginfo.h>
#include <linux/init.h>
#include <asm/signal.h>
#include <linux/timer.h>
#include <asm/uaccess.h>
 
#define DEVICE_NAME "mybeep"
 
volatile unsigned long *GPBCON;
volatile unsigned long *GPBDAT;
volatile unsigned long *GPBUP;
void beep_start(void);
void beep_stop(void);
int  beep_irq_register(void);
unsigned int flag=1;
 
static struct fasync_struct *async; //声明fasync_struct
struct key_irq_desc {
	unsigned int irq;
	int pin;
	int pin_setting;
	int number;
	char *name;
};
 
static int beep_fasync(int fd, struct file *filp, int mode)
{
	printk("application  fasync!\n");
	return fasync_helper(fd, filp, mode, &async);         //注册上层调用进程的信息，上层调用fcntl设置FASYNC会调用这个系统调用
}
 
static struct key_irq_desc key_irqs [] = {
	{IRQ_EINT8, S3C2410_GPG(0), S3C2410_GPG0_EINT8, 0, "KEY1"},
};
 
static irqreturn_t key_interrupt(int irq, void *dev_id)
{
	kill_fasync(&async, SIGIO, POLL_IN);  //向打开设备文件的进程发出SIGIO信号
	return (IRQ_HANDLED);
}
 
void beep_gpiob_init(void)
{
	*GPBCON&=~((1<<0)|(1<<1));
	*GPBCON|=(1<<0);
	*GPBUP&=~(1<<0);
}
 
void beep_start(void)
{
	*GPBDAT|=(1<<0);
}
 
void beep_stop(void)
{
	*GPBDAT&=~(1<<0);
}
 
int beep_open(struct inode *inode, struct file *filp)
{
	if(beep_irq_register() != 0)
	{
		printk("Request irq error!\n");
	}
	printk(KERN_ALERT "application  open!\n");
	return 0;
}
 
ssize_t beep_read(struct file *file, char __user *buff, size_t count, loff_t *offp)
{
	printk("application  read!\n");
	return 0;
}
 
ssize_t beep_write(struct file *file, const char __user *buff, size_t count, loff_t *offp)
{
	printk("application  write!\n");
	return 0;
}
 
static int beep_release(struct inode *inode, struct file *file)
{
	disable_irq(key_irqs[0].irq);
	free_irq(key_irqs[0].irq, (void *)&key_irqs[0]);
	printk("application  close!\n");
	return beep_fasync(-1, file, 0);
}
 
static int beep_ioctl(struct inode *inode, struct file *file, unsigned int cmd, unsigned long arg)
{
	switch(cmd)
	{
	case 0:
		beep_start();
		break;
	case 1:
		beep_stop();
		break;
	default:
		break;
	}
	return 0;
}
 
static struct file_operations beep_ops = {
	.owner = THIS_MODULE,
	.open = beep_open,
	.release = beep_release,
	.ioctl = beep_ioctl,
	.read = beep_read,
	.write = beep_write,
	.fasync = beep_fasync,
};
 
static struct miscdevice beep_misc = {
	.minor = MISC_DYNAMIC_MINOR,
	.name = DEVICE_NAME,
	.fops = &beep_ops,
};
 
int beep_irq_register(void)
{
	int err;
	err = request_irq(key_irqs[0].irq, key_interrupt, 0, key_irqs[0].name, (void *)&key_irqs[0]);
	set_irq_type(key_irqs[0].irq, IRQ_TYPE_EDGE_RISING);
	if(err)
	{
		disable_irq(key_irqs[0].irq);
		free_irq(key_irqs[0].irq, (void *)&key_irqs[0]);
		return -EBUSY;
	}
	return 0;
}
 
static int __init beep_init(void)
{
	int ret;
	ret=misc_register(&beep_misc);
	if(ret <0)
	{
		printk("register miscdevice error code:%d\n",ret);
		return ret;
	}
	printk("beep device create!\n");
	GPBCON=(volatile unsigned long *)ioremap(0x56000010,12);
	GPBDAT=GPBCON+1;
	GPBUP=GPBCON+2;
	beep_gpiob_init();
	return 0;
}
 
static void __exit beep_exit(void)
{
	iounmap(GPBCON);
	misc_deregister(&beep_misc);
	printk("beep device delete!\n");
}
 
MODULE_LICENSE("GPL");
MODULE_AUTHOR("kingdragonfly");
module_init(beep_init);
module_exit(beep_exit);
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

