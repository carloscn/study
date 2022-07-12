# 02_ARMv8 boot Flow

ARMv7和ARMv8在引导流程上面完全不同的思路。ARMv8要兼容secure boot，需要在不同的异常等级做相应的处理，而且还需要给SoC厂商一些可配的灵活度，所以在boot上会引入不同的概念，相应的，比ARMv7（及以前）设计层面的复杂度要高很多。

# 1. boot high-level

在ARMv8里面boot程序是有trustzone-firmware执行引导，参考https://github.com/carloscn/arm-trusted-firmware/里面设计boot armv8的所有流程。

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

# 2. SoC BL1 Booting

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



# 3. SoC BL2



# 4. SoC BL31



# 3.Ref

[^1]:[聊聊SOC启动（一）armv8启动总体流程 ](https://blog.csdn.net/lgjjeff/article/details/124786983?spm=1001.2014.3001.5502)
[^2]:[聊聊SOC启动（二） BL1启动流程](https://blog.csdn.net/lgjjeff/article/details/124853411?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-124853411-blog-124786983.pc_relevant_multi_platform_whitelistv1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-124853411-blog-124786983.pc_relevant_multi_platform_whitelistv1&utm_relevant_index=2)
[^3]:[聊聊SOC启动（三） BL2启动流程](https://blog.csdn.net/lgjjeff/article/details/124864918?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-124864918-blog-124853411.pc_relevant_multi_platform_whitelistv2&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-124864918-blog-124853411.pc_relevant_multi_platform_whitelistv2&utm_relevant_index=1)
[^4]:[聊聊SOC启动（四） BL31启动流程](https://blog.csdn.net/lgjjeff/article/details/124884825)
[^5]:[聊聊SOC启动（五） uboot启动流程一](https://blog.csdn.net/lgjjeff/article/details/124937777?spm=1001.2014.3001.5502)
[^6]:[聊聊SOC启动（六）uboot启动流程二](https://blog.csdn.net/lgjjeff/article/details/124967694?spm=1001.2014.3001.5502)
[^7]:[聊聊SOC启动（七） uboot启动流程三](https://blog.csdn.net/lgjjeff/article/details/124994184?spm=1001.2014.3001.5502)
[^8]:[聊聊SOC启动（八）spl启动流程](https://blog.csdn.net/lgjjeff/article/details/90702226?spm=1001.2014.3001.5502)
[^9]:[聊聊SOC启动（九） 为uboot 添加新的board](https://blog.csdn.net/lgjjeff/article/details/125013403?spm=1001.2014.3001.5502)
[^10]:[聊聊SOC启动（十） 内核启动先导知识](https://blog.csdn.net/lgjjeff/article/details/125052189?spm=1001.2014.3001.5502)
[^11]:[聊聊SOC启动（十一） 内核初始化](https://blog.csdn.net/lgjjeff/article/details/125081790?spm=1001.2014.3001.5502)
[^12]:[ARM Trusted Firmware Design ](https://chromium.googlesource.com/external/github.com/ARM-software/arm-trusted-firmware/+/v1.4-rc0/docs/firmware-design.md#bl1-architectural-initialization)
[^13]:[SCTLR_EL3, System Control Register (EL3) ](https://developer.arm.com/documentation/ddi0601/2022-03/AArch64-Registers/SCTLR-EL3--System-Control-Register--EL3-?lang=en)
[^14]:[地址无关代码 ](https://baike.baidu.com/item/%E5%9C%B0%E5%9D%80%E6%97%A0%E5%85%B3%E4%BB%A3%E7%A0%81/22702477?fr=aladdin)

