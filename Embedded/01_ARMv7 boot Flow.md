# 01_Embedded_ARMv7/v8 non-secure Boot Flow

从学习uboot开始，就误以为一个板子的开启，上电后的所有操作都是uboot来做的，那时候只关注于uboot能把内核引导成功；工作之后又接触到了secure boot，而在我们的secure boot设计中有好几级引导，我知道了boot并不是只限于uboot，我们可以根据工程的需要设计多级的boot；再当学到SoC上面的知识的时候，在boot阶段除了引导之外还有很多很多工作要做。因此，我决定把boot这块所有的相关的内容整理出来，顺便把secure boot的一些做法写出来，看看在boot哪个阶段，我们可以用secure boot。

# 0. secure and non-secure booting

我们在[^9]中介绍了ARMv8架构下面ATF启动的的过程。实际上ATF是arm为了增强系统安全性引入的一种可信性固件，我们要使用ARMv7/v8的secure安全feature，就必须要使用ATF作为先前引导，再由ATF引导uboot的启动。而uboot嵌入式系统的通用引导程序，其历史比ATF更加久远，而且可以支持平台和架构更多。在ARM架构上，**uboot在默认情况下并不需要与ATF共同启动**，而且uboot的自身设计本身就支持完整的多级启动链（包含SPL、TPL和uboot三个阶段），但无法使能ARM的secure执行状态。因此，**uboot可以在不使用安全feature的情况下独立完成ARMv7/v8的启动引导工作**。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraembedded_01_non_secure_boot_flow.drawio.svg" width="100%" />
</div>

non-secure的流程完整流程如图所示，主要被分为4个部分，第一个bootrom（or XIP）阶段，启动链一般由bootrom中的SoC Bootloader开始，接着加载SPL作为第二级启动镜像BL2，主要作用完成一些基础模块和DDR的初始化。TPL是uboot引申出来的一级引导，由于SPL是要被放到SoC上面的SRAM执行的，但是由于SRAM空间有限，故将SPL内部的功能进行拆分完成，这个在uboot文档中介绍主要给PowerPC这种空间不足的架构平台使用的，而ARM平台罕见定义TPL[^10]。

在ARM平台的流程如图所示：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraembedded_01_non_secure_boot_flow_arm.drawio.svg" width="100%" />
</div>

若不需要支持tpl，则uboot的典型启动流程可精简为如下方式Option A（这也是uboot最常见的运行方式）。当然，对于有些启动速度要求较高的场景，还可以进一步简化其启动流程。如可将其设计为下面这种跳过uboot，直接通过SPL启动操作系统的方式，此时其启动流程如Option B。

对于ATF和uboot组合方式启动的方法和流程请参考[^9]。本文的核心点默认为booting with non-secure的流程。

# 1. boot high-level

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220520180903952.png" width="70%" />
</div>

boot阶段整体的overview可以如上图所示，我们把boot阶段分为两大部分：Bootloader Phase和Kernel Phase (or Little Kernel Phase)。在Bootloader阶段我们又分为三个过程，

*   **SoC ROM bootloader**[^6]。当一个处理器上电之后，在SoC ROM bootloader中开始执行所谓的上电第一段代码，主要工作就是对CPU进行一系列的初始化，包括：**CPU secure boot 初始化，栈的建立，MPU，还有各个子模块的锁相环时钟的初始化，并且读取pin引脚决定从哪个地方启动**，除此之外还要根据硬件引脚从哪里启动的配置来加载第一阶段的**bootloader到SoC internal RAM**，这里需要注意的是，**SoC ROM bootloader的自己的引导程序也是从BootROM中加载到内部的SoC级internal RAM中执行的**，最后SoC ROM bootloader跳转到First stage bootloader，把控制权交由First stage bootloader。
*   **First stage bootloader**[^6][^7]，也被称为**SPL（Secondary Program Loader）**，因为这部分的工作**主要负责把Second Stage Bootloader复制到external RAM中**，并且跳转到Second Stage Bootloader上，所以也称为**MLO（Memory Loader）**。在ARMv8-A中启动secure boot，SPL负责加载arm-tf或者op-tee的启动[^4]。
*   **Second Stage Bootloader**的工作就比较多了，而且由于在外部的RAM上有足够大的空间，这部分甚至都可以支持文件系统来完成配置，查找Linux内核和设备树，最终完成引导（**通常uboot就在这一层次**）。在某些情况下，second stage bootloader引导嵌套引导使用，例如在ARMv7架构中，SPL加载完u-boot之后，可以选择使用u-boot再去boot op-tee[^4]。

# 2. boot low-level

我们以TI的AM335x microprocessor（Cortex - A8）为例子，其他的嵌入式设备大同小异。注意，我们讲述的角度是以SoC的角度来说明的，因此这个层级和boot开发者不一致，我们比boot开发者要多考虑一级，比如SoC的初始化，正常这部分是有芯片厂家编写，boot开发者并不需要关心这部分，但如果我们的产业层级就是芯片厂家，那么这部分还是需要关注的。如果你和boot开发者交流技术，一定要注意他说的第一级的引导可能是你说的第二级引导。

## 2.1 SoC ROM Bootloader

### 2.1.1 BootROM的硬件支持

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220520125333265.png" width="60%" />
</div>
我们来看一下AM335x的结构[^1]，我们在bootloader比较关心的就是176KB的ROM（里面包含了分配给我们的BootROM），还有64KB的CPU载RAM，及SoC内部的shared RAM。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220520125954948.png" width="80%" />
</div>

我们在[^2]的117页找到关于芯片的Memory Map的章节，可以看到**BootROM分配大小是128KB + 48KB刚好等于176KB上图的大小**。这个BootROM存储的就是引导程序。对于这部分ROM的分配，**128KB是给secure booting的，而48KB是public boot ROM**。如图，所示[^2]5022页。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220520140134429.png" width="60%" />
</div>

### 2.1.2 BootROM里面都有什么？

#### Public BootROM

我们来看一下TI的设计，图片展示的是在**48KB的public bootrom**中的软件架构图，我们可以看出Public BootROM是一个比较大和丰富的程序，并不是想象中一个精简的程序。High-level主要就是封装了clock、booting，还有文件系统等等等等接口，driver上面就是UART、USB、SPI这些外设的支持；TI也采用HAL层来隔离硬件和软件。**这部分代码是由TI来写的，在制作SoC之前，就会被固化到芯片中，后续量产并无法修改。**这里还需要注意SEC_ENTRY，在ARM架构上面根据trustzone技术的要求，BootROM的执行模式需要在EL3特权等级。从TI的文档中可以猜测，这部分SEC_ENTRY是一个开启异常的操作，需要在异常处理机制中来处理secure boot。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220521091704643.png" width="67%" />
</div>

#### Secure BootROM

由于MPU（memory protection unit）[^3]的要求，所以**SoC在启动的时候总是被要求以安全模式启动**。Secure ROM 代码是在reset handler异常处理机制中实现的（trustzone要求），当通过了Secure ROM的代码之后，才会从0x20000地址启动public ROM boot程序。可以如下图展示SoC ROM阶段的实际上也分为两种，一个secure SoC ROM阶段，还有一个是Public SoC ROM阶段。

