# 11_ARMv8_异常处理（二）- Legacy 中断处理

异常处理还没有完成，在异常处理（一）里面算是把ARMV8a上面的一些比较基本的异常相关的知识罗列出来了，但是还没有具体的分析。在ARM的世界，中断是异常的一种。这一节的目标是利用Linux内核研究：

* ARMv8架构下的中断处理过程？
* Linux内核在ARMv8a的架构下中断是如何处理？
* ARMv8a中断一些局限性，Linux内核的机制如何弥补？
* Cortex-A72多核多CPU的中断协调问题？
* Cortex-M33（armv8-m）与arm-v7a中断处理与armv8a的不同？
* freeRTOS事件处理机制在Cortex-M33上的处理？

ARMv8的中断有两种模式可以配置，一种是传统的中断处理Legacy模式，一种是使用GIC中断管理器模式。我们这次目标是Legacy模式，下一节异常处理（三）会学习GIC中断管理。

---

## 1 Overview interrupt

![image-20220416132633869](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220416132633869.png)

外部设备assert一个中断线，中断控制器会产生一个IRQ使CPU进入到异常处理流程，保存一些PC->ELR（子函数返回地址），保存PSTATE状态信息，还会切入EL1异常等级（栈指针），接着就是CPU和Kernel共同完成的中断向量表，根据中断向量表的地址进入到内核的异常handler，执行内核中断上下文。

### 1.1 Interrupt controller

#### 1.1.1 RTL design

我并非研究ARM的RTL设计的人员，但是从RTL里面能给我们提供一些本质的信息。我们这里用ZYNQ设计的双核Cortex-A9为例[^1]，看看整个中断控制器如何工作的。

![image-20220416134100235](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220416134100235.png)

中断控制器的RTL设计如图所示，我在图片上标注了中断控制器的输入输出，输入一共是两个，分别是图中的source 1和source 2，一个是外部的通过外设引脚输入进来的中断源，一个是内部的CPU产生的中断源，通过或的关系合并成FIQ和IRQ两个中断输出，中断的输出被接入到CPU0和CPU1的接口。我们在CPU上能观测到的就是异常的产生。

* CPU内部中断：Timer，AWDT
* CPU外部中断：SPI

**我们可以在PSTATE寄存器中控制IRQ和FIQ的开关。**

#### 1.1.2 Legacy interrupt controller

参考一些做CortexA72的二级产商的技术手册，NXP的imx8[^3]和德州仪器TDA4VM[^4]，并没有设计相应的legacy中断，我以树莓派的Cortex A72(x4)为例[^2]，了解一下legacy interrupt怎么设计的，我们在异常处理（三）中研究一下不同厂商如何将GIC集成到自己的ARM处理器上的。

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220416140153009.png" alt="image-20220416140153009" width="67%" />

树莓派是这样做的，将中断线分为几类，ARM coreN（arm核心组中断），ARM_LOCAL(只有CPU能访问的中断), ARMC（CPU和GPU共享的中断），videocore（GPU的中断）和ETH_PCIe（网络PCIE中断）。

![image-20220416145105813](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220416145105813.png)

中断比较多，传统的中断寄存器就是使用串联的方法，待定寄存器三个FIQn/IRQn_PENDING1 + FIQn/IRQn_PENDING0，还有一个PENDING2，串联起来，相当于把ARMC和外设类的中断打包，共用一个位，最后组装成ARM_LOCAL FIQ/IRQ_SOURCEn寄存器。所以我们使用这些寄存器来控制和读取中断状态。

在树莓派的手册里面举了个例子，一个UART4的FIQ中断发送到ARM core3上的处理时序：

* 进入到FIQ的handler
* 读FIQ_SOURCE3
* 判断FIQ_SOURCES3[8]为1， 就去读FIQ3_PENDING2
* 判断FIQ3_PENDING2[25]为1，就去读FIQ3_PENDING1
* 判定FIQ3_PENDING1[25]为1，就去读PACTL_CS[20:16]
* 判断PACTRL_CS[17]为1，就去读UART4_MIS断定什么造成的中断。

