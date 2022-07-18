# 02_ARMv8 ATF Boot Flow (BL1/BL2/BL31)

ARMv7和ARMv8在引导流程上面完全不同的思路。ARMv8要兼容secure boot，需要在不同的异常等级做相应的处理，而且还需要给SoC厂商一些可配的灵活度，所以在boot上会引入不同的概念，相应的，比ARMv7（及以前）设计层面的复杂度要高很多。

# 1. boot high-level

在ARMv8里面boot程序是有trustzone-firmware执行引导，参考：https://github.com/carloscn/arm-trusted-firmware/ 里面设计boot armv8的所有流程。

## 1.1 boot overview

ARMv8流程相比如ARMv7要的多，包含了多个新引入的阶段，包含BL1、BL2、BL3（BL31/32/33），这些阶段可以适当的进行剪裁或者添加，并可以被编译成独立的启动镜像。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220711115835952.png" width="70%" />
</div>

*   **【BL1- BootROM】**：(boot level 1: bootrom stage) 被定义为ARMv8的SoC的启动第一阶段，**若芯片支持XIP技术，可以被存储到片外的存储介质上面（norflash**）。若**不支持XIP技术**，则需要存储在芯片的片内**BootROM中**（存储在BootROM中的代码需要在SoC厂商在流片的时候固化，无法做任何修改）。若芯片支持secure boot，则需要在BootROM中存储启动时候的信任根，比如OTP的public key hash。**在BL1阶段，ARMv8的异常等级为最高EL3等级**。除此之外，BL1还需要将BL2的image并跳转。（在BootROM中执行，不需要RAM）
*   **【BL2 - Boot Firmware】**：可视为ARMv7 SPL的流程，此时DDR还没有初始化，这一步总需要被加载到片内的SRAM执行，一般在这个阶段会完成DDR的初始化，因此后面的image可以被加载到DDR中。BL3所有的image都是由BL2加载的，BL31/BL32是可选的配置。若没有trusted os可以没有BL2，若不支持EL3异常等级及secure monitor，则可以去掉BL31。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220711121724788.png" width="40%" />
</div>

*   【BL33 - bootloader】：等同于ARMv7的second stage bootloader，通常是uboot，由uboot启动kernel。这个步骤SoC处于EL2-non-secure模式。

## 1.2 exception switching

ARMv8关于启动的时候异常等级的切换如下所示，假定该系统支持最高的异常等级EL3，且支持secure monitor和Trusted OS，同时BL2运行Secure E1，EL33运行在non-seucre EL1的状态。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220711134600895.png" width="60%" />
</div>

*	（1）由于ARMv8规定，ARM core复位之后进入到当前系统支持最高的异常等级，因此BL1运行在EL3，它执行完毕之后会通过ERET的方式跳转到EL2，当前BL2处于S-EL1异常等级，可以见得，从BL1->BL2这个过程，属于降权过程EL3->S/EL1（越过了EL2等级）

*	（2/3）BL1引导BL2的时候根据不同的应用场景有两个形式，这两种形式影响着Boot的引导模式和配置。**第一种是BL2处于Secure-EL1异常等级**，在这种模式下就涉及ARM core的异常等级的切换，在BL1处于最高权限EL3，要进入BL2-Secure-EL1这种状态需要使用ERET指令，但若要进一步引导BL31，BL31又回到了EL3，S-EL1只能通过SMC指令进入更高的EL3等级，而且需要注意的是，由于BL31在这个时候还没有注册异常处理，无法支持处理该异常，因此就需要在BL1程序中初始化该异常，帮忙代理处理BL31的异常处理。因此BL1在退出之前先设置smc异常处理函数，BL2触发smc启动BL31时，BL1捕获该异常并根据BL2传入的参数设置BL31的入口地址和系统初始状态，并通过ERET跳转到BL31的入口地址处执；**第二种，若BL2在EL3这种应用场景，直接通过ERET路由到BL31**，前几种状态都是EL3，并没有涉及异常等级的切换。

*	（4/5/6）BL32阶段会引导trusted os启动，它运行于EL1异常等级，BL31可根据其镜像加载信息设置入口地址和其他状态，完成跳转。BL32加载完成后通过SMC返回到BL31，和BL2引导BL31的情景一致，需要BL31代理处理权限变换的异常。

## 1.3 memory layout

通常的SoC包含ROM、SRAM、DDR，其中ROM和SRAM位于SoC内部，DRAM位于SoC外部[^1]。

>   它们的特点如下：
>   （1）ROM中的内容断电后不会消失，不仅可用于代码执行，还可以用于镜像存储，但其只有只读权限
>   （2）SRAM和DDR中的内容在断电后都会消失，因此只能被用于代码的动态执行，而不能用于镜像存储
>   （3）ROM和SRAM都是直接连接总线上，系统上电后即可直接执行。而DDR需要通过DDR phy和DDR controller连接到总线上，因此使用之前必须要先对其执行初始化操作
>
>   根据上述各种内存的特点和前面镜像加载启动流程的需求，在内存规划中我们需要考虑以下几个问题：
>   （1）由于BL1需要固化到ROM中，且是系统最先执行的，因此ROM地址需要被映射到cpu的重启地址处
>   （2）由于ROM是只读的，BL1镜像除了代码段和只读数据段之外还包含可读写数据段，这部分数据在BL1启动时需要从ROM重定位到SRAM中
>   （3）由于BL1被固化在ROM中，芯片出产后就不能更改，因此DDR初始化代码不能集成到BL1中。故BL2需要被加载到SRAM中执行，且在BL2中执行DDR初始化流程
>   （4）BL2之后的其它镜像既可以运行于SRAM中，也可以运行于DDR中
>   （5）从前面的镜像启动流程可知，若BL2运行于secure EL1下，当其执行完成后，需要通过smc再次陷入BL1去执行BL31流程，因此BL2和BL1的地址不能有重叠
>   （6）BL31除了执行启动流程外，在系统运行过程中还会以secure monitor的方式驻留，为normal空间的smc异常提供服务历程，以及为normal os和trust os之间提供消息转发、中断路由转发等功能。因此，BL31镜像需要永久驻留内存，在系统启动完成后不能被回收
>   （7）与BL31类似，BL32在启动后需要驻留内存为系统提供安全相关服务，因此为其所分配的内存也不能被回收
>   （8）除此之外，BL1、BL2和BL33（一般为uboot）的内存在系统启动完成后都可以被释放给操作系统使用
>
>   根据以上内存规划原则，qemu virt machine各启动阶段的内存规划如下：
>
>   | 类型 | 起始地址   | 结束地址    | 长度 | 是否secure内存 | 作用           |
>   | ---- | ---------- | ----------- | ---- | -------------- | -------------- |
>   | ROM  | 0x00000000 | 0x00020000  | 128k | Yes            | BL1（bootrom） |
>   | SRAM | 0x0e04e000 | 0x0e060000  | 72k  | Yes            | Bl1（rw data） |
>   | SRAM | 0x0e000000 | 0x0e001000  | 4k   | Yes            | Shared ram     |
>   | SRAM | 0x0e01b000 | 0x0e040000  | 148k | Yes            | BL2            |
>   | SRAM | 0x0e040000 | 0x0e060000  | 128k | Yes            | BL31           |
>   | SRAM | 0x0e001000 | 0x0e040000  | 252k | YES            | BL32           |
>   | DDR  | 0x60000000 | 0x100000000 | 2.5G | NO             | BL33           |

# 2. SoC BL1 Booting[^2]

BL1是SoC系统启动的第一个阶段，其主要目的是初始化系统环境和启动第二阶段镜像BL2。在Trust zone firmware里面被定义为`Architectural initialization`，`platform initaliztion`, `firmware update detection and execution`阶段[^12]，顾名思义，在这个阶段需要对整个SoC完成初始化。需要包含以下流程：

*   Architectural initialization
    *   Exception vectors 初始化异常向量表
    *   CPU initializtion CPU初始化
    *   Control register setup for aarch64 控制寄存器配置
*   platform initaliztion
    *   Enable the Trusted watchdog 使能看门狗
    *   initialze the console 初始化控制台
    *   configure the interconnect to enable hardware coherency 配置使能硬件一致性维护机制
    *   enable MMU and map the memory it needs to access 使能MMU，映射内存
    *   configure any required platform storage to load the next bootloader image(BL2) load bl2
*   firmware update detection and execution

**在bl1/bl1.ld.S中通过ENTRY标号定义为**：

```cmake
OUTPUT_FORMAT(PLATFORM_LINKER_FORMAT)
OUTPUT_ARCH(PLATFORM_LINKER_ARCH)
ENTRY(bl1_entrypoint)
```

我们可以把bl_entrypoint（ https://github.com/carloscn/arm-trusted-firmware/blob/master/bl1/aarch64/bl1_entrypoint.S#L22 ）总结如下思维导图。