### 2.1.3 BootROM初始化

在TI-AM335X的参考设计里面CPU上电最开始执行128KB的，secure BootROM中的secure boot程序，验证通过之后跳入自己定义的。**TI在这里并没有给出secureboot如何设计的**。public bootrom的程序对系统进行初始化、建立栈空间、看门狗、配置模块的时钟、接着开始。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220520193936469.png" width="40%" />
</div>

在Secure SoC ROM阶段，Secure ROM中的Secure Boot程序会被加载到SoC内的secure RAM中执行。（通常Secure Boot程序包含Manifest/Pubkey HASH/加密后的images），而且在ARMv8组织架构中，secure boot要求，“secure ROM” + "Root of Trust in OTP" + "ATF"组成一个完整的授权链。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220520174307914.png" width="67%" />
</div>

我们再参考一下IMX.8[^8]130页。关于secure boot这块，在SoC等级每家芯片厂商并不是一样的。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220521092927170.png" width="67%" />
</div>

### 2.1.4 Booting过程

TI的设计是采用遍历模式，基于用户的配置或者SYSBOOT的引脚，列出所有设备，进行遍历初始化，如果发现是memory booting就进入到memory booting的流程，发现是periphera booting，就进入到相应的流程。memory设备是NOR、NAND、MMC或者SPI-EEPROM；而peripheral设备是Ethernet、usb、UART等。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220521094714829.png" width="67%" />
</div>

#### memory booting

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220521100121644.png" width="67%" />
</div>

memory boot之后就开始进入到image的读取的过程。注意这里有个XIP的识别，XIP是一种不需要拷贝到RAM中，直接在XIP上运行程序的一种东西。

#### peripheral booting

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220521100338452.png" width="67%" />
</div>

### 2.1.5 总结

**我们来总结一下关于SoC boot ROM：**

*   **初始化SoC时钟、电源、还有外设。**
*   **选择引导媒介（U盘、TF卡、或者FLASH等）**
*   **从引导媒介中读取SPL级引导程序，并且把这个部分程序加载到SRAM中。**
*   **如果使能了Secure Boot，会验证SPL的签名和解密SPL程序**
*   **跳转SPL**

## 2.2 First Stage Bootloader （SPL/MLO）

通常状况的SPL是一个非常小的程序（64KB）以下，运行在SoC内部的SRAM上面。SPL通常存储在启动媒介里面（XIP或者BootROM），**因此这部分程序并不是由soc厂家负责开发的**。放在启动媒介的原因是，SPI-FLASH，NAND FLASH这些外部存储器，都需要相应的程序驱动，在boot阶段是没有这些驱动的，因此cpu执行的第一段启动程序都是SoC厂家固化到启动媒介中的。在SPL阶段现代处理器可以选择性的选择引导媒介如U盘，TF卡，相应的在SPL阶段还需要load这些存储驱动，并且需要把存储媒介中的引导程序加载到SPL中。

补充一句，在嵌入式板子上面经常有拨码开关用于选择从哪个地方引导SD卡，从NANDFLASH等，这部分代码实际上是在BootROM中控制，换句话说，BootROM也需要支持这些驱动，而需要从存储媒介中加载的则是SPL和UBOOT两个阶段的程序。

从uboot工程上来看，SPL复用的是uboot里面的代码[^12]，通过`CONFIG_SPL_BUILD`宏定义来隔离、复用uboot的代码，因此可以见得，SPL和uboot本身功能有一些重复的。SPL的编译是uboot的一部分，和uboot.bin是两条编译流程，正常来说，会编译主题的uboot.bin，然后在编译uboot-spl，也就是uboot-spl.bin[^13]（编译过程也可以参考文献[^13]）。

本章主要参考两个文献，文献A《[uboot] （第三章）uboot流程——uboot-spl代码流程》[^14]基于ARMv7架构的分析，以及文献B《聊聊SOC启动（七） SPL启动分析》[^11]基于ARMv8架构的分析。

### 2.2.1 ARMv7 uboot-spl analysising

**对于ARMv7在uboot-spl上面主要做的事情**[^14]：
* 关闭中断，进入SVC模式
* 协处理的初始化CP15
* SoC级、板级的初始化操作：
	* IO初始化
	* 时钟初始化
	* 内存初始化
	* 串口/nandflash
* 加载BL2
* 跳转到BL2（uboot）

我们可以注意一下ARMv7的整体spl流程如图所示：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraarmv7_spl1.svg" width="100%" />
</div>

#### 2.2.1.1  `_start`
armv7架构的入口在：（https://github.com/ARM-software/u-boot/blob/master/arch/arm/cpu/u-boot-spl.lds）

`ENTRY(_start)`

```assembly
_start:
#ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
    .word   CONFIG_SYS_DV_NOR_BOOT_CFG
#endif
    b   reset
```

#### 2.2.1.2  reset
https://github.com/ARM-software/u-boot/blob/master/arch/arm/cpu/armv7/start.S#L38 :

```assembly
/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 * Do important init only if we don't start from memory!
 * Setup memory and board specific bits prior to relocation.
 * Relocate armboot to ram. Setup stack.
 *
 *************************************************************************/

reset:
	/* Allow the board to save important registers */
	b	save_boot_params
save_boot_params_ret:
#ifdef CONFIG_ARMV7_LPAE
/*
 * check for Hypervisor support
 */
	mrc	p15, 0, r0, c0, c1, 1		@ read ID_PFR1
	and	r0, r0, #CPUID_ARM_VIRT_MASK	@ mask virtualization bits
	cmp	r0, #(1 << CPUID_ARM_VIRT_SHIFT)
	beq	switch_to_hypervisor
switch_to_hypervisor_ret:
#endif
	/*
	 * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
	 * except if in HYP mode already
	 */
	mrs	r0, cpsr
	and	r1, r0, #0x1f		@ mask mode bits
	teq	r1, #0x1a		@ test for HYP mode
	bicne	r0, r0, #0x1f		@ clear all mode bits
	orrne	r0, r0, #0x13		@ set SVC mode
	orr	r0, r0, #0xc0		@ disable FIQ and IRQ
	msr	cpsr,r0

/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
	/* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
	mrc	p15, 0, r0, c1, c0, 0	@ Read CP15 SCTLR Register
	bic	r0, #CR_V		@ V = 0
	mcr	p15, 0, r0, c1, c0, 0	@ Write CP15 SCTLR Register

#ifdef CONFIG_HAS_VBAR
	/* Set vector address in CP15 VBAR register */
	ldr	r0, =_start
	mcr	p15, 0, r0, c12, c0, 0	@Set VBAR
#endif
#endif

	/* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
#ifdef CONFIG_CPU_V7A
	bl	cpu_init_cp15
#endif
#ifndef CONFIG_SKIP_LOWLEVEL_INIT_ONLY
	bl	cpu_init_crit
#endif
#endif

	bl	_main
```