这部分应该是中断上半段处理的内容。

### 1.2 CPU work

中断也是一种异常和异常处理（一）的处理方式一样，摘录于异常处理（一）：

>**CPU自动做的事情：**
>
>* S1: PSTATE保存到SPSR_ELx
>* S2: 返回地址PC保存到ELR_ELx
>* S3: PSTATE寄存器里面的DAIF域都设定为1（等同于关闭调试异常、SError，IRQ FIQ中断）
>* S4: 更新ESR_ELx寄存器（包含了同步异常发生的原因）
>* S5: SP执行SP_ELx
>* S6: 切换到对应的EL，接着跳转到异常向量表执行
>
>**操作系统需要做的事情：**
>
>* 备份上面的寄存器到栈。
>
>* 识别异常发生的类型
>* 跳转到合适的异常向量表（包含异常跳转函数）
>* 处理异常
>* 操作系统执行eret指令

当中断发生的时候有两个现场需要保护：

* CPU需要保护中断现场，CPU在EL0异常等级，将中断现场都保存在**EL1**的栈里面。
* Kernel保护中断现场，发生在中断上半段。

这两个中断现场有个比较明显的分界线，就是是否跳转异常向量表。CPU的中断现场在跳转异常向量表之前，此时还没有进入到EL1，但是备份到EL1；Kernel的保护中断现场上半段，此时已经在CPU的EL1，栈已经切到了EL1的栈空间上面，保护程序逻辑的现场。

我们看一下CPU保护中断现场的工作，kernel保护中断现场放在1.4.1 top half来说。CPU发生异常的时候会自动把SP/PC/PSTATE这些数据备份到寄存器里面，但是这里有个问题，如果我们对这个数据不加以备份，当异常嵌套异常的时候，寄存器的值就会被冲走，我们只能恢复一级的异常，因此还要备份异常到相应的栈里面，当多级嵌套的异常从根节点一路路返回的时候，从栈中拿出寄存器的数据进行返回。这个好比QQ幻想里面的飞空艇仙子，龙城->长乐村->桃源村->天都，我们手里的信息只能回溯到上一站，等我们返回桃源村的时候，打开飞空艇仙子的对话框，就不知道去哪里了。因此我们每飞一个地方之后，需要在各个城市的飞空艇仙子的复印一个副本，告诉他我来自哪里，到天都之后我就知道回到桃源村，到桃源村的时候我就知道来自长乐村，最后我就能返回龙城。这就相当于中断现场的保护，在每一次中断过来之后，备份SP/PC/PSTATE到栈空间。

<img  src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220416160526722.png" alt="image-20220416160526722" width="70%"/>

在entry.S中kernel_entry就是CPU保护中断现场的工作，当然处理保护中断现场，还有很多关于内存的指令，这里面先不做过多的关注。

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220416161416901.png" alt="image-20220416161416901" width="70%" />

| REGS     | STACK          | Content |
| -------- | -------------- | ------- |
| SPSR_EL1 | pt.pstate      | PSTATE  |
| ELR_EL1  | pt.sp          | SP      |
| LR       | pt.regs[30]    | LR      |
| X29      | pt.regs[29]    | X29     |
| X28-X0   | pt.regs[28..0] | X28-X0  |

### 1.4 kernel:exception handler

这部分我们在Linux Kernel专题里面来讨论，在ARM64处理器这块就暂时不讨论了。

## 2 Example

对于Legacy中断模式，我们启动系统的定时器，在Cortex-A72处理器的树莓派4b上面实现一个定时器，需要自己完成寄存器的配置，异常向量表的跳转，寄存器的备份工作。

### 2.1 Timer Pre-knowledge

#### 2.1.2 regs

Cortex-A72一共是有4个定时器[^6]：

| 定时器    | ELx  | 安全模式   | 虚拟                   | 中断源      |
| --------- | ---- | ---------- | ---------------------- | ----------- |
| PS定时器  | EL1  | 安全模式   | 物理定时器             | CNT_PS_IRQ  |
| PNS定时器 | EL1  | 非安全模式 | 物理定时器             | CNT_PNS_IRQ |
| HP定时器  | EL2  | x          | 虚拟环境下的物理定时器 | CNT_HP_IRQ  |
| V定时器   | EL1  | x          | 虚拟定时器             | CNT_V_IRQ   |