![bl_entrypoint](https://raw.githubusercontent.com/carloscn/images/main/typorabl_entrypoint.svg)

## 2.1 el3_entrypoint_common flow

https://github.com/carloscn/arm-trusted-firmware/blob/master/bl1/aarch64/bl1_entrypoint.S#L29

```assembly
	/* ---------------------------------------------------------------------
	 * If the reset address is programmable then bl1_entrypoint() is
	 * executed only on the cold boot path. Therefore, we can skip the warm
	 * boot mailbox mechanism.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_init_sctlr=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=bl1_exceptions		\
		_pie_fixup_size=0
```

该宏所有需要在EL3下执行的镜像共享，如BL1和BL31都会入口处调用该函数，只是传入的参数有所区别。其主要完成的功能如下：
*   初始化sctlr_el3寄存器，以初始化系统控制参数。 （sctlr_el3寄存器）
*   判断当前启动方式是冷启动还是热启动，并执行相应的处理。
*   PIE相关的处理
*   设置异常向量表
*   特定cpu相关的reset处理
*   了架构相关的el3的初始化
*   冷启动时secondary cpu的处理
*   c运行环境初始化
*   初始化运行栈

### 2.1.1 sctlr_el3 config

关于寄存器引用一下说明[^13]：
>sctlr_el3系统控制寄存器layout如下，主要提供系统high-level层级的状态控制信息。
>
><div align='center'>
><img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220712145930487.png" width="80%" />
></div>
>
>*   M：用于设置系统是否使能EL3下面的MMU
>*   A：用于设置是否使能EL3的对齐检查
>*   C：用于设置EL3下面d-cache
>*   I：用于设置EL3下面的i-cache
>*   WXN：用于设置EL3下写权限内存是否不可执行。
>*   EE：配置EL3的大小端

这个实现被定义在：https://github.com/carloscn/arm-trusted-firmware/blob/master/include/arch/aarch32/el3_common_macros.S#L193

```assembly
	.if \_init_sctlr
		/* -------------------------------------------------------------
		 * This is the initialisation of SCTLR and so must ensure that
		 * all fields are explicitly set rather than relying on hw. Some
		 * fields reset to an IMPLEMENTATION DEFINED value.
		 *
		 * SCTLR.TE: Set to zero so that exceptions to an Exception
		 *  Level executing at PL1 are taken to A32 state.
		 *
		 * SCTLR.EE: Set the CPU endianness before doing anything that
		 *  might involve memory reads or writes. Set to zero to select
		 *  Little Endian.
		 *
		 * SCTLR.V: Set to zero to select the normal exception vectors
		 *  with base address held in VBAR.
		 *
		 * SCTLR.DSSBS: Set to zero to disable speculation store bypass
		 *  safe behaviour upon exception entry to EL3.
		 * -------------------------------------------------------------
		 */
		ldr     r0, =(SCTLR_RESET_VAL & ~(SCTLR_TE_BIT | SCTLR_EE_BIT | \
				SCTLR_V_BIT | SCTLR_DSSBS_BIT))
		stcopr	r0, SCTLR
		isb
	.endif /* _init_sctlr */
```

在el1层级主要设定包含：
*   系统的大小端
*   禁止了对齐错误、栈对齐错误检查
*   以及禁止了可写内存的执行权限
*   其他值沿用SCTLR_RESET_VAL的定义

### 2.1.2 cold_boot and warm_boot

冷启动和热启动的最大区别是冷启动需要完整的系统初始化流程，而热启动在启动之前保存了相关的状态，因此可以跳过这些阶段，从而加速启动速度。

```assembly
	.if \_warm_boot_mailbox
		/* -------------------------------------------------------------
		 * This code will be executed for both warm and cold resets.
		 * Now is the time to distinguish between the two.
		 * Query the platform entrypoint address and if it is not zero
		 * then it means it is a warm boot so jump to this address.
		 * -------------------------------------------------------------
		 */
		bl	plat_get_my_entrypoint
		cbz	x0, do_cold_boot
		br	x0

	do_cold_boot:
	.endif /* _warm_boot_mailbox */
```

先通过`plat_get_my_entrypoint`从特定的平台获取启动地址，若获取地址成功之后直接执行热启动流程。若地址获取失败，该函数会返回0，此时表明本次启动是冷启动，因此需要执行冷启动启动流程。

### 2.1.3 PIE dealing

在计算机领域中，地址无关代码 (position-independent code，PIC)，又称地址无关可执行文件 (position-independent executable，PIE) ，是指可在[主存储器](https://baike.baidu.com/item/主存储器/10635399)中任意位置正确地运行，而不受其绝对地址影响的一种机器码[^14]。在ELF文件中，标定了代码段、数据段和符合表这些地址，而代码的执行或者访问数据的时候是需要访问这些地址的，而加载地址和链接地址并不是同一个地址，这个时候寻址就会失败。PIE就是为了解决该问题。

```assembly
#if ENABLE_PIE
		/*
		 * ------------------------------------------------------------
		 * If PIE is enabled fixup the Global descriptor Table only
		 * once during primary core cold boot path.
		 *
		 * Compile time base address, required for fixup, is calculated
		 * using "pie_fixup" label present within first page.
		 * ------------------------------------------------------------
		 */
	pie_fixup:
		ldr	x0, =pie_fixup
		and	x0, x0, #~(PAGE_SIZE_MASK)
		mov_imm	x1, \_pie_fixup_size
		add	x1, x1, x0
		bl	fixup_gdt_reloc
#endif /* ENABLE_PIE */
	.endif /* _pie_fixup_size */
```

程序中的函数调用和数据读写，若其可以转换为相对寻址的，则将其用相对寻址方式代替绝对地址。如ARM64的adr执行，通过pc+offset的方式寻址（缺陷是寻址范围）。

若地址不能转换为相对寻址，则将其放在一个独立的段内Global Descriptor Table(GDT)，并在镜像启动时通过实际加载地址调整这些地址值。因此，PIE的实现需要编译和加载的共同完成，在构建时候添加如下选项`-fpie`，链接是添加选项`-pie`。在加载时需要对GDT表中内容进行调整，以上代码就是处于这个目的。

### 2.1.4 exception config

#### 2.1.4.1 exception vector

将bl1的异常向量表加载到el3的向量表基地址寄存器中，代码如下:

```assembly
	/* ---------------------------------------------------------------------
	 * Set the exception vectors.
	 * ---------------------------------------------------------------------
	 */
	adr	x0, \_exception_vectors
	msr	vbar_el3, x0
	isb
```

异常向量表在：https://github.com/carloscn/arm-trusted-firmware/blob/master/bl1/aarch64/bl1_exceptions.S

#### 2.1.4.2 reset handler

reset handler用于执行特定cpu的reset处理函数，这些处理函数在定义的时候会被放在一个特殊的段中，在执行reset_handler函数时候就从该段中查找操作函数的函数指针，并且执行相应的回调函数。以cortex-a53为例子，其cpu ops的定义流程如下：https://github.com/carloscn/arm-trusted-firmware/blob/master/lib/cpus/aarch64/cortex_a53.S

```assembly
lib/cpus/aarch64/cortex_a53.S：
declare_cpu_ops cortex_a53, CORTEX_A53_MIDR, \
	cortex_a53_reset_func, \
	cortex_a53_core_pwr_dwn, \
	cortex_a53_cluster_pwr_dwn

-->

include/lib/cpus/aarch64/cpu_macros.S：
.macro declare_cpu_ops _name:req, _midr:req, _resetfunc:req, \
		_power_down_ops:vararg
		declare_cpu_ops_base \_name, \_midr, \_resetfunc, 0, 0, 0, \
			\_power_down_ops
.endm

-->

include/lib/cpus/aarch64/cpu_macros.S：
.macro declare_cpu_ops_base _name:req, _midr:req, _resetfunc:req, \
                _extra1:req, _extra2:req, _e_handler:req, _power_down_ops:vararg
        .section cpu_ops, "a"
        .align 3
        .type cpu_ops_\_name, %object
        .quad \_midr
#if defined(IMAGE_AT_EL3)
        .quad \_resetfunc
#endif
        .quad \_extra1
        .quad \_extra2
        .quad \_e_handler
…
.endm
```

其中cpu_ops段的地址定义在链接脚本头文件include/common/bl_common.ld.h中，即该地址位于__CPU_OPS_START__到__CPU_OPS_END__之间

```assembly
	/* -------------------------------------------------
	 * The CPU Ops reset function for Cortex-A53.
	 * Shall clobber: x0-x19
	 * -------------------------------------------------
	 */
func cortex_a53_reset_func
	mov	x19, x30
	bl	cpu_get_rev_var
	mov	x18, x0


#if ERRATA_A53_826319
	mov	x0, x18
	bl	errata_a53_826319_wa
#endif

#if ERRATA_A53_836870
	mov	x0, x18
	bl	a53_disable_non_temporal_hint
#endif

#if ERRATA_A53_855873
	mov	x0, x18
	bl	errata_a53_855873_wa
#endif

	/* ---------------------------------------------
	 * Enable the SMP bit.
	 * ---------------------------------------------
	 */
	mrs	x0, CORTEX_A53_ECTLR_EL1
	orr	x0, x0, #CORTEX_A53_ECTLR_SMP_BIT
	msr	CORTEX_A53_ECTLR_EL1, x0
	isb
	ret	x19
endfunc cortex_a53_reset_func
```

reset_handler的流程比较简单，就是查找__CPU_OPS_START__到__CPU_OPS_END__之间的cpu_ops结构体，并调用其reset_func回调函数，具体流程不再赘述。对于cortex-a53 平台，其reset函数定义如下，该流程主要是执行一些cpu相关的errata操作，以及使能SMP位。

### 2.1.5 platform level init

该流程主要执行一些系统寄存器相关的配置，以设置系统的状态。其aarch64架构流程如下：

```assembly
.macro el3_arch_init_common
	mov	x1, #(SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)                           （1）
	mrs	x0, sctlr_el3
	orr	x0, x0, x1
	msr	sctlr_el3, x0
	isb

#ifdef IMAGE_BL31
	bl	init_cpu_data_ptr
#endif /* IMAGE_BL31 */
	mov_imm	x0, ((SCR_RESET_VAL | SCR_EA_BIT | SCR_SIF_BIT) \
			& ~(SCR_TWE_BIT | SCR_TWI_BIT | SCR_SMD_BIT))                       （2）
#if CTX_INCLUDE_PAUTH_REGS
	orr	x0, x0, #(SCR_API_BIT | SCR_APK_BIT)                                        （3）
#endif
	msr	scr_el3, x0
		mov_imm	x0, ((MDCR_EL3_RESET_VAL | MDCR_SDD_BIT | \
		      MDCR_SPD32(MDCR_SPD32_DISABLE) | MDCR_SCCD_BIT | \
		      MDCR_MCCD_BIT) & ~(MDCR_SPME_BIT | MDCR_TDOSA_BIT | \
		      MDCR_TDA_BIT | MDCR_TPM_BIT))                                    （4）

	msr	mdcr_el3, x0
	mov_imm	x0, ((PMCR_EL0_RESET_VAL | PMCR_EL0_LP_BIT | \
		      PMCR_EL0_LC_BIT | PMCR_EL0_DP_BIT) & \
		    ~(PMCR_EL0_X_BIT | PMCR_EL0_D_BIT))                                 （5）

	msr	pmcr_el0, x0
		msr	daifclr, #DAIF_ABT_BIT                                                  （6）
	mov_imm x0, (CPTR_EL3_RESET_VAL & ~(TCPAC_BIT | TTA_BIT | TFP_BIT))           （7）         
	msr	cptr_el3, x0

	mrs	x0, id_aa64pfr0_el1
	ubfx	x0, x0, #ID_AA64PFR0_DIT_SHIFT, #ID_AA64PFR0_DIT_LENGTH                 （8）
	cmp	x0, #ID_AA64PFR0_DIT_SUPPORTED
	bne	1f
	mov	x0, #DIT_BIT
	msr	DIT, x0
1:
	.endm
```

*   使能i-cache，地址\栈对齐检查
*   secure寄存器的相关配置，主要用于设置某些操作是否路由到EL3执行，如果设置SCR_EA_BIT会将所有的异常等级下的external abort和serror路由到EL3来处理。
*   API和APK配置用于使能指针签名特性PAC。由于armv8的虚拟地址没有完全使用，48位虚拟地址，其高16位是空闲的，完全可以用于存储一些其他信息。因此arm支持了指针签名技术，它通过密钥和签名算法对指针进行签名，并将截断后的签名保存到虚拟地址的高位，在使用该指针时则对高签名进行验证，以确保没有被篡改。它主要是保护栈中的数据安全，防御ROP JOP攻击。
*   mdcr_el3寄存器用于设置debug和performance monitor的相关功能
*   用于设置performance monitor配置，如一些性能事件计数器的行为
*   用于使能serror异常，此后bl1能够接受serror异常，并处理SMC调用
*   设置一些特定时间是否要陷入EL3
*   设置DIT特性，使能了DIT，则DIT相关的指令执行时间与数据不相关。由侧道攻击可以利用某些敏感指令的执行时间和功耗来推断出数据的内容。

### 2.1.6  secondary cpu

由于启动代码不支持并发，因此在smp系统中只有一个cpu(primary cpu)执行启动流程，而其他cpu(scondary cpu)需要将自身设定为一个安全状态，等待primary cpu启动完成之后通过spintable或psci等方式启动他们。

```assembly
bl	plat_is_my_cpu_primary                          （1）
		cbnz	w0, do_primary_cold_boot                （2）
		bl	plat_secondary_cold_boot_setup            （3）
		bl	el3_panic
do_primary_cold_boot:
```

*	(1) 当前CPU是否为primary CPU
*	(2) 若其为primary CPU，继续执行cold boot流程
*	(3) 若为secondary CPU，执行平台定义的secondary CPU启动设置函数。

### 2.1.7 memory init

执行平台的 platform_mem_init

### 2.1.8 C env init

C语言运行需要依赖stack和bss段，因此在跳转到c函数之前需要设置他们。而且由于bl1的镜像一般烧写在rom中，因此需要将其可写数据段从rom重定位到ram中。以下代码实现：

```assembly
		adrp	x0, __RW_START__
		add	x0, x0, :lo12:__RW_START__                                    （1）
		adrp	x1, __RW_END__
		add	x1, x1, :lo12:__RW_END__
		sub	x1, x1, x0                                                     （2）
		bl	inv_dcache_range                                               （3）
        …
		adrp	x0, __BSS_START__
		add	x0, x0, :lo12:__BSS_START__
		adrp	x1, __BSS_END__
		add	x1, x1, :lo12:__BSS_END__
		sub	x1, x1, x0                                                                       
		bl	zeromem                                                       （4）
        …
#if defined(IMAGE_BL1) || (defined(IMAGE_BL2) && BL2_AT_EL3 && BL2_IN_XIP_MEM)
		adrp	x0, __DATA_RAM_START__
		add	x0, x0, :lo12:__DATA_RAM_START__                              
		adrp	x1, __DATA_ROM_START__
		add	x1, x1, :lo12:__DATA_ROM_START__                              
		adrp	x2, __DATA_RAM_END__
		add	x2, x2, :lo12:__DATA_RAM_END__                                
		sub	x2, x2, x0                                                       
		bl	memcpy16                                                      （5）
#endif
```

*	（1）计算数据段的起始位置，由于adrp指令加载地址会将低12bit mask掉，使其4k对齐。因此需要加上其低12bit的数据，恢复其原始值。
*	（2）计算段的长度
*	（3）失效这段sram内存的dcache
*	（4）获取bss段的起始地址，并计算长度，清零
*	（5）获取bl1可读写数据段在rom中的地址，以及将要被重定位的ram地址，计算数据长度，并执行重定位操作。

### 2.1.9 运行栈设置

C语言的函数调用返回地址，上层栈指针地址、局部变量以及参数传递都可能用到栈。本函数通过设定运行栈指针，为跳转到c语言执行做准别，其代码如下：

```assembly
msr	spsel, #0                                      （1）
		bl	plat_set_my_stack                      （2）
#if STACK_PROTECTOR_ENABLED
	.if \_init_c_runtime
	bl	update_stack_protector_canary                （3）
	.endif
#endif
```

*	（1）使用sp_el0作为栈指针寄存器
*	（2）设置运行栈，该函数会获取一个定义好的栈指针，并将其设置到当前栈指针寄存器sp中
*	（3）在栈顶设置一个canary值，用于检测栈溢出。

## 2.2 bl1_setup

这部分是关于平台初始化的代码，主要调用的子函数：

*   bl1_early_platform_setup函数
*   bl1_plat_arch_setup函数

### 2.2.1 bl1_early_platform_setup

这部分代码都是平台相关部分的实现，我们从arm-trusted-firmware中搜寻到这部分代码的实现，全部都是在plat目录，而且这部分在文档中也提出了要求。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220713140142320.png" width="80%" />
</div>

```tex
Function : bl1_early_platform_setup() [mandatory]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::
    Argument : void
    Return   : void
This function executes with the MMU and data caches disabled. It is only called
by the primary CPU.
On Arm standard platforms, this function:
-  Enables a secure instance of SP805 to act as the Trusted Watchdog.
-  Initializes a UART (PL011 console), which enables access to the ``printf``
   family of functions in BL1.
-  Enables issuing of snoop and DVM (Distributed Virtual Memory) requests to
   the CCI slave interface corresponding to the cluster that includes the
   primary CPU.
```

我们来参考一下marvell的：

```c
/*
 * BL1 specific platform actions shared between Marvell standard platforms.
 */
void marvell_bl1_early_platform_setup(void)
{
	/* Initialize the console to provide early debug support */
	marvell_console_boot_init();

	/* Allow BL1 to see the whole Trusted RAM */
	bl1_ram_layout.total_base = MARVELL_BL_RAM_BASE;
	bl1_ram_layout.total_size = MARVELL_BL_RAM_SIZE;
}
```

这部分的实现就是：

*   控制台初始化
*   设置secure ram的内存地址范围。

### 2.2.2 bl1_plat_arch_setup

同样的bl1_plat_arch_setup也是需要交给平台来porting的代码，在trusted firmware内部，要求的是：

>Function : bl1_plat_arch_setup() [mandatory]
>
>This function performs any platform-specific and architectural setup that the
>platform requires. Platform-specific setup might include configuration of
>memory controllers and the interconnect.
>
>In Arm standard platforms, this function enables the MMU.
>
>This function helps fulfill requirement 2 above.

```c
/*
 * Perform the very early platform specific architecture setup shared between
 * MARVELL standard platforms. This only does basic initialization. Later
 * architectural setup (bl1_arch_setup()) does not do anything platform
 * specific.
 */
void marvell_bl1_plat_arch_setup(void)
{
	marvell_setup_page_tables(bl1_ram_layout.total_base,
				  bl1_ram_layout.total_size,
				  BL1_RO_BASE,
				  BL1_RO_LIMIT,
				  BL1_RO_DATA_BASE,
				  BL1_RO_DATA_END
#if USE_COHERENT_MEM
				, BL_COHERENT_RAM_BASE,
				  BL_COHERENT_RAM_END
#endif
				);
	enable_mmu_el3(0);
}
```

该函数用于为所有bl1需要访问地址建立MMU页表，并且使能d-cache。注意，bl1中的物理地址和虚拟地址属于恒等映射，之所以开启MMU主要是为了开启D-cache，以加快后面BL2镜像加载速度。

## 2.3 bl1_main

bl1_main函数属于真正意义的进入到了bl1的启动流程，前面都是做一些准备工作。bl1_main需要做的工作是：

*   合法性检查，包括一些寄存器的配置，cache writeback granule
*   Ensure that MMU/Caches and coherency are turned on
*   Perform remaining generic architectural setup from EL3 
*   Initialize authentication module
*   Initialize the measured boot
*   Perform platform setup in BL1.
*   Get the image id of next image to load and run. 
*   Teardown the measured boot driver 

```C
/*******************************************************************************
 * Function to perform late architectural and platform specific initialization.
 * It also queries the platform to load and run next BL image. Only called
 * by the primary cpu after a cold boot.
 ******************************************************************************/
void bl1_main(void)
{
	unsigned int image_id;

	/* Announce our arrival */
	NOTICE(FIRMWARE_WELCOME_STR);
	NOTICE("BL1: %s\n", version_string);
	NOTICE("BL1: %s\n", build_message);

	INFO("BL1: RAM %p - %p\n", (void *)BL1_RAM_BASE, (void *)BL1_RAM_LIMIT);

	print_errata_status();

#if ENABLE_ASSERTIONS
	u_register_t val;
	/*
	 * Ensure that MMU/Caches and coherency are turned on
	 */
#ifdef __aarch64__
	val = read_sctlr_el3();
#else
	val = read_sctlr();
#endif
	assert((val & SCTLR_M_BIT) != 0);
	assert((val & SCTLR_C_BIT) != 0);
	assert((val & SCTLR_I_BIT) != 0);
	/*
	 * Check that Cache Writeback Granule (CWG) in CTR_EL0 matches the
	 * provided platform value
	 */
	val = (read_ctr_el0() >> CTR_CWG_SHIFT) & CTR_CWG_MASK;
	/*
	 * If CWG is zero, then no CWG information is available but we can
	 * at least check the platform value is less than the architectural
	 * maximum.
	 */
	if (val != 0)
		assert(CACHE_WRITEBACK_GRANULE == SIZE_FROM_LOG2_WORDS(val));
	else
		assert(CACHE_WRITEBACK_GRANULE <= MAX_CACHE_LINE_SIZE);
#endif /* ENABLE_ASSERTIONS */

	/* Perform remaining generic architectural setup from EL3 */
	bl1_arch_setup();

	crypto_mod_init();

	/* Initialize authentication module */
	auth_mod_init();

	/* Initialize the measured boot */
	bl1_plat_mboot_init();

	/* Perform platform setup in BL1. */
	bl1_platform_setup();

#if ENABLE_PAUTH
	/* Store APIAKey_EL1 key */
	bl1_apiakey[0] = read_apiakeylo_el1();
	bl1_apiakey[1] = read_apiakeyhi_el1();
#endif /* ENABLE_PAUTH */

	/* Get the image id of next image to load and run. */
	image_id = bl1_plat_get_next_image_id();

	/*
	 * We currently interpret any image id other than
	 * BL2_IMAGE_ID as the start of firmware update.
	 */
	if (image_id == BL2_IMAGE_ID)
		bl1_load_bl2();
	else
		NOTICE("BL1-FWU: *******FWU Process Started*******\n");

	/* Teardown the measured boot driver */
	bl1_plat_mboot_finish();

	bl1_prepare_next_image(image_id);

	console_flush();
}
```

### 2.3.1 bl1_arch_setup

```C
void bl1_arch_setup(void) {
	write_scr_el3(read_scr_el3() | SCR_RW_BIT);
}
```

**该函数的作用为将下一个异常等级的执行状态设置为aarch64**

### 2.3.2 secure boot init

Secure boot用于校验镜像的合法性，它通常需要一个包含镜像签名信息的镜像头。签名信息可在打包时完成，一般包括计算镜像的hash值，然后使用非对称算法（如RSA或ECDSA）对该hash值执行签名操作，并将签名信息保存到镜像头中。在系统启动时，需要校验该签名是否合法，若不合法表明镜像被破坏或被替换了，因此系统需要停止启动流程。

这部分包含两个为空的实现，**一个是对加密引擎的初始化操作 crypto_mod_init，还有secboot的auth_mod_init**。需要用户来实现这部分。我们的secure boot的bl1的初始化就在这个位置。

### 2.3.3 bl1_platform_setup

>Function : bl1_platform_setup() [mandatory]
>
>This function executes with the MMU and data caches enabled. It is responsible
>for performing any remaining platform-specific setup that can occur after the
>MMU and data cache have been enabled.
>
>if support for multiple boot sources is required, it initializes the boot
>sequence used by plat_try_next_boot_source().
>
>In Arm standard platforms, this function initializes the storage abstraction
>layer used to load the next bootloader image.
>
>This function helps fulfill requirement 4 above.

这部分需要SoC厂商实现自己的平台初始化。并声明到这里，MMU和d-cache已经被打开了。需要注意这些问题。

### 2.3.4 load image

剩下就是获取image的id和对bl2的镜像的加载了，镜像加载流程包含了镜像从storage中的加载以及镜像合法性验证两部分，这部分就进入到了secboot的真正的逻辑部分。

#### 2.3.4.1 get image id

这部分由自己soc厂商负责，这个实现是hikey，作为一种参考。

```c
/*
 * The following function checks if Firmware update is needed,
 * by checking if TOC in FIP image is valid or not.
 */
unsigned int bl1_plat_get_next_image_id(void)
{
	int32_t boot_mode;
	unsigned int ret;

	boot_mode = mmio_read_32(ONCHIPROM_PARAM_BASE);
	switch (boot_mode) {
	case BOOT_USB_DOWNLOAD:
	case BOOT_UART_DOWNLOAD:
		ret = NS_BL1U_IMAGE_ID;
		break;
	default:
		WARN("Invalid boot mode is found:%d\n", boot_mode);
		panic();
	}
	return ret;
}
```

#### 2.3.4.2  bl2 image load

```c
desc = bl1_plat_get_image_desc(BL2_IMAGE_ID);                                          （1）
info = &desc->image_info;
err = bl1_plat_handle_pre_image_load(BL2_IMAGE_ID);                                     （2）
if (err != 0) {
	ERROR("Failure in pre image load handling of BL2 (%d)\n", err);
	plat_error_handler(err);
}
err = load_auth_image(BL2_IMAGE_ID, info);                                              （3）
if (err != 0) {
	ERROR("Failed to load BL2 firmware.\n");
	plat_error_handler(err);
}
	err = bl1_plat_handle_post_image_load(BL2_IMAGE_ID);                                     （4）
if (err != 0) {
	ERROR("Failure in post image load handling of BL2 (%d)\n", err);
	plat_error_handler(err);
}
```

它主要包含以下几部分内容：

*   获取待加载镜像描述信息。在atf中，镜像描述信息主要包含镜像id、镜像加载器使用的信息image_info和镜像跳转时使用的信息ep_info，其结构如下：

    ![image-20220713153151555](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220713153151555.png)

*   bl1_plat_get_image_desc用于获取bl2镜像的信息；

*   加载之前的处理，它由平台函数bl1_plat_handle_pre_image_load处理，qemu平台未对其做任何处理

*   从storage中加载镜像，它会根据先前获取到的bl2镜像描述信息，从storage中将镜像数据加载到给定地址上。qemu支持fip和semihosting类型的加载方式

*   加载之后的处理，它主要用于设置bl1向bl2传递的参数，上面结构体中的args即用于该目的，它一共包括8个参数，在bl1跳转到bl2之前会分别被设置到x0 – x7寄存器中。bl1只需通过x1寄存器向bl2传送其可用的secure内存region即可。以下为其代码主体流程：

    ```c
    image_desc = bl1_plat_get_image_desc(BL2_IMAGE_ID);
    ep_info = &image_desc->ep_info;                                        （a）
    bl1_secram_layout = bl1_plat_sec_mem_layout();                            （b）
    bl2_secram_layout = (meminfo_t *) bl1_secram_layout->total_base;
    bl1_calc_bl2_mem_layout(bl1_secram_layout, bl2_secram_layout);              （c）
    ep_info->args.arg1 = (uintptr_t)bl2_secram_layout;                           （d）
    ```

    >a 获取bl2的ep信息
    >b 获取bl1的secure内存region
    >c 将总的内存减去bl1已使用的sram内存，作为bl2的可用内存
    >d 将bl2的可用内存信息保存到参数传递信息中

#### 2.3.4.3 next stage image preparing

在atf中定义了一个异常等级切换相关的cpu context结构体，该结构体包含了切换时所需的所有的信息，如gp寄存器的值，el1、el2系统寄存器以及el3状态的值等。由于armv8包含secure和non secure两种安全状态，因此在el3中为这两种状态分别保留了一份独立的上下文信息，我们在执行上下文切换准备工作时，实际上就是填充对应security状态的结构体内容。以下是该结构体的定义：

```c
typedef struct cpu_context {
	gp_regs_t gpregs_ctx;
	el3_state_t el3state_ctx;
	el1_sysregs_t el1_sysregs_ctx;
#if CTX_INCLUDE_EL2_REGS
	el2_sysregs_t el2_sysregs_ctx;
#endif
#if CTX_INCLUDE_FPREGS
	fp_regs_t fpregs_ctx;
#endif
	cve_2018_3639_t cve_2018_3639_ctx;
#if CTX_INCLUDE_PAUTH_REGS
	pauth_t pauth_ctx;
#endif
} cpu_context_t;
```

bl1_prepare_next_image的主要工作就是初始化primary cpu的cpu_context上下文，并填充该结构体的相关信息，其主要流程如下：

```c
desc = bl1_plat_get_image_desc(image_id);
	next_bl_ep = &desc->ep_info;                                                             （1）
	security_state = GET_SECURITY_STATE(next_bl_ep->h.attr);                                  （2）

	if (cm_get_context(security_state) == NULL)
		cm_set_context(&bl1_cpu_context[security_state], security_state);                             （3）

	if ((security_state != SECURE) && (el_implemented(2) != EL_IMPL_NONE)) {                     （4）
		mode = MODE_EL2;
	}
	next_bl_ep->spsr = (uint32_t)SPSR_64((uint64_t) mode,
		(uint64_t)MODE_SP_ELX, DISABLE_ALL_EXCEPTIONS);                                （5）

	bl1_plat_set_ep_info(image_id, next_bl_ep);
	cm_init_my_context(next_bl_ep);                                                            （6）
	cm_prepare_el3_exit(security_state);                                                          （7）
	desc->state = IMAGE_STATE_EXECUTED;
```

（1）获取bl2的ep信息
（2）从bl2的ep信息中获取其security状态
（3）若context内存未分配，则为其分配内存
（4）默认的下一阶段镜像异常等级为其支持的最高等级，即若支持el2，则下一异常等级为EL2
（5）计算spsr的值，即异常等级为step 4计算的值，栈指针使用sp_elx，关闭所有DAIF异常
（6）该函数为待切换异常等级初始化上下文，如scr_el3，scr_el3，pc，spsr以及参数传递寄存器x0 – x7的值
（7）将context中参数设置到实际的寄存器中

## 2.4 el3_exit

该函数执行实际的异常等级切换流程，包括设置scr_el3，spsr_el3，elr_el3寄存器，以及执行eret指令跳转到elr_el3设定的bl2入口函数处执行。其定义如下：

```assembly
func el3_exit
	mov	x17, sp                                                                    （1）
	msr	spsel, #MODE_SP_ELX                                                      （2）
	str	x17, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]                       （3）

	ldr	x18, [sp, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]
	ldp	x16, x17, [sp, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]                      （4）
	msr	scr_el3, x18
	msr	spsr_el3, x16
	msr	elr_el3, x17                                                                 （5）

#if IMAGE_BL31
	ldp	x19, x20, [sp, #CTX_EL3STATE_OFFSET + CTX_CPTR_EL3]
	msr	cptr_el3, x19

	ands	x19, x19, #CPTR_EZ_BIT
	beq	sve_not_enabled

	isb
	msr	S3_6_C1_C2_0, x20 /* zcr_el3 */
sve_not_enabled:
#endif

#if IMAGE_BL31 && DYNAMIC_WORKAROUND_CVE_2018_3639
	ldr	x17, [sp, #CTX_CVE_2018_3639_OFFSET + CTX_CVE_2018_3639_DISABLE]
	cbz	x17, 1f
	blr	x17
1:
#endif
	restore_ptw_el1_sys_regs

	bl	restore_gp_pmcr_pauth_regs                                                      （6）
	ldr	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]

#if IMAGE_BL31 && RAS_EXTENSION
	esb
#else
	dsb	sy
#endif
#ifdef IMAGE_BL31
	str	xzr, [sp, #CTX_EL3STATE_OFFSET + CTX_IS_IN_EL3]
#endif
	exception_return                                                                  （7）
endfunc el3_exit
```

（1）将sp_el0栈指针暂存到x17寄存器中
（2）将栈指针切换到sp_el3，其中sp_el3指向前面context的el3state_ctx指针，即它被用于保存el3的上下文
（3）将sp_el0的值保存的el3 context中
（4）从el3 context中加载scr_el3、spsr_el3和elr_el3寄存器的值
（5）设置scr_el3、spsr_el3和elr_el3寄存器
（6）恢复gp寄存器等寄存器的值
（7）执行eret指令，此后cpu将离开bl1跳转到bl2的入口处执行了

# 3. SoC BL2 Booting[^3]
在ARM的而每一个启动阶段都是单独的一个镜像，BL2 boot stage也一样。BL2启动流程类似于bl1，流程上比较简单，但需要加载更多的image，由于BL31和BL32是可选的配置，因此在BL2 stage 还需要注意配置选项，是否启动了BL31和BL32，还是直接引导BL33阶段。另外，还需要注意，BL2可以被在secure-EL1层级，也可以被配置为EL3层级，他们的入口函数和处理流程肯定会有一些区别，本章参考[^3]，选取常见的secure-EL1的方式，其总体执行流程：

![](https://raw.githubusercontent.com/carloscn/images/main/typorabl2_entrypoint_sig.svg)

## 3.1 bl2 init

在bl2 init流程包包含以下过程：
* 保存参数
* 异常设置
* 设置sctlr_el1寄存器
* 初始化bss段
* 栈的初始化

### 3.1.1 parameters 
```assembly
func bl2_entrypoint
	/*---------------------------------------------
	 * Save arguments x0 - x3 from BL1 for future
	 * use.
	 * ---------------------------------------------
	 */
	mov	x20, x0
	mov	x21, x1
	mov	x22, x2
	mov	x23, x3
```
BL1通过x0-x3四个通用寄存器传递参数过来，从这里可以看出，boot不同的阶段并没有对寄存器的值进行reset。bl1阶段实际上定义了x0~x7寄存器向bl2传递参数。
### 3.1.2 exception setting
```assembly
	/* ---------------------------------------------
	 * Set the exception vector to something sane.
	 * ---------------------------------------------
	 */
	adr	x0, early_exceptions
	msr	vbar_el1, x0
	isb

	/* ---------------------------------------------
	 * Enable the SError interrupt now that the
	 * exception vectors have been setup.
	 * ---------------------------------------------
	 */
	msr	daifclr, #DAIF_ABT_BIT
```
该流程是设定el1异常向量表的基地址，其定义位于 https://github.com/carloscn/arm-trusted-firmware/blob/master/common/aarch64/early_exceptions.S#L17 。从其定义可知，从定义上为空可以知道，不会对其进行实际的处理，而只是打印出相应的异常信息，然后设定系统为panic状态。

异常向量表设定完成之后，bl2使能serror和external abort异常，显然这些异常一般意味着系统出现未定义指令、空指针等严重错误，因此需要捕获异常将系统设定为安全状态。

### 3.1.3 set sctlr_el1 register
```assembly
	/* ---------------------------------------------
	 * Enable the instruction cache, stack pointer
	 * and data access alignment checks and disable
	 * speculative loads.
	 * ---------------------------------------------
	 */
	mov	x1, #(SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)
	mrs	x0, sctlr_el1
	orr	x0, x0, x1
	bic	x0, x0, #SCTLR_DSSBS_BIT
	msr	sctlr_el1, x0
	isb

	/* ---------------------------------------------
	 * Invalidate the RW memory used by the BL2
	 * image. This includes the data and NOBITS
	 * sections. This is done to safeguard against
	 * possible corruption of this memory by dirty
	 * cache lines in a system cache as a result of
	 * use by an earlier boot loader stage.
	 * ---------------------------------------------
	 */
	adr	x0, __RW_START__
	adr	x1, __RW_END__
	sub	x1, x1, x0
	bl	inv_dcache_range
```
这部分和bl1很像，配置如下：
* 使能i-cache
* 对齐检查
* 栈对齐检查
### 3.1.4 C env and stack
```assembly
	/* ---------------------------------------------
	 * Zero out NOBITS sections. There are 2 of them:
	 *   - the .bss section;
	 *   - the coherent memory section.
	 * ---------------------------------------------
	 */
	adrp	x0, __BSS_START__
	add	x0, x0, :lo12:__BSS_START__
	adrp	x1, __BSS_END__
	add	x1, x1, :lo12:__BSS_END__
	sub	x1, x1, x0
	bl	zeromem

#if USE_COHERENT_MEM
	adrp	x0, __COHERENT_RAM_START__
	add	x0, x0, :lo12:__COHERENT_RAM_START__
	adrp	x1, __COHERENT_RAM_END_UNALIGNED__
	add	x1, x1, :lo12:__COHERENT_RAM_END_UNALIGNED__
	sub	x1, x1, x0
	bl	zeromem
#endif

	/* --------------------------------------------
	 * Allocate a stack whose memory will be marked
	 * as Normal-IS-WBWA when the MMU is enabled.
	 * There is no risk of reading stale stack
	 * memory after enabling the MMU as only the
	 * primary cpu is running at the moment.
	 * --------------------------------------------
	 */
	bl	plat_set_my_stack

	/* ---------------------------------------------
	 * Initialize the stack protector canary before
	 * any C code is called.
	 * ---------------------------------------------
	 */
#if STACK_PROTECTOR_ENABLED
	bl	update_stack_protector_canary
#endif
```
C语言运行环境是有要求的，我们需要注意的是：
* 对于bss段的初始化，对这段空间的空间分配和清零
* 如果使用coherent mem区域[^15]，需要对这个区域清零 (coherent memory是对任何硬件GPU或者DMA可见的内存地址，一旦有一方对这个区域进行了写操作，所有观察者都可以看见)
* 初始化stack区域（设定栈指针寄存器，并把栈指针存储在SP里面）
* 设定canary值 （checksec linux工具可以查看是否开启了canary）

## 3.2 bl2 platform setup
它主要执行参数处理和平台初始化流程，后面的分析我们仍然以qemu平台为例。参数处理流程如下（plat/qemu/common/qemu_bl2_setup.c）：
```c
meminfo_t *mem_layout = (void *)arg1;               （1）
qemu_console_init();                            	（2）
bl2_tzram_layout = *mem_layout;                     （3）
plat_qemu_io_setup();                               （4）
```
* （1）从x1参数中获取bl2的内存layout信息
* （2）初始化串口控制台
* （3）设置bl2的内存layout信息
* （4）初始化qemu的storage加载驱动
qemu平台初始化主要是为bl2内存建立MMU页表，并启动MMU和dcache，其主要目的是加快后面镜像加载的速度。其代码如下：
```c
	QEMU_CONFIGURE_BL2_MMU(bl2_tzram_layout.total_base,
			      bl2_tzram_layout.total_size,
			      BL_CODE_BASE, BL_CODE_END,
			      BL_RO_DATA_BASE, BL_RO_DATA_END,
			      BL_COHERENT_RAM_BASE, BL_COHERENT_RAM_END);
```

## 3.3 bl2 image load
每一个boot stage实际上都有个image load，这个地方正好是我们secure boot运行的入口。

### 3.3.1 before  load

在镜像加载前需要做一些平台的准备，因为C语言环境已经准备完成，这部分已经可以运行C语言，所以你看到的都是C代码。aarch64架构下使能fp和simd寄存器访问权限，其代码如下：
```C
 write_cpacr(CPACR_EL1_FPEN(CPACR_EL1_FP_TRAP_NONE));
```
剩下就是对于secure boot和加密引擎的初始化。auth_mod_init

### 3.3.2 behind load
Bl2需要加载的镜像信息由平台定义，对于qemu平台其定义位于plat/qemu/common/qemu_bl2_mem_params_desc.c中，以下代码选取了其在aarch64架构下只加载bl31和bl33的典型配置：
```c
static bl_mem_params_node_t bl2_mem_params_descs[] = {
#ifdef __aarch64__
	{ .image_id = BL31_IMAGE_ID,

	  SET_STATIC_PARAM_HEAD(ep_info, PARAM_EP, VERSION_2,
				entry_point_info_t,
				SECURE | EXECUTABLE | EP_FIRST_EXE),
	  .ep_info.pc = BL31_BASE,
	  .ep_info.spsr = SPSR_64(MODE_EL3, MODE_SP_ELX,
				  DISABLE_ALL_EXCEPTIONS),
# if DEBUG
	  .ep_info.args.arg1 = QEMU_BL31_PLAT_PARAM_VAL,
# endif
	  SET_STATIC_PARAM_HEAD(image_info, PARAM_EP, VERSION_2, image_info_t,
				IMAGE_ATTRIB_PLAT_SETUP),
	  .image_info.image_base = BL31_BASE,
	  .image_info.image_max_size = BL31_LIMIT - BL31_BASE,

# ifdef QEMU_LOAD_BL32
	  .next_handoff_image_id = BL32_IMAGE_ID,
# else
	  .next_handoff_image_id = BL33_IMAGE_ID,
# endif
	},
#endif /* __aarch64__ */
	{ .image_id = BL33_IMAGE_ID,
	  SET_STATIC_PARAM_HEAD(ep_info, PARAM_EP, VERSION_2,
				entry_point_info_t, NON_SECURE | EXECUTABLE),
# ifdef PRELOADED_BL33_BASE
	  .ep_info.pc = PRELOADED_BL33_BASE,

	  SET_STATIC_PARAM_HEAD(image_info, PARAM_EP, VERSION_2, image_info_t,
				IMAGE_ATTRIB_SKIP_LOADING),
# else /* PRELOADED_BL33_BASE */
	  .ep_info.pc = NS_IMAGE_OFFSET,

	  SET_STATIC_PARAM_HEAD(image_info, PARAM_EP, VERSION_2, image_info_t,
				0),
	  .image_info.image_base = NS_IMAGE_OFFSET,
	  .image_info.image_max_size = NS_IMAGE_MAX_SIZE,
# endif /* !PRELOADED_BL33_BASE */

	  .next_handoff_image_id = INVALID_IMAGE_ID,
	}
};
```
该结构和bl1镜像描述比较类似，只是多了下一个阶段的镜像id，以及加载参数链表的节点信息。该结构体定义完成后需要通过以下接口注册到系统中：
REGISTER_BL_IMAGE_DESCS(bl2_mem_params_descs)
 在启动阶段，可通过plat_get_bl_image_load_info获取以上镜像加载信息，此后启动代码将遍历这些接在信息，并分别执行以下流程分别加载和处理这些镜像
（1）bl2_platform_setup：该函数用于bl2平台相关的设置，如security设置，timer设置以及dtb设置等
（2）bl2_plat_handle_pre_image_load：镜像加载前平台可以执行一些其特定的流程
（3）load_auth_image：该接口用于实际的镜像加载流程，其与bl1的镜像加载流程完全一样
（4）bl2_plat_handle_post_image_load：该接口用于设置镜像加载相关信息，qemu平台代码如下（plat/qemu/common/qemu_bl2_setup.c）：
```c
static int qemu_bl2_handle_post_image_load(unsigned int image_id)
{
	int err = 0;
	bl_mem_params_node_t *bl_mem_params = get_bl_mem_params_node(image_id);
	…
	switch (image_id) {
	case BL32_IMAGE_ID:
        …                                                                               （a）
	case BL33_IMAGE_ID:
#if ARM_LINUX_KERNEL_AS_BL33
		bl_mem_params->ep_info.args.arg0 =
			(u_register_t)ARM_PRELOADED_DTB_BASE;
		bl_mem_params->ep_info.args.arg1 = 0U;
		bl_mem_params->ep_info.args.arg2 = 0U;
		bl_mem_params->ep_info.args.arg3 = 0U;                                          （b）
#else
		bl_mem_params->ep_info.args.arg0 = 0xffff & read_mpidr();                       （c）
#endif
		bl_mem_params->ep_info.spsr = qemu_get_spsr_for_bl33_entry();                   （d）
		break;
	default:
		break;
	}

	return err;
}

```
a bl32用于加载trust os，在启动流程中不是必须的，此处暂时不讨论
b 若由bl2直接启动linux，则设置linux的启动参数。我们知道armv8架构的linux启动参数都是通过dtb传递的，因此这里将dtb地址设置为其启动参数
c 对于其它类型的bl33（如uboot），则将当前处理器的affinity信息作为其启动参数
d 设置bl33的spsr
### 3.3.3 params settings

bl2可能会加载bl31、bl32、bl33镜像，因此其需要将这些被加载镜像的信息传给下一阶段。Bl2通过链表方式来组织这些参数，其中每一级镜像是链表的一个节点，其具体结构如下图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-photo-size-5-6233049732335382981-x.jpg)

bl2若运行在S-EL1下，则镜像加载完成并准备好参数后，需要通过smc异常再次进入bl1，由bl1的smc处理函数来执行实际的镜像切换流程。在执行smc命令之前，我们需要为其设置好参数，上面的bl_params->head->ep_info会设置为smc的调用参数，同时bl_params还需要被设置为第一级镜像的arg0参数，即在启动第一级镜像（如bl31）时，通过其x0寄存器传给它的也是bl_params指针，从而使bl31可以继续启动其后的镜像。

由于bl_params位于sram内存中，而bl2开启了dcache，因此在跳转到smc之前，需要将这部分数据从cache刷到sram中。最后我们就可以调用下面的smc指令返回bl1了 :

`smc(BL1_SMC_RUN_IMAGE, (unsigned long)next_bl_ep_info, 0, 0, 0, 0, 0, 0);`

## 3.4 bl1 handle next-image
bl1的smc处理流程如下：
```c
vector_entry SynchronousExceptionA64                -->
smc_handler64 
```
其中smc_handler64会判断bl2传入的命令，若命令为BL1_SMC_RUN_IMAGE，则从x1寄存器中获取下一阶段镜像的ep_info，执行上下文切换的准备，并最终跳转到下一阶段镜像的入口执行。其代码流程如下：
```assembly
func smc_handler64
	mov	x30, #BL1_SMC_RUN_IMAGE
	cmp	x30, x0                                                               
	b.ne	smc_handler                                                      （1）

	mrs	x30, scr_el3
	tst	x30, #SCR_NS_BIT                                                     （2）
	b.ne	unexpected_sync_exception

	ldr	x30, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	msr	spsel, #MODE_SP_EL0
	mov	sp, x30                                                              （3）

	mov	x20, x1                                                              （4）

	mov	x0, x20
	bl	bl1_print_next_bl_ep_info                                            （5）

	ldp	x0, x1, [x20, #ENTRY_POINT_INFO_PC_OFFSET]                         
	msr	elr_el3, x0
	msr	spsr_el3, x1                                                         （6）
	ubfx	x0, x1, #MODE_EL_SHIFT, #2
	cmp	x0, #MODE_EL3                                                        （7）
	b.ne	unexpected_sync_exception
    …
	mov	x0, x20
	bl	bl1_plat_prepare_exit                                                （8）

	ldp	x6, x7, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x30)]
	ldp	x4, x5, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x20)]
	ldp	x2, x3, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x10)]
	ldp	x0, x1, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x0)]                  （9）
	exception_return                                                          （10）
endfunc smc_handler64
```
（1）判断通过x0寄存器传入的smc命令是否为BL1_SMC_RUN_IMAGE，若不是则执行smc_handler，否则继续执行下面的镜像跳转流程
（2）判断scr_el3的secure位是否为0。该值表示EL0 – EL2等级的secure状态，因此实际指的是smc跳转之前的执行状态。所以该值为0就表示smc跳转前bl2执行在secure状态，否则表示bl2执行在non secure状态。在atf架构中，为了系统的安全性bl2必须要运行在secure状态（因为通常在bl2需要执行一些secure相关的设置，如tzasc，tzma，tzpc等）
（3）从sp_el3栈中获取先前el3_exit流程中保存的运行时栈的值，将其恢复回sp_el0，并将栈指针切换回sp_el0
（4）获取smc指令通过x1寄存器传入的next_bl_ep_info参数
（5）打印参数信息
（6）从next_bl_ep_info参数中获取bl1的入口地址和spsr寄存器值，分别将其设置到elr_el3和spsr_el3，为跳转到下一阶段镜像做准备
（7）判断下一阶段镜像是否运行于el3，若不是则出错
（8）平台相关的跳转前自定义流程，该函数默认什么都不做
（9）通过x0 – x7设置跳转参数，在bl2中只向arg0设置了bl_params指针这一个参数，因此bl2传给bl31的参数为描述镜像信息的bl_params指针
（10）通过eret跳转到下一阶段镜像入口函数处执行
# 4. SoC BL31 Booting[^4]
bl31属于已经进入到runtime阶段，提供runtime阶段的一些服务（系统启动完成之后继续驻留在系统中，并处理来自其他异常等级的SMC异常），比如电源管理，arm架构服务和SoC服务，甚至是board级的服务。这部分有两种启动路径：
*	由BL31引导BL32(Trusted - OS)启动，再由Trusted OS返回到BL31启动BL33
*	还有一种是直接启动BL33（没有trusted os）

bl31启动流程主要包含以下工作：
*	CPU初始化
*	C运行环境初始化
*	基本硬件初始化，如GIC，串口，timer等
*	页表创建和cache使能
*	启动后级image以及新image的跳转
*	若bl3支持el3中断，则需要初始化中断处理框架
*	运行不同的secure状态的smc处理，以及异常等级切换上下文的初始化
*	用于处理smc命令的运行服务注册

![](https://raw.githubusercontent.com/carloscn/images/main/typorabl31_entrypoint.svg)

## 4.1 bl31 init
bl31的初始化包含：
* 保存参数（x0-x3来自于bl2的参数）
* common函数
* 做一些清理工作
### 4.1.1 params saving
```assembly
mov	x20, x0
mov	x21, x1
mov	x22, x2
mov	x23, x3
```
与bl1进入bl2相同，将bl2传递的参数从caller寄存器保存到callee的寄存器中。
## 4.2 bl31 common
```assembly
#if !RESET_TO_BL31
	/* ---------------------------------------------------------------------
	 * For !RESET_TO_BL31 systems, only the primary CPU ever reaches
	 * bl31_entrypoint() during the cold boot flow, so the cold/warm boot
	 * and primary/secondary CPU logic should not be executed in this case.
	 *
	 * Also, assume that the previous bootloader has already initialised the
	 * SCTLR_EL3, including the endianness, and has initialised the memory.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_init_sctlr=0					\
		_warm_boot_mailbox=0				\
		_secondary_cold_boot=0				\
		_init_memory=0					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions		\
		_pie_fixup_size=BL31_LIMIT - BL31_BASE
#else

	/* ---------------------------------------------------------------------
	 * For RESET_TO_BL31 systems which have a programmable reset address,
	 * bl31_entrypoint() is executed only on the cold boot path so we can
	 * skip the warm boot mailbox mechanism.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_init_sctlr=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions		\
		_pie_fixup_size=BL31_LIMIT - BL31_BASE
```
从上面的代码可知，根据是否设置了`RESET_TO_BL31`，该函数有两套不同的调用参数。这是因为ATF支持两种启动方式：
* （1）启动从bl1开始执行，这是atf默认的启动方式。此时由于bl1已经执行过el3_entrypoint_common函数，系统基本配置都已经设置完成。因此像设置sctlr寄存器、热启动跳转处理、secondary cpu处理，以及内存初始化流程在bl1中都已经完成，bl31中就可以跳过它们了
* （2）支持从bl31开始启动的基础是armv8支持动态设置cpu的重启地址，armv8架构提供了RVBAR（reset vector base address register）寄存器用于设置reset时cpu的启动位置。该寄存器一共有三个：RVBAR_EL1、RVBAR_EL2和RVBAR_EL3，根据系统实现的最高异常等级确定使用哪一个。我们知道armv8重启总是从最高异常等级开始执行，因此我们只需要设置最高异常等级的RVBAR寄存器即可。由于bl31运行在el3下，故若我们需要支持启动从bl31开始，就可通过将地址设置到RVBAR_EL3寄存器实现。

若启动从bl31开始，则由于它是第一级启动镜像，因此el3_entrypoint_common需要从头设置系统状态，因此该函数中的sctlr寄存器、启动跳转处理、secondary cpu处理，以及内存初始化流程等都需要执行。
虽然el3_entrypoint_common需要做的工作有点多，但这种方式直接跳过了bl1和bl2两级启动流程，相比于第一种方式其启动速度要更快，这也是它的最大优势。
最后这种方式将参数保存寄存器x20 – x23的值清零也非常好理解，因为此时bl31是启动的第一级镜像，自然就没有前级镜像传递的参数，此时将这些值清零可避免后面参数解析时出现问题。
## 4.3 bl31 main
### 4.3.1 bl31_early_platform_setup2
该函数先初始化qemu控制台，然后解析bl2传入的镜像描述链表参数，并将解析到的bl32和bl33镜像ep_info保存到全局变量中。其主要流程如下：
```C
qemu_console_init();                                 （1）
bl_params_t *params_from_bl2 = (bl_params_t *)arg0;  （2）
		…
bl_params_node_t *bl_params = params_from_bl2->head; （3）
while (bl_params) {                                  （4）
	if (bl_params->image_id == BL32_IMAGE_ID) {
		bl32_image_ep_info = *bl_params->ep_info;    （5）
	}

	if (bl_params->image_id == BL33_IMAGE_ID){
		bl33_image_ep_info = *bl_params->ep_info;    （6）
	}

	bl_params = bl_params->next_params_info;
}
if (!bl33_image_ep_info.pc)          （7）                                       
	panic();
```
（1）控制台初始化
（2）获取arg0传入的镜像描述参数指针
（3）获取镜像链表头节点
（4）遍历镜像链表
（5）若该链表中含有bl32镜像描述符，则将其ep_info保存到全局变量
（6）多该链表中含有bl33镜像描述符，同样将其ep_info保存到全局变量
（7）校验bl33镜像的入口地址
### 4.3.2 bl31_plat_arch_setup
该函数用于为bl31相关内存创建页表，并使能MMU和dcache，其代码如下：
```C
void bl31_plat_arch_setup(void)
{
	qemu_configure_mmu_el3(BL31_BASE, (BL31_END - BL31_BASE),
	BL_CODE_BASE, BL_CODE_END,
	BL_RO_DATA_BASE, BL_RO_DATA_END,
	BL_COHERENT_RAM_BASE, BL_COHERENT_RAM_END);
}
```
### 4.3.3 bl31_platform_setup
该函数是平台相关的，qemu平台的实现如下：
```C
void bl31_platform_setup(void)
{
	plat_qemu_gic_init();                （1）
	qemu_gpio_init();                    （2）
}
```
（1）初始化gic，包括gic的distributor，redistributor，cpu interface等的初始化。关于bl31 gic和中断处理的详细流程。
（2）初始化qemu平台的gpio，即为其设置gpio基地址和操作相关的回调函数
### 4.3.4 ehf init
ehf用于初始化el3中断处理相关的功能。在gicv3中中断被分为三个group：group0、secure group1和non secure group 1，它们根据scr_el3的irq和fiq位配置不同可分别路由到不同的异常等级处理。Ehf用于处理group0中断，这种中断总是以fiq形式触发，通过设置scr_el3将其路由到el3处理就可以在bl31中处理这种类型中断了。
ehf初始化流程主要就是设置group 0的路由方式，并为其设置一个总的中断处理函数。其主要流程如下：
```c
void __init ehf_init(void)
{
	unsigned int flags = 0;
	int ret __unused;
    …
	set_interrupt_rm_flag(flags, NON_SECURE);
	set_interrupt_rm_flag(flags, SECURE);                              （1）

	ret = register_interrupt_type_handler(INTR_TYPE_EL3,
			ehf_el3_interrupt_handler, flags);                 （2）
	assert(ret == 0);
}
```
（1）计算中断路由相关的flag
（2）设置EL3类型（group 0）中断的中断路由方式和bl31总的中断处理函数
bl31中断处理函数ehf_el3_interrupt_handler会由异常向量表处理流程调用，它会继续根据中断优先级调用实际每个优先级对应的处理函数。中断优先级对应处理函数的注册流程分为以下共有两步，以下是中断注册流程的示例：
```C
ehf_pri_desc_t plat_exceptions[] = {
#if RAS_EXTENSION
	EHF_PRI_DESC(PLAT_PRI_BITS, PLAT_RAS_PRI),
#endif
#if SDEI_SUPPORT
	EHF_PRI_DESC(PLAT_PRI_BITS, PLAT_SDEI_CRITICAL_PRI),
	EHF_PRI_DESC(PLAT_PRI_BITS, PLAT_SDEI_NORMAL_PRI),
#endif
#if SPM_MM
	EHF_PRI_DESC(PLAT_PRI_BITS, PLAT_SP_PRI),
#endif
#ifdef PLAT_EHF_DESC
	PLAT_EHF_DESC,
#endif
};

EHF_REGISTER_PRIORITIES(plat_exceptions, ARRAY_SIZE(plat_exceptions), PLAT_PRI_BITS);
```
上面的例子中注册了RAS、SDEI等中断，并为它们分配了不同的优先级，但是此时只是为中断处理函数占了一个位，而并未实际定义。它们实际上要在驱动中通过ehf_register_priority_handler注册。如对于sdei，其注册流程如下：
```C
void sdei_init(void)
{
	…
	ehf_register_priority_handler(PLAT_SDEI_CRITICAL_PRI,
			sdei_intr_handler);
	ehf_register_priority_handler(PLAT_SDEI_NORMAL_PRI,
			sdei_intr_handler);
}
```
当ehf_register_priority_handler注册完成后，理论上bl31就可以接收和处理el3中断了。但是实际上bl31正在执行时，PSTATE的irq和fiq中断掩码都是被mask掉的，即el3中断只有在cpu运行于低于EL3异常等级的时候才能真正被触发和处理。
### 4.3.5 运行时服务初始化
前面我们提到bl31在系统初始化完成后还需要驻留系统，并处理来自低异常等级的smc异常，其异常处理流程被称为运行时服务。Arm为它们的使用场景定义了一系列的规范，分别用于处理类型不同的任务，如cpu电源管理规范PSCI、代理non secure world处理中断的软件事件代理规范SDEI，以及用于trust os相关调用的SPD等。显然这些服务被使用之前，其服务处理函数需要先注册到bl31中，运行时服务初始化流程即是用于该目的。
在分析运行时服务初始化流程之前，我们先看下其注册方式。以下是其注册接口DECLARE_RT_SVC的定义：
```c
#define DECLARE_RT_SVC(_name, _start, _end, _type, _setup, _smch)	\
	static const rt_svc_desc_t __svc_desc_ ## _name			\                 （1）
		__section("rt_svc_descs") __used = {			\                 （2）
			.start_oen = (_start),				\
			.end_oen = (_end),				\
			.call_type = (_type),				\
			.name = #_name,					\
			.init = (_setup),				\
			.handle = (_smch)				\
		}
typedef struct rt_svc_desc {
    uint8_t start_oen;-------------------service内部启动oen
    uint8_t end_oen;---------------------service内部末尾oen
    uint8_t call_type;-------------------smc类型，是fast call还是standard call
    const char *name;--------------------service名称
    rt_svc_init_t init;------------------service初始化函数
    rt_svc_handle_t handle;--------------对应function id的调用函数
} rt_svc_desc_t;
```
该接口定义了一个结构体__svc_desc_ ## _name，并将其放到了一个特殊的段rt_svc_descs中。这段的定义位于链接脚本头文件include/common/bl_common.ld.h中，其定义如下：
```C
#define RT_SVC_DESCS                                    \
        . = ALIGN(STRUCT_ALIGN);                        \
        __RT_SVC_DESCS_START__ = .;                     \
        KEEP(*(rt_svc_descs))                           \
        __RT_SVC_DESCS_END__ = .;
```
即这些被注册的运行时服务结构体都被保存到以`__RT_SVC_DESCS_START__`开头，`__RT_SVC_DESCS_END__`结尾的rt_svc_descs段中。

因此若需要获取这些结构体指针，只需遍历这段地址就可以了。运行时服务初始化函数runtime_svc_init流即是如此，其定义如下：
```c
void __init runtime_svc_init(void)
{
	…
	rt_svc_descs = (rt_svc_desc_t *) RT_SVC_DESCS_START;                 （1）
	for (index = 0U; index < RT_SVC_DECS_NUM; index++) {                 （2）
		rt_svc_desc_t *service = &rt_svc_descs[index];

		rc = validate_rt_svc_desc(service);                  （3）
		if (rc != 0) {
			ERROR("Invalid runtime service descriptor %p\n",
				(void *) service);
			panic();
		}

		if (service->init != NULL) {            
			rc = service->init();                                 （4）
			if (rc != 0) {
				ERROR("Error initializing runtime service %s\n",
						service->name);
				continue;
			}
		}
		…
	}
}
```
（1）获取rt_svc_descs段的起始地址RT_SVC_DESCS_START
（2）遍历该段中所有已注册rt_svc_desc_t结构体相应的运行时服务
（3）校验运行时服务有效性
（4）调用该服务对应的初始化回调，该回调函数是在DECLARE_RT_SVC注册宏中通过参数_setup传入的。

## 4.4 bl32 start
Bl32主要用于运行trust os，它主要用来保护用户的敏感数据（如密码、指纹、人脸等），以及与其相关的功能模块，如加解密算法，ta的加载与执行，secure storage等。各个厂家的trust os实现都有所不同，但基本思路是类似的，下面分析中涉及到具体的trust os时，我们将选取开源框架optee为例。
启动流程中bl32运行流程如下：
```C
	if (bl32_init != NULL) {
		INFO("BL31: Initializing BL32\n");

		int32_t rc = (*bl32_init)();

		if (rc == 0)
			WARN("BL31: BL32 initialization failed\n");
	}
```
它首先判断bl32_init是否已注册，若已注册则通过调用该函数执行实际的bl32运行流程。我们先看下optee架构下bl32_init注册流程（services/spd/opteed）：
```c
DECLARE_RT_SVC(
	opteed_fast,

	OEN_TOS_START,
	OEN_TOS_END,
	SMC_TYPE_FAST,
	opteed_setup,                                                 （1）
	opteed_smc_handler
);
static int32_t opteed_setup(void)
{
	…
	bl31_register_bl32_init(&opteed_init)                          （2）
	return 0;
}
void bl31_register_bl32_init(int32_t (*func)(void))
{
	bl32_init = func;                                              （3）
}
```
（1）通过DECLARE_RT_SVC设置optee的初始化回调opteed_setup
（2）将opteed_init函数注册为bl32的启动函数
（3）实际的回调注册
因此optee的bl32启动函数为opteed_init，它的流程与我们先前bl1启动bl2的跳转方式类似，其流程图如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typoraopteed_init1.svg)

它先获取先前保存的secure镜像ep信息（即bl32的ep信息），然后用其初始化异常等级切换的上下文，设置secure el1的系统寄存器，spsr_el3和elr_el3等。然后调用opteed_enter_sp函数跳转到bl32。这里有个问题，bl31除了启动bl32后，还需要继续启动bl33，因此bl32启动完成后还需要跳转回bl31并继续执行bl33启动流程。由于bl32在secure EL1执行，其同步进入bl31只能使用smc方式，因此需要在smc处理流程中跳转到原先的断点处。Armv8中c语言的lr寄存器为x30，因此若我们在跳转之前保存x30及运行上下文，然后再smc处理流程中恢复这些上下文即可以实现恢复断点处执行了。以下为opteed_enter_sp函数的上下文保存流程：
```assembly
func opteed_enter_sp
	mov	x3, sp
	str	x3, [x0, #0]
	sub	sp, sp, #OPTEED_C_RT_CTX_SIZE

	stp	x19, x20, [sp, #OPTEED_C_RT_CTX_X19]
	stp	x21, x22, [sp, #OPTEED_C_RT_CTX_X21]
	stp	x23, x24, [sp, #OPTEED_C_RT_CTX_X23]
	stp	x25, x26, [sp, #OPTEED_C_RT_CTX_X25]
	stp	x27, x28, [sp, #OPTEED_C_RT_CTX_X27]
	stp	x29, x30, [sp, #OPTEED_C_RT_CTX_X29]

	b	el3_exit
endfunc opteed_enter_sp
```
在该函数中上下文会被保存到全局变量opteed_sp_context中，optee初始化完成后返回smc处理的流程如下（services/spd/opteed/opteed_main.c）：
```c
uintptr_t opteed_smc_handler(…)
{
optee_context_t *optee_ctx = &opteed_sp_context[linear_id];
    …
	switch (smc_fid) {
	case TEESMC_OPTEED_RETURN_ENTRY_DONE:                             （1）
		assert(optee_vector_table == NULL);
		optee_vector_table = (optee_vectors_t *) x1;
		…
		opteed_synchronous_sp_exit(optee_ctx, x1);                 （2）
		break;
    …
	}
}
```
（1）表明本次smc调用是bl32启动完成后返回
（2）调用该函数恢复进入bl32之前保存的上下文，返回断点处继续执行。该函数的定义如下：
```assembly
func opteed_exit_sp
	mov	sp, x0                                                                  （1）

	ldp	x19, x20, [x0, #(OPTEED_C_RT_CTX_X19 - OPTEED_C_RT_CTX_SIZE)]
	ldp	x21, x22, [x0, #(OPTEED_C_RT_CTX_X21 - OPTEED_C_RT_CTX_SIZE)]
	ldp	x23, x24, [x0, #(OPTEED_C_RT_CTX_X23 - OPTEED_C_RT_CTX_SIZE)]
	ldp	x25, x26, [x0, #(OPTEED_C_RT_CTX_X25 - OPTEED_C_RT_CTX_SIZE)]
	ldp	x27, x28, [x0, #(OPTEED_C_RT_CTX_X27 - OPTEED_C_RT_CTX_SIZE)]
	ldp	x29, x30, [x0, #(OPTEED_C_RT_CTX_X29 - OPTEED_C_RT_CTX_SIZE)]            （2）

	mov	x0, x1
	ret                                                                              （3）
endfunc opteed_exit_sp
```
（1）恢复进入bl32之前保存在context中的栈
（2）恢复进入bl32之前保存的callee寄存器
（3）返回断点处继续执行，兜兜转转一圈，我们好不容易又返回到bl31_main函数了
![](https://raw.githubusercontent.com/carloscn/images/main/typoraflowchar1_bl31_bl32.svg)

## 4.5 bl33 booting
bl33启动流程与前面各级镜像启动流程，类似，也是根据ep_info设置bl33的上下文、入口地址和参数，然后跳转到入口执行。大家有兴趣可以自行根据代码分析一下，这里不再赘述。好了，atf启动流程总算走完了，接下来我们将跳转到bl33（uboot）的世界，一切的准备都是为了uboot启动kernel那一刻的美好！
# 3.Ref

[^1]:[聊聊SOC启动（一）armv8启动总体流程 ](https://zhuanlan.zhihu.com/p/520039243)
[^2]:[聊聊SOC启动（二） ATF BL1启动流程](https://zhuanlan.zhihu.com/p/520039243)
[^3]:[聊聊SOC启动（三） ATF BL2启动流程](https://zhuanlan.zhihu.com/p/520039243)
[^4]:[聊聊SOC启动（四） ATF BL31启动流程](https://zhuanlan.zhihu.com/p/520052961)
[^5]:[聊聊SOC启动（五） uboot启动流程一](https://zhuanlan.zhihu.com/p/520060653)
[^6]:[聊聊SOC启动（六） uboot启动流程二](https://zhuanlan.zhihu.com/p/520087511)
[^7]:[聊聊SOC启动（七） SPL启动分析](https://zhuanlan.zhihu.com/p/520189611)
[^8]:[聊聊SOC启动（八） uboot启动流程三](https://zhuanlan.zhihu.com/p/520575102)
[^9]:[聊聊SOC启动（九） 为uboot 添加新的board](https://zhuanlan.zhihu.com/p/521069920)
[^10]:[聊聊SOC启动（十） 内核启动先导知识](https://zhuanlan.zhihu.com/p/522195519)
[^11]:[聊聊SOC启动（十一） 内核初始化](https://zhuanlan.zhihu.com/p/522991248)
[^12]:[ARM Trusted Firmware Design ](https://chromium.googlesource.com/external/github.com/ARM-software/arm-trusted-firmware/+/v1.4-rc0/docs/firmware-design.md#bl1-architectural-initialization)
[^13]:[SCTLR_EL3, System Control Register (EL3) ](https://developer.arm.com/documentation/ddi0601/2022-03/AArch64-Registers/SCTLR-EL3--System-Control-Register--EL3-?lang=en)
[^14]:[地址无关代码 ](https://baike.baidu.com/item/%E5%9C%B0%E5%9D%80%E6%97%A0%E5%85%B3%E4%BB%A3%E7%A0%81/22702477?fr=aladdin)
[^15]:[coherent memory](https://blog.csdn.net/denglin12315/article/details/120291393)