#### 2.2.1.3 `cpu_init_cp15`
cpu_init_cp15，cpu_init_cp15主要用于对cp15协处理器进行初始化，其主要目的就是关闭其MMU和TLB，如下: https://github.com/ARM-software/u-boot/blob/master/arch/arm/cpu/armv7/start.S#L139

该过程包含：
* 使L1的caches失效，并且使用内存屏障等待完成
* 关闭MMU和Caches
* 一些ARM 的errata的处理

```assembly
ENTRY(cpu_init_cp15)
	/*
	 * Invalidate L1 I/D
	 */
	mov	r0, #0			@ set up for MCR
	mcr	p15, 0, r0, c8, c7, 0	@ invalidate TLBs
	mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache
	mcr	p15, 0, r0, c7, c5, 6	@ invalidate BP array
	mcr     p15, 0, r0, c7, c10, 4	@ DSB
	mcr     p15, 0, r0, c7, c5, 4	@ ISB

	/*
	 * disable MMU stuff and caches
	 */
	mrc	p15, 0, r0, c1, c0, 0
	bic	r0, r0, #0x00002000	@ clear bits 13 (--V-)
	bic	r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)
	orr	r0, r0, #0x00000002	@ set bit 1 (--A-) Align
	orr	r0, r0, #0x00000800	@ set bit 11 (Z---) BTB
#ifdef CONFIG_SYS_ICACHE_OFF
	bic	r0, r0, #0x00001000	@ clear bit 12 (I) I-cache
#else
	orr	r0, r0, #0x00001000	@ set bit 12 (I) I-cache
#endif
	mcr	p15, 0, r0, c1, c0, 0

#ifdef CONFIG_ARM_ERRATA_716044
	mrc	p15, 0, r0, c1, c0, 0	@ read system control register
	orr	r0, r0, #1 << 11	@ set bit #11
	mcr	p15, 0, r0, c1, c0, 0	@ write system control register
#endif
    
	@.....

	mov	pc, r5			@ back to my caller
ENDPROC(cpu_init_cp15)

```

#### 2.2.1.4 `cpu_init_crit`
cpu_init_crit，进行一些关键的初始化动作，也就是平台级和板级的初始化。其代码核心就是lowlevel_init，如下：https://github.com/ARM-software/u-boot/blob/master/arch/arm/cpu/armv7/start.S#L324

```assembly
ENTRY(cpu_init_crit)
	/*
	 * Jump to board specific initialization...
	 * The Mask ROM will have already initialized
	 * basic memory. Go here to bump up clock rate and handle
	 * wake up conditions.
	 */
	b	lowlevel_init		@ go setup pll,mux,memory
ENDPROC(cpu_init_crit)
```