相关寄存器也在这里[^7]：

| ame            | Type                                                         | Reset                                                        | Width  |                    Description                    |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------ | :-----------------------------------------------: |
| CNTKCTL_EL1    | RW                                                           | -[*a*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#fntarg_a) | 32-bit |           Timer Control register (EL1)            |
| CNTFRQ_EL0     | RW [*b*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#fntarg_b) | UNK                                                          | 32-bit |         Timer Counter Frequency register          |
| CNTPCT_EL0     | RO                                                           | UNK                                                          | 64-bit |           Physical Timer Count register           |
| CNTVCT_EL0     | RO                                                           | UNK                                                          | 64-bit |           Virtual Timer Count register            |
| CNTP_TVAL_EL0  | RW                                                           | UNK                                                          | 32-bit |          Physical Timer TimerValue (EL0)          |
| CNTP_CTL_EL0   | RW                                                           | -[*c*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#way1382454514990__CHDHDIJG) | 32-bit |       Physical Timer Control register (EL0)       |
| CNTP_CVAL_EL0  | RW                                                           | UNK                                                          | 64-bit |    Physical Timer CompareValue register (EL0)     |
| CNTV_TVAL_EL0  | RW                                                           | UNK                                                          | 32-bit |         Virtual Timer TimerValue register         |
| CNTV_CTL_EL0   | RW                                                           | -[*c*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#way1382454514990__CHDHDIJG) | 32-bit |          Virtual Timer Control register           |
| CNTV_CVAL_EL0  | RW                                                           | UNK                                                          | 64-bit |        Virtual Timer CompareValue register        |
| CNTVOFF_EL2    | RW                                                           | UNK                                                          | 64-bit |           Virtual Timer Offset register           |
| CNTHCTL_EL2    | RW                                                           | -[*d*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#fntarg_d) | 32-bit |           Timer Control register (EL2)            |
| CNTHP_TVAL_EL2 | RW                                                           | UNK                                                          | 32-bit |     Physical Timer TimerValue register (EL2)      |
| CNTHP_CTL_EL2  | RW                                                           | -[*c*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#way1382454514990__CHDHDIJG) | 32-bit |       Physical Timer Control register (EL2)       |
| CNTHP_CVAL_EL2 | RW                                                           | UNK                                                          | 64-bit |    Physical Timer CompareValue register (EL2)     |
| CNTPS_TVAL_EL1 | RW                                                           | UNK                                                          | 32-bit |     Physical Timer TimerValue register (EL2)      |
| CNTPS_CTL_EL1  | RW                                                           | -[*c*](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en#way1382454514990__CHDHDIJG) | 32-bit |   Physical Secure Timer Control register (EL1)    |
| CNTPS_CVAL_EL1 | RW                                                           | UNK                                                          | 64-bit | Physical Secure Timer CompareValue register (EL1) |

4个通用定时器总中断设置是在ARM_LOCAL的中断组的寄存器中配置。到此，其实我读到这里就有个疑问了，根据手册，CPU的4个通用定时器是在ARM_Core的中断组的，ARM_LOCAL里面只有一个local timer并不是这4个ARM的通用定时器，为什么要在ARM_LOCAL中断组寄存器里面配置。后面看了legacy中断的路由信息可以注意到：

![image-20220417123141473](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220417123141473.png)

虽然几个定时器被分配到了ARMCore组，但是实际上是在ARM_LOCAL里面的寄存器处理的，这个设计真的是有点令人很迷惑。

#### 2.1.2 config

定时器需要配置，根据常识都需要：

* 配置时间，计数多久？
* 中断子开关，中断总开关。
* 开始计时

##### ARM: CNTP_CTRL_EL0

我们选定的是CNT定时器，也就是EL1里面的PS定时器。在手册里面，CNTP_CTL_EL0来配置，这里还有个奇怪的事情，就是PS定时器是EL1里面的，但是配置寄存器是在CNTP_CTL_EL0内配置的（我搜了一下手册，是没有CNTP_CTL_EL1的，可能配置都是EL0寄存器里面完成的）

CNTP_CTL_EL0, counter_timer physical timer control register.

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220417132038101.png" alt="image-20220417132038101" width="67%" />

| bits       | function                                                     |
| ---------- | ------------------------------------------------------------ |
| 0: ENABLE  | Enables the timer. 0 关闭，1打开                             |
| 1: IMASK   | interrupt 掩码位， 0不会被iMASK bit掩码；1会被 iMASK bit掩码 |
| 2: ISTATUS | 定时器的状态，0定时器中断状态不满足，1定时器中断状态满足     |

```assembly
timer_ps0_enable:
    ldr x4, =TIMER_CNTRL0_REG_ADDR
    mov x5, #2
    str x5, [x4]
    ret
```

##### ARM: CNTP_TVAL_EL0

Timervalue的初始值，这个值会递减到0的时候会触发中断，还需要在handler里面重新赋值。

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220417134737306.png" alt="image-20220417134737306" width="67%" />

```assembly
#define ARM_LOCAL_REG_BASE_ADDR (0xFF800000)
#define TIMER_CNTRL0_REG_ADDR   (ARM_LOCAL_REG_BASE_ADDR + 0x40)

timer_ps0_enable:
    ldr x4, =TIMER_CNTRL0_REG_ADDR
    mov x5, #2
    str x5, [x4]
    ret
```

##### CortexA72: TIMER_CNTRLx

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220417135133905.png" alt="image-20220417135133905" width="67%" />

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220417135233516.png" alt="image-20220417135233516" width="67%" />

树莓派一共三个这样的寄存器，这个寄存器用于软件决定从ARM core接收一个FIQ的中断请求。我们关注的应该是BIT0，是cortex-A72处理器内核的PS定时器。

```assembly
timer_ps0_set_value:
    msr cntp_tval_el0, x0
    ret
```

##### ARM: PSTATE

配置PSTATE上面的DAIF使能总的中断开关。

```assembly
arch_enable_daif:
		msr	daifclr, #2
    ret

arch_disable_daif:
    msr	daifset, #2
    ret
```

#### 2.1.3 process

```c
// IRQ_SOURCE0_REG_ADDR
void irq_handle(void)
{
	unsigned int irq = 0;
	unsigned int regs = IRQ_SOURCE0_REG_ADDR;

	irq = readl(IRQ_SOURCE0_REG_ADDR);
	switch (irq) {
		case (CNT_PNS_IRQ):
		handle_timer_irq(100);
		break;

		default:
		printk("Unkown IRQ 0x%x\n", irq);
		break;
	}
}

void handle_timer_irq(unsigned int val)
{
	timer_ps0_set_value(val);
	printk("Core0 timer interrupt recved\n\r");
}
```

## 3 Reference

[^1]:[FPGA Technical Tutorials - Zynq System-on-Chip Design Overview Interrupts](https://www.fpgakey.com/tutorial/section428)
[^2]:[BCM2711 ARM Peripherals]()
[^3]:[i.MX 8QuadMax Applications Processor Reference Manual - **3.1.2 Interrupt Interface**](https://www.nxp.com/webapp/Download?colCode=IMX8QMRM)
[^4]:[ DRA829/TDA4VM Technical Reference Manual (Rev. C) - **9.1 Interrupt Architecture**](https://www.ti.com/lit/zip/spruil1)
[^5]:[10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47)
[^6]:[ARM Cortex-A72 MPCore Processor Technical Reference Manual r0p3 - Generic Timer functional description](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-functional-description?lang=en)
[^7]:[ARM Cortex-A72 MPCore Processor Technical Reference Manual r0p3 - AArch64 Generic Timer register summary](https://developer.arm.com/documentation/100095/0003/Generic-Timer/Generic-Timer-register-summary/AArch64-Generic-Timer-register-summary?lang=en)