lowlevel_init一般是由板级代码自己实现的。但是对于某些平台来说，也可以使用通用的lowlevel_init，其定义在[arch/arm/cpu/lowlevel_init.S](https://github.com/ARM-software/u-boot/blob/master/arch/arm/cpu/armv7/lowlevel_init.S)中，以tiny210为例，在移植tiny210的过程中，就需要在board/samsung/tiny210下，也就是板级目录下面创建lowlevel_init.S，在内部实现lowlevel_init。（其实只要实现了lowlevel_init了就好，没必要说在哪里是实现，但是通常规范都是创建了lowlevel_init.S来专门实现lowlevel_init函数）。

arm在默认实现中，使用`WEAK(lowlevel_init)`弱符号进行定义，用法为若其它的地方定义了同名的函数或全局变量，则会使用重定义的值，否则就使用WEAK标号中的定义。实际上它是一个很有用的特性，如我们可以为某个函数定义一个默认的定义，并将其用WEAK关键字修饰，当调用该函数的用户希望其使用自己定义的特殊实现时，就可以在其它的文件中重新定义一个非WEAK的同名函数，此时链接器链接时就会链接新的定义，而自动忽略掉用WEAK修饰的定义，从而可以实现函数功能的扩展，或者用于一些debug操作等。可以参考C实验[^15]。

类似于三星平台，会在自己board目录下面做自己的初始化操作：

https://github.com/ARM-software/u-boot/blob/402465214395ed26d6fa72d9b6097c7adbf6a966/board/samsung/smdkc100/lowlevel_init.S

包含了一些工作：
* 检查一些复位状态  
* 关闭看门狗  
* 系统时钟的初始化  
* 内存、DDR的初始化  
* 串口初始化（可选）  
* Nand flash的初始化

#### 2.2.1.5 `_main`

spl的main的主要目标是调用board_init_f进行先前的板级初始化动作，在tiny210中，主要设计为，加载BL2到DDR上并且跳转到BL2中。DDR在上述lowlevel_init中已经初始化好了。**由于board_init_f是以C语言的方式实现，所以需要先构造C语言环境**。注意：**uboot-spl和uboot的代码是通用的，其区别就是通过CONFIG_SPL_BUILD宏来进行区分的。**  所以以下代码中，我们只列出spl相关的部分，也就是被CONFIG_SPL_BUILD包含的部分。

https://github.com/ARM-software/u-boot/blob/v2018.09/arch/arm/lib/crt0.S#L66

`_main`处理包含：
* C语言环境，首先设置堆栈 (only intermediate)
* 设定中间环境 (SP和global data)
* 重新分配向量表
* 清理.bss段
* LED灯显示
* 调用`board_init_f`

```assembly
ENTRY(_main)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */

#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
	ldr	r0, =(CONFIG_SPL_STACK)
#else
	ldr	r0, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
	bic	r0, r0, #7	/* 8-byte alignment for ABI compliance */
	mov	sp, r0
	bl	board_init_f_alloc_reserve
	mov	sp, r0
	/* set up gd here, outside any C code */
	mov	r9, r0
	bl	board_init_f_init_reserve

	mov	r0, #0
	bl	board_init_f

#if ! defined(CONFIG_SPL_BUILD)

/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we'll return
 * 'here' but relocated.
 */

	ldr	r0, [r9, #GD_START_ADDR_SP]	/* sp = gd->start_addr_sp */
	bic	r0, r0, #7	/* 8-byte alignment for ABI compliance */
	mov	sp, r0
	ldr	r9, [r9, #GD_BD]		/* r9 = gd->bd */
	sub	r9, r9, #GD_SIZE		/* new GD is below bd */

	adr	lr, here
	ldr	r0, [r9, #GD_RELOC_OFF]		/* r0 = gd->reloc_off */
	add	lr, lr, r0
#if defined(CONFIG_CPU_V7M)
	orr	lr, #1				/* As required by Thumb-only */
#endif
	ldr	r0, [r9, #GD_RELOCADDR]		/* r0 = gd->relocaddr */
	b	relocate_code
here:
/*
 * now relocate vectors
 */

	bl	relocate_vectors

/* Set up final (full) environment */

	bl	c_runtime_cpu_setup	/* we still call old routine here */
#endif
#if !defined(CONFIG_SPL_BUILD) || defined(CONFIG_SPL_FRAMEWORK)
# ifdef CONFIG_SPL_BUILD
	/* Use a DRAM stack for the rest of SPL, if requested */
	bl	spl_relocate_stack_gd
	cmp	r0, #0
	movne	sp, r0
	movne	r9, r0
# endif
	ldr	r0, =__bss_start	/* this is auto-relocated! */

#ifdef CONFIG_USE_ARCH_MEMSET
	ldr	r3, =__bss_end		/* this is auto-relocated! */
	mov	r1, #0x00000000		/* prepare zero to clear BSS */

	subs	r2, r3, r0		/* r2 = memset len */
	bl	memset
#else
	ldr	r1, =__bss_end		/* this is auto-relocated! */
	mov	r2, #0x00000000		/* prepare zero to clear BSS */

clbss_l:cmp	r0, r1			/* while not at end of BSS */
#if defined(CONFIG_CPU_V7M)
	itt	lo
#endif
	strlo	r2, [r0]		/* clear 32-bit BSS word */
	addlo	r0, r0, #4		/* move to next */
	blo	clbss_l
#endif

#if ! defined(CONFIG_SPL_BUILD)
	bl coloured_LED_init
	bl red_led_on
#endif
	/* call board_init_r(gd_t *id, ulong dest_addr) */
	mov     r0, r9                  /* gd_t */
	ldr	r1, [r9, #GD_RELOCADDR]	/* dest_addr */
	/* call board_init_r */
#if CONFIG_IS_ENABLED(SYS_THUMB_BUILD)
	ldr	lr, =board_init_r	/* this is auto-relocated! */
	bx	lr
#else
	ldr	pc, =board_init_r	/* this is auto-relocated! */
#endif
	/* we should not return here. */
#endif

ENDPROC(_main)

```

设定C语言环境阶段，使用了2个函数，分别是：
* board_init_f_alloc_reserve
* board_init_f_init_reserve

代码在这个位置，是一个C语言的函数：

https://github.com/ARM-software/u-boot/blob/v2018.09/common/init/board_init.c

顾名思义，这两个函数都是为了board_init_f做一些储备性质的工作准备。从“top”地址分配保留空间用作“globals”，并返回所分配空间的“bottom”地址。

这里就不得不提一下关于global data structure了，简称GD，可以简单理解为uboot的全局变量都要放在这里。关于GD的结构体定义如下[^16]：

https://github.com/ARM-software/u-boot/blob/v2018.09/include/asm-generic/global_data.h

>Globally required fields are held in the global data structure. A pointer to the structure is available as symbol gd. The symbol is made available by the macro %DECLARE_GLOBAL_DATA_PTR.

```C
ulong board_init_f_alloc_reserve(ulong top)
{
    /* Reserve early malloc arena */
    /* LAST : reserve GD (rounded up to a multiple of 16 bytes) */
    top = rounddown(top-sizeof(struct global_data), 16);
	// 现将top（也就是r0寄存器，前面说过存放了暂时的指针地址），减去sizeof(struct global_data)，也就是预留出一部分空间给sizeof(struct global_data)使用。
	// rounddown表示向下16个字节对其

    return top;
	// 到这里，top就存放了GD的地址，也是SP的地址
	// 把top返回，注意，返回后，其实还是存放在了r0寄存器中。
}

void board_init_f_init_reserve(ulong base)
{
    struct global_data *gd_ptr;
    int *ptr;
    /*
     * clear GD entirely and set it up.
     * Use gd_ptr, as gd may not be properly set yet.
     */

    gd_ptr = (struct global_data *)base;
	// 从r0获取GD的地址
    /* zero the area */
    for (ptr = (int *)gd_ptr; ptr < (int *)(gd_ptr + 1); )
        *ptr++ = 0;
	// 对GD的空间进行清零
}
```
其实GD在spl中没什么使用，主要是用在uboot中，但在uboot中的时候还需要另外分配空间。

#### 2.2.1.6 Jumping to BL2

在SPL处理流程的最后，需要跳转到板级前期的初始化函数中，如下代码：

`bl board_init_f`

board_init_f需要由板级代码自己实现。 在这个函数中，tiny210主要是实现了从SD卡上加载了BL2到ddr上，然后跳转到BL2的相应位置上 tiny210的实现如下: [arch/arm/mach-rockchip/rk3036-board-spl.c](https://github.com/ARM-software/u-boot/blob/v2018.09/arch/arm/mach-rockchip/rk3036-board-spl.c) 这个是ARM的仓库里面的uboot，在一些SoC厂商订制的uboot中，会在board目录下面增加自己的board_init_f的实现，例如，三星tiny210主要是实现了SD卡上加载BL2到ddr上，然后跳转到BL2的相应位置上的操作。

```C
#ifdef CONFIG_SPL_BUILD
void board_init_f(ulong bootflag)
{
    __attribute__((noreturn)) void (*uboot)(void);
    int val;
#define DDR_TEST_ADDR 0x30000000
#define DDR_TEST_CODE 0xaa
    tiny210_early_debug(0x1);
    writel(DDR_TEST_CODE, DDR_TEST_ADDR);
    val = readl(DDR_TEST_ADDR);
    if(val == DDR_TEST_CODE)
        tiny210_early_debug(0x3);
    else
    {
        tiny210_early_debug(0x2);
        while(1);
    }
	// 先测试DDR是否完成
    copy_bl2_to_ddr();
	// 加载BL2的代码到ddr上
    uboot = (void *)CONFIG_SYS_TEXT_BASE;
	// uboot函数设置为BL2的加载地址上
    (*uboot)();
	// 调用uboot函数，也就跳转到BL2的代码中
}
#endif
```

关于copy_bl2_to_ddr的实现，也就是如何从SD卡或者nand flash上加载BL2到DDR上的问题，到此，SPL的任务就完成了，也已经跳到了BL2也就是uboot里面去了。

### 2.2.2 ARMv8 uboot-spl analysising

**对于ARMv8在uboot-spl上面主要做的事情**[^11]：
* 设置CPU的状态，如cache、MMU、大小端设定
* 准备C语言环境，包括设定栈指针、清空BSS段
* 为GD分配空间
* 初始化DRAM，并将代码拷贝到DRAM中执行。
* 加载BL2
* 跳转到BL2（uboot）

实际上ARMv7和ARMv8整体的需要做的事情基本相同，除了一些SoC级别的差异外。

我们可以注意一下ARMv8的整体spl流程如图所示：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraarmv8.svg" width="80%" />
</div>

#### 2.2.2.1 ARMv8 uboot-spl entrypoint
armv8架构的入口在：（https://github.com/ARM-software/u-boot/blob/master/arch/arm/cpu/armv8/u-boot-spl.lds#L21）

`ENTRY(_start)`

armv8架构下的SPL入口函数位于`arch/arm/cpu/armv8/start.S`文件的_start，它的定义如下：https://github.com/ARM-software/u-boot/blob/v2018.09/arch/arm/cpu/armv8/start.S#L19

```assembly
.globl  _start
_start:
#ifdef CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK
/*
 * Various SoCs need something special and SoC-specific up front in
 * order to boot, allow them to set that in their boot0.h file and then
 * use it here.
 */
#include <asm/arch/boot0.h>
#else
    b   reset
#endif
```

它有两种情况，一种是某些平台会定义自己特殊的启动代码，此处我们看通用的情况，即else的分支中，它直接跳转到了reset处。它的定义如下：

```assembly
reset:
    /* Allow the board to save important registers */
    b   save_boot_params
.globl  save_boot_params_ret
save_boot_params_ret:

#ifdef CONFIG_SYS_RESET_SCTRL
    # 操作sctrl的值，以配置相关设置
    bl reset_sctrl
#endif
```

此处也是一处跳转指令，它会跳转到save_boot_params处，它的定义如下：

```assembly
WEAK(save_boot_params)
    b   save_boot_params_ret    /* back to my caller */
ENDPROC(save_boot_params)
```

#### 2.2.2.2 switch_el and set register

其后根据是否配置了CONFIG_SYS_RESET_SCTRL参数决定是否执行reset_sctrl的内容。我们看下它的实现如下：

```assembly
#ifdef CONFIG_SYS_RESET_SCTRL
reset_sctrl:
    switch_el x1, 3f, 2f, 1f
3:
    mrs x0, sctlr_el3
    b   0f
2:
    mrs x0, sctlr_el2
    b   0f
1:
    mrs x0, sctlr_el1
0:
    ldr x1, =0xfdfffffa
    and x0, x0, x1

    switch_el x1, 6f, 5f, 4f
6:
    msr sctlr_el3, x0
    b   7f
5:
    msr sctlr_el2, x0
    b   7f
4:
    msr sctlr_el1, x0
7:
    dsb sy
    isb
    b   __asm_invalidate_tlb_all
    ret
#endif
```

它首先调用switch_el 函数，该函数的定义位于[arch/arm/include/asm/macro.h](https://github.com/ARM-software/u-boot/blob/v2018.09/arch/arm/include/asm/macro.h#L70)，我们先看下它的功能。mrs是arm读取系统寄存器内容的指令，此处它会读取CurrentEL寄存器的值，该寄存器存放了cpu当前所处的异常等级。它将读到的值与0xc比较，该比较指令会根据比较结果设置NZCV标志位。若他们的值相等，则会设置Z标志位。根据Z标志位判断寄存器的值是否等于0xc，若相等则跳转到el3_label，即第二个参数处，否则继续比较，根据相应的值跳转到不同分支。

我们再回到reset_sctrl的内容，它含有0 - 7一共8个标号，为了描述方便，后面涉及到EL的分支时，我们都以EL1为例描述。在标号1处会将stlr_el1的内容读到x0寄存器，然后将立即数0xfdfffffa加载到x1寄存器，并将x0和x1执行位与操作，即它会清除sctlr_el1的bit0，bit2和bit24。sctlr_el1及各bit的定义如下图，从中可以看到bit0用于关MMU，bit2用于关cache，bit24用于选择大小端。接下来的switch_el继续根据当前异常等级选择不同的分支，在EL1时会执行标号4，该操作即是将修改好的值写回到sctlr_el1寄存器中。

后面是两个内存屏障的操作，内存屏障主要用于同步内存的访问顺序，其中dsb是数据内存屏障，isb是指令内存屏障。接下来将执行__asm_invalidate_tlb_all，它定义在arch/arm/cpu/armv8/tlb.S中，代码如下：

```assembly
ENTRY(__asm_invalidate_tlb_all)
    switch_el x9, 3f, 2f, 1f
3:  tlbi    alle3
    dsb sy
    isb
    b   0f
2:  tlbi    alle2
    dsb sy
    isb
    b   0f
1:  tlbi    vmalle1
    dsb sy
    isb
0:
    ret
ENDPROC(__asm_invalidate_tlb_all)
```

首先根据当前的el等级跳转到不同的标号，我们还是看EL1的情况，它执行了一条tlbi指令，用于失效tlb中的内容，然后执行了两条内存屏障操作并返回。tlb是物理地址和虚拟地址转换表的高速缓存，因为页表是存放在内存中的，若没有tlb则每次虚拟地址到物理地址的转换都需要通过访问内存来获取转换信息，显然这个速度是非常缓慢的，因此在内存和cpu之间添加了一个tlb缓存，用于存储最近的一些内存转换信息，以加速对虚拟地址的操作。与cache的情况类似，tlb的内容也可能和实际的页表出现不一致，如在页表建立之前，tlb中的内容其实都是无效数据，还有在进程上下文切换时，由于每个进程的页表是独立的，因此tlb中的内容也将会不一致，因此，在这些操作中都需要将老的tlb内容失效掉以防出现数据不一致的问题。

#### 2.2.2.3 set vector
代码返回到reset_sctrl之后的位置，接下来会设置异常向量表，并disable trap的功能，代码如下：

```assembly
    adr x0, vectors
    switch_el x1, 3f, 2f, 1f
3:  msr vbar_el3, x0
    mrs x0, scr_el3
    orr x0, x0, #0xf            /* SCR_EL3.NS|IRQ|FIQ|EA */
    msr scr_el3, x0
    msr cptr_el3, xzr           /* Enable FP/SIMD */
#ifdef COUNTER_FREQUENCY
    ldr x0, =COUNTER_FREQUENCY
    msr cntfrq_el0, x0          /* Initialize CNTFRQ */
#endif
    b   0f
2:  msr vbar_el2, x0
    mov x0, #0x33ff
    msr cptr_el2, x0            /* Enable FP/SIMD */
    b   0f
1:  msr vbar_el1, x0
    mov x0, #3 << 20
    msr cpacr_el1, x0           /* Enable FP/SIMD */
0:
```

首先将vectors变量的值加载到x0寄存器中，vectors定义在arch/arm/cpu/armv8/exceptions.S中，代码如下，即其定义了cpu的异常向量表。对于arm处理器，在发生异常时就会跳转到预先定义好的异常向量表处执行，比如若发生了外部中断，中断控制器GICvx会设置irq中断线引起cpu的irq异常，此时cpu就会跳转到异常向量表中irq相关项的偏移处执行该条指令，如此处的`b _do_bad_irq`。       

cpu是如何知道自己将要跳转到哪里的呢？这就是接下来代码所做的工作了。我们回到上面的代码中，当异常向量表的首地址vectors被加载到x0寄存器之后，就根据当前的异常等级跳转到相应标号处执行，在EL1时会将x0的值写入系统寄存器vbar_el1中，

```assembly
    .align  11
    .globl  vectors
vectors:
    .align  7
    b   _do_bad_sync    /* Current EL Synchronous Thread */

    .align  7
    b   _do_bad_irq /* Current EL IRQ Thread */

    .align  7
    b   _do_bad_fiq /* Current EL FIQ Thread */

    .align  7
    b   _do_bad_error   /* Current EL Error Thread */

    .align  7
    b   _do_sync    /* Current EL Synchronous Handler */

    .align  7
    b   _do_irq     /* Current EL IRQ Handler */

    .align  7
    b   _do_fiq     /* Current EL FIQ Handler */

    .align  7
    b   _do_error   /* Current EL Error Handler */

_do_bad_sync:
    exception_entry
    bl  do_bad_sync
    b   exception_exit

_do_bad_irq:
    exception_entry
    bl  do_bad_irq
    b   exception_exit

_do_bad_fiq:
    exception_entry
    bl  do_bad_fiq
    b   exception_exit
    ...
```

vbar_el1寄存器的定义如图 ，该寄存器用来保存vector的基地址，因此cpu发生异常后就可以根据保存在该寄存器中的地址值找到相应的异常向量表了。

接下来将立即数3左移20位后写入cpacr_el1中，该寄存器及其bit20/bit21的定义如下，设置这两位会关闭在EL0和EL1中SVE，SIMD和FP指令的trap功能。

#### 2.2.2.4 lowlevel_init

代码会执行lowlevel_init，它在start.s和lowlevel_init.S中都有定义，其中start.s中定义为weak类型，其代码如下。而lowlevel_init.S中是强符号定义，我们再看arch/arm/cpu/armv8/Makefile，其中有一句obj-$(CONFIG_ARCH_SUNXI) += lowlevel_init.o，即只有在SUNXI架构下才会使用该定义，其余架构下都是使用如下的weak定义的函数。

```text
WEAK(lowlevel_init)
    mov x29, lr         /* Save LR */                      （1）

#if defined(CONFIG_GICV2) || defined(CONFIG_GICV3)
    branch_if_slave x0, 1f                                 （2）
    ldr x0, =GICD_BASE                                     （3）
    bl  gic_init_secure                                    （4）
1:
#if defined(CONFIG_GICV3)
    ldr x0, =GICR_BASE                                     （5）
    bl  gic_init_secure_percpu                             （6）
#elif defined(CONFIG_GICV2)
    ldr x0, =GICD_BASE                                     （7）
    ldr x1, =GICC_BASE                                     （8）
    bl  gic_init_secure_percpu
#endif
#endif

#ifdef CONFIG_ARMV8_MULTIENTRY                             （9）
    branch_if_master x0, x1, 2f

    /*
     * Slave should wait for master clearing spin table.
     * This sync prevent salves observing incorrect
     * value of spin table and jumping to wrong place.
     */
#if defined(CONFIG_GICV2) || defined(CONFIG_GICV3)
#ifdef CONFIG_GICV2
    ldr x0, =GICC_BASE
#endif
    bl  gic_wait_for_interrupt
#endif

    /*
     * All slaves will enter EL2 and optionally EL1.
     */
    adr x4, lowlevel_in_el2
    ldr x5, =ES_TO_AARCH64
    bl  armv8_switch_to_el2

lowlevel_in_el2:
#ifdef CONFIG_ARMV8_SWITCH_TO_EL1
    adr x4, lowlevel_in_el1
    ldr x5, =ES_TO_AARCH64
    bl  armv8_switch_to_el1

lowlevel_in_el1:
#endif

#endif /* CONFIG_ARMV8_MULTIENTRY */

2:
    mov lr, x29         /* Restore LR */                     （10）
    ret
ENDPROC(lowlevel_init)
```

（1）该函数首先将链接寄存器的值lr保存到x29中，然后根据中断控制器的型号分别处理。假设我们系统中的中断控制器为GICV3，则会执行第二步。       
（2）branch_if_slave 定义在rch/arm/include/asm/macro.h中，代码如下。它会读取控制寄存器mpidr_el1的值，然后测试它的相应字段，以确定其是否slave。mpidr_el1寄存器用于在多处理器系统中标识不同的处理器，此处就是通过对该值的判断来确定当前处理器是否为master的。为了介绍方便，后面我们都假设当前cpu为master。       
（3）若当前cpu为master，则先将GICD_BASE的基地址加载到x0寄存器中       
（4）跳转到gic_init_secure宏中， 该宏的定义位于arm/lib/gic_64.S中，它的作用是为了初始化中断控制器gic。我们知道arm处理器的外设中断是通过irq和fiq中断线触发的，实际上在arm和外设之间还有一个处理中断的设备GIC，外设中断线连接到GIC上，当其中断线触发中断时GIC就会接收到中断事件，然后它根据配置情况将该中断分发给cpu，此时cpu才进入irq或fiq异常处理中断。       
（5）和（6）设置GIC对每个cpu相关的配置       
（9）arm的多处理器相关的设置，主要是slave cpu和master cpu同步相关的操作      
（10）恢复前面保存的lr值，并返回

```text
.macro  branch_if_slave, xreg, slave_label
#ifdef CONFIG_ARMV8_MULTIENTRY
    /* NOTE: MPIDR handling will be erroneous on multi-cluster machines */
    mrs \xreg, mpidr_el1
    tst \xreg, #0xff        /* Test Affinity 0 */
    b.ne    \slave_label
    lsr \xreg, \xreg, #8
    tst \xreg, #0xff        /* Test Affinity 1 */
    b.ne    \slave_label
    lsr \xreg, \xreg, #8
    tst \xreg, #0xff        /* Test Affinity 2 */
    b.ne    \slave_label
    lsr \xreg, \xreg, #16
    tst \xreg, #0xff        /* Test Affinity 3 */
    b.ne    \slave_label
#endif
.endm
```

接下来就是start.S中的最后一段代码如下：

```text
#if defined(CONFIG_ARMV8_SPIN_TABLE) && !defined(CONFIG_SPL_BUILD)          （1）
    branch_if_master x0, x1, master_cpu
    b   spin_table_secondary_jump
    /* never return */
#elif defined(CONFIG_ARMV8_MULTIENTRY)                                      （2）
    branch_if_master x0, x1, master_cpu

    /*
     * Slave CPUs
     */
slave_cpu:
    wfe
    ldr x1, =CPU_RELEASE_ADDR
    ldr x0, [x1]
    cbz x0, slave_cpu
    br  x0          /* branch to the given address */
#endif /* CONFIG_ARMV8_MULTIENTRY */
master_cpu:                                                                （3）
    bl  _main
```

（1）它只有在非spl时才执行。       
（2）它只有在多处理器时才执行，若当前cpu为master，则直接跳到（3），否则若为slave cpu，则执行wfe（wait for event）指令，该指令会让cpu休眠进入低功耗模式，此后该cpu将不再活动，直到SEV或SEVL指令唤醒它为止。因此，此后将只有master cpu会执行，而其它的cpu都进入休眠模式了。       
（3）跳转到_main处执行，该函数的定义位于arch/arm/lib/crt0_64.S中。它主要是初始化c语言的执行环境，crt的意思即为c run time。

#### 2.2.2.5 `_main`

`_main`的代码如下：

```text
ENTRY(_main)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */
#if defined(CONFIG_TPL_BUILD) && defined(CONFIG_TPL_NEEDS_SEPARATE_STACK)      （1）
    ldr x0, =(CONFIG_TPL_STACK)
#elif defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)                   
    ldr x0, =(CONFIG_SPL_STACK)
#else
    ldr x0, =(CONFIG_SYS_INIT_SP_ADDR)                                         
#endif
    bic sp, x0, #0xf    /* 16-byte alignment for ABI compliance */             （2）
    mov x0, sp                                                                 （3）
    bl  board_init_f_alloc_reserve                                             （4）
    mov sp, x0                                                                 （5）
    /* set up gd here, outside any C code */
    mov x18, x0                                                                （6）
    bl  board_init_f_init_reserve                                              （7）

    mov x0, #0                                                                 （8）
    bl  board_init_f                                                           （9）

#if !defined(CONFIG_SPL_BUILD)                                                 （10）
/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we'll return
 * 'here' but relocated.
 */
    ldr x0, [x18, #GD_START_ADDR_SP]    /* x0 <- gd->start_addr_sp */
    bic sp, x0, #0xf    /* 16-byte alignment for ABI compliance */
    ldr x18, [x18, #GD_BD]      /* x18 <- gd->bd */
    sub x18, x18, #GD_SIZE      /* new GD is below bd */

    adr lr, relocation_return
    ldr x9, [x18, #GD_RELOC_OFF]    /* x9 <- gd->reloc_off */
    add lr, lr, x9  /* new return address after relocation */
    ldr x0, [x18, #GD_RELOCADDR]    /* x0 <- gd->relocaddr */
    b   relocate_code

relocation_return:

/*
 * Set up final (full) environment
 */
    bl  c_runtime_cpu_setup     /* still call old routine */
#endif /* !CONFIG_SPL_BUILD */
#if defined(CONFIG_SPL_BUILD)                                               （11）
    bl  spl_relocate_stack_gd           /* may return NULL */               （12）
    /*
     * Perform 'sp = (x0 != NULL) ? x0 : sp' while working
     * around the constraint that conditional moves can not
     * have 'sp' as an operand
     */
    mov x1, sp                                                              （13）
    cmp x0, #0                                                              （14）
    csel    x0, x0, x1, ne                                                  （15）
    mov sp, x0                                                              （16）
#endif

/*
 * Clear BSS section
 */
    ldr x0, =__bss_start        /* this is auto-relocated! */               （17）
    ldr x1, =__bss_end          /* this is auto-relocated! */
clear_loop:                                                                 （18）
    str xzr, [x0], #8                                                       （19）
    cmp x0, x1                                                              （20）
    b.lo    clear_loop                                                      （21）

    /* call board_init_r(gd_t *id, ulong dest_addr) */
    mov x0, x18             /* gd_t */                                      （22）
    ldr x1, [x18, #GD_RELOCADDR]    /* dest_addr */                         （23）
    b   board_init_r            /* PC relative jump */                      （24）

    /* NOTREACHED - board_init_r() does not return */

ENDPROC(_main)
```

（1）将配置文件中设置的栈指针地址加载到x0寄存器中，在spl中应该是CONFIG_SPL_STACK的值，它一般位于include/configs/xxx中。       
（2）将x0寄存器中的值清除低4位，使其16字节对齐，然后将它存入栈指针寄存器sp中，在armv8中栈指针寄存器为x31。       
（3）由于sp中的值是做过对齐操作的，因此将其保存到x0中作为函数传参，在armv8中x0 - x7寄存器可以用于函数传参，其中x0为第一个参数。       
（4）调用board_init_f_alloc_reserve函数，它定义在common/init/board_init.c中，代码如下。即若定义了early malloc功能，则为malloc预留一些内存，其中top就是通过x0传入的参数，由于栈是向低地址伸展的，因此将高地址留给early malloc，只需要将栈地址往下移即可。在保留过之后，继续将新的指针做16字节对齐。该函数是一个c语言实现，由于c语言需要栈的支持，而上面的第二步已经设置了栈指针，因此调用该函数不会有问题。

```c
ulong board_init_f_alloc_reserve(ulong top)
{
    /* Reserve early malloc arena */
#if CONFIG_VAL(SYS_MALLOC_F_LEN)
    top -= CONFIG_VAL(SYS_MALLOC_F_LEN);
#endif
    /* LAST : reserve GD (rounded up to a multiple of 16 bytes) */
    top = rounddown(top-sizeof(struct global_data), 16);

    return top;
}
```

（5）将新的指针地址保存到SP中，以更新栈指针       
（6）将x0的值暂存到x18中，以腾出x0寄存器。由于栈是向低地址伸展，而步骤7介绍的gd是向高地址伸展的，因此它是栈顶指针，同时也是gd的基地址。因此，后续若需要使用gd，则可以直接从x18寄存器中取得它的指针。       
（7）board_init_f_init_reserve也是定义在common/init/board_init.c中，代码如下。base参数由x0传入，即当前的栈指针，将它作为gd的基地址，然后将gd到gd + sizeof（gd）之间的地址分配给global data并清空该段内存。将base指针更新为(align 16)(gd + sizeof（gd）)的位置。

我们知道，若前面保留了early malloc地址，则gd就被分配到early malloc的最低地址处，否则它会被分配到以sp为基地址的位置，因此若定义了early malloc，则需要更新malloc指针。因此这步的主要工作是在early malloc区域或者sp以上的区域为gd保留并清空一段内存空间，若是从early malloc中分配的，则随之更新malloc指针，更新后的内存布局如下图所示。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220720131001.png" width="50%" />
</div>

```c
void board_init_f_init_reserve(ulong base)
{
    struct global_data *gd_ptr;

    /*
     * clear GD entirely and set it up.
     * Use gd_ptr, as gd may not be properly set yet.
     */

    gd_ptr = (struct global_data *)base;
    /* zero the area */
    memset(gd_ptr, '\0', sizeof(*gd));
    /* set GD unless architecture did it already */
#if !defined(CONFIG_ARM)
    arch_setup_gd(gd_ptr);
#endif
    /* next alloc will be higher by one GD plus 16-byte alignment */
    base += roundup(sizeof(struct global_data), 16);

    /*
     * record early malloc arena start.
     * Use gd as it is now properly set for all architectures.
     */

#if CONFIG_VAL(SYS_MALLOC_F_LEN)
    /* go down one 'early malloc arena' */
    gd->malloc_base = base;
    /* next alloc will be higher by one 'early malloc arena' size */
    base += CONFIG_VAL(SYS_MALLOC_F_LEN);
#endif
}
```

（8）将立即数0放入x0寄存器，作为参数传给board_init_f函数       
（9）执行board_init_f函数，该函数的定义在arch/arm/lib/spl.c中，代码如下：

```c
void __weak board_init_f(ulong dummy)
{
}

```

该函数是一个空函数，但也带有__weak关键字。与我们上面分析的一样，它是一个弱函数，因此各平台可以根据自己的实际需求对其进行重定义。我们选取位于arch/arm/cpu/armv8/fsl-layerscape/spl.c中的定义为例，代码如下：

```c
void board_init_f(ulong dummy)
{
    /* Clear global data */
    memset((void *)gd, 0, sizeof(gd_t));                                   （a）
    board_early_init_f();                                                  （b）
    timer_init();                                                          （c）
#ifdef CONFIG_ARCH_LS2080A                                                 （d）
    env_init();
#endif
    get_clocks();                                                          （e）

    preloader_console_init();                                              （f）

#ifdef CONFIG_SPL_I2C_SUPPORT                                              （g）
    i2c_init_all();
#endif
    dram_init();                                                           （h）
}
```

该函数主要做一些board基本功能相关的初始化。如清空gd内存，定时器的初始化，获取系统时钟，总线时钟频率，console的初始化以及ddr的初始化等。下面对各步骤做一简要介绍:

（a）清空gd的内存。其中gd的定义位于arch/arm/include/asm/global_data.h中，它会从x18寄存器中获取gd指针，具体代码比较简单，这里不贴了。       
（b）这个函数是每个board特定的一些初始化操作。       
（c）定时器的初始化，对于fsl-layerscape平台其定义位于arch/arm/cpu/armv8/fsl-layerscape/cpu.c中，感兴趣的同学可以自行参阅。       
（d）与特定的配置相关       
（e）获取时钟频率，该函数的定义位于arch/arm/cpu/armv8/fsl-layerscape/fsl_lsch2_speed.c（fsl_lsch3_speed.c）中，它的主要功能是获取处理器0的cpu时钟频率，总线时钟频率和ddr时钟频率等。  
（f）该函数用于初始化串口，其定义位于common/spl/spl.c中，代码如下。它首先根据配置信息设置串口的波特率，然后调用serial_init函数初始化串口，初始化完成后串口就可以输出信息了，此时设置gd的have_console标志，后续的代码可以通过判断该标志来确定当前串口是否可用，最后若设置了相关配置，则打印一些spl相关的信息。

```text
void preloader_console_init(void)
{
    gd->baudrate = CONFIG_BAUDRATE;

    serial_init();      /* serial communications setup */

    gd->have_console = 1;

#if CONFIG_IS_ENABLED(BANNER_PRINT)
    puts("\nU-Boot " SPL_TPL_NAME " " PLAIN_VERSION " (" U_BOOT_DATE " - "
         U_BOOT_TIME " " U_BOOT_TZ ")\n");
#endif
#ifdef CONFIG_SPL_DISPLAY_PRINT
    spl_display_print();
#endif
}
```

（g）与特定配置相关，不做介绍。       
（h）ddr相关的初始化，对于fsl-layerscape平台会获取dram的size，并将其存放到gd->ram_size中

回到_main中，步骤（10）是uboo重定位流程，其在SPL时不执行，故此处对其不做分析。因为start.s和crt0_64.s都是spl和uboot共用的，故相关函数只是通过相应的宏定义来控制代码的执行流程。

（12）spl_relocate_stack_gd，该函数定义在common/spl/spl.c中。前面我们说过spl一般是运行在sram中，且此时的栈和gd数据都存放在sram中。但是现在ddr已经初始化完成，这时ddr已经可用，我们可以将其栈和gd重定位到ddr中。重定位的主要过程就是将栈指针，gd指针，malloc指针等设置到位于ddr中的新地址处，然后将老的gd数据等拷贝到新地址处。

（13）-（16）注释写的很清楚，将x0和立即数0比较，若其不等于0（NULL），则将sp设置为等于x0，否则保持原来的值不变，即根据上面步骤（12）的结果来确定是否更新栈指针。

（17）-（21）将bss段的内容清空。其中bss段的起始地址bss_start 和结束地址bss_end定义在spl的链接脚本arch/arm/cpu/armv8/u-boot-spl.lds中。其中循环的执行步骤为：       
str xzr, [x0], #8 ：xzr为0寄存器（x zero register），任何读该寄存器的操作都会返回0,。因此这条指令的含义是将0写入x0寄存器中内容为地址的内存中，然后x0 = x0 + 8.。由于xzr是64位寄存器，因此每次可以操作8个字节。

cmp x0, x1：比较x0和x1寄存器的内容，用来判断循环的退出条件 
b.lo clear_loop：实际执行判断，当x0小于x1，即若未执行到bss段的结束地址（__bss_end）时，继续跳转到clear_loop标号处执行循环，否则结束循环。       

（22）该操作将gd指针放入x0寄存器中，以作为参数传给board_init_r函数。       
（23）将x18 + GD_RELOCADDR地址的内容加载到x1中       
（14）调用board_init_r函数，此处跳转命令为b，而不是bl，因此它不会再返回。

board_init_r定义在common/spl/spl.c中，主要作用是进行一些必要的初始化工作，然后根据相关的配置情况，加载并启动下一阶段的镜像（一般为uboot）。由于该部分代码逻辑比较清晰，此处不再过多赘述。

# 3.Ref

[^1]:[AM335x Sitara Processors datasheet.pdf](https://github.com/carloscn/doclib/blob/master/man/arm/ti/AM335x%20Sitara%20Processors%20datasheet.pdf)
[^2]:[**AM335x and AMIC110 Sitara Processors TRM.pdf**](https://github.com/carloscn/doclib/blob/master/man/arm/ti/AM335x%20and%20AMIC110%20Sitara%20Processors%20TRM.pdf)
[^3]:[**armv8m_architecture_memory_protection_unit_100699_0100_00_en.pdf**](https://github.com/carloscn/doclib/blob/master/man/arm/armv8/armv8m_architecture_memory_protection_unit_100699_0100_00_en.pdf)
[^4]:[OP-TEE基本的从芯片设计到给客户的安全问题浅析](https://blog.csdn.net/zxpblog/article/details/107410971)
[^5]:[An Exploration of ARM TrustZone Technology](https://genode.org/documentation/articles/trustzone)
[^6]:[U-Boot 之五 详解 U-Boot 及 SPL 的启动流程](https://blog.csdn.net/ZCShouCSDN/article/details/121925283)
[^7]:[Embedded Linux Booting Process (Multi-Stage Bootloaders, Kernel, Filesystem)](https://www.youtube.com/watch?v=DV5S_ZSdK0s)
[^8]:[**i.MX 8QuadMax Applications Processor Reference Manual.pdf**](https://github.com/carloscn/doclib/blob/master/man/embedded/nxp/i.MX%208QuadMax%20Applications%20Processor%20Reference%20Manual.pdf)
[^9]:[# 02_ARMv8 ATF Secure Boot Flow (BL1/BL2/BL31)](https://github.com/carloscn/blog/issues/65)
[^10]:[uboot-REAMD.TPL](https://github.com/ARM-software/u-boot/blob/master/doc/README.TPL)
[^11]:[# 聊聊SOC启动（七） SPL启动分析](https://zhuanlan.zhihu.com/p/520189611)
[^12]:[# U-boot中SPL功能和源码流程分析](http://t.zoukankan.com/dylancao-p-8621789.html)
[^13]:[# [uboot] （第二章）uboot流程——uboot-spl编译流程](https://blog.csdn.net/ooonebook/article/details/52949584)
[^14]:[# [uboot] （第三章）uboot流程——uboot-spl代码流程](https://blog.csdn.net/ooonebook/article/details/52957395?spm=1001.2014.3001.5502)
[^15]:[test_strong_weak_symbol](https://github.com/carloscn/clab/tree/master/macos/test_strong_weak_symbol)
[^16]:[Develop U-Boot - Global data](https://u-boot.readthedocs.io/en/latest/develop/global_data.html)
[^17]:[]()
[^18]:[]()
[^19]:[]()







