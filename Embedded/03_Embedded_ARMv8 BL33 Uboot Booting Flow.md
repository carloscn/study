# 03_Embedded_ARMv8 BL33 Uboot Booting Flow

ARM上电后会从BootROM（或XIP）启动第一段程序，BootROM/XIP固化着SoC厂商自己写的驱动程序，这部分代码除了一些ARM公司规定的必要启动流程，SoC厂商根据自己的SoC设计内置一些初始化的操作，我们把这一阶段称为`SoC BootROM booting`阶段。由于BootROM内部的程序没有办法重新烧写，因为在芯片的流片的之前，这部分程序必须要确定下来且经过质量测试，一旦流片这部分程序将无法重新烧写，因此是重中之重的一环。在一些Secure Boot设计中，这部分程序要固化初试的根信任信息。

SoC BootROM之后的阶段，有多种叫法，中文称作`第二级程序引导`，缩写为SPL，一些公司例如德州仪器也会将这个引导的名字称为MLO。引入SPL的原因是因为SRAM的空间比较小，我们需要在正式引导之前有一个加载的程序，可以将正式的引导导入到DDR中。在某些架构下从SPL延伸出TPL的概念，我们从整个boot的过程中也将其视为SPL阶段。SPL在uboot可以找到其身影，这部分代码SoC厂商会根据自己的平台编写好自己的程序，并合入到uboot的代码仓库中。对于OEM厂可以对这部分代码根据需求进行订制和修改。

ARMv8上提供了ATF固件，ATF固件可以cover整个boot流程。若我们在ARMv8上面使用安全feature，必然要使用ATF固件作为整个引导的主导程序。当然，强大uboot也会和ATF功能有所重叠，在不使用ATF情况下，uboot也可以挑起启动的重任，但是无法使用其安全feature。

SPL之后，将会引导OEM厂最熟悉的uboot或者其他品牌的引导程序。关于整个ARMv7/v8的非安全和安全启动，可以参考：#65[^2] 和 #61[^1] 

 本节的重点关注于ARMv8的BL33阶段的uboot booting过程:
 * uboot的初始化
 * uboot的驱动模型
 * uboot的board_init_x
 * uboot的内核映像的封装

# 1. uboot的初始化[^3]
在BL33阶段的uboot初始化部分很多功能化和SPL是共用的，如下图所示为uboot整体的流程，其中标注为绿色的是BL33 uboot特有的部分，其他没有标注的，可以参考( https://github.com/carloscn/blog/issues/61  2.2.2 ARMv8 uboot-spl analysising)[^1]

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typorauboot_init%20(BL33).svg" width="70%" />
</div>

该流程主要包含了以下部分：  
* **save_boot_params**保存上一级镜像传入的参数，该函数由平台自行定义
* 若支持pie则检查代码段是否为4k对齐（因为由于指令集中操作数长度的限制，adr等类型指令的寻址范围是需要4k对齐的）
* **pie_fixup为pie重定位全局地址相关的.rela.dyn段内容**
* reset_sctrl根据配置确定是否重设sctlr寄存器
* 为uboot设置异常向量表。spl和uboot异常向量表设置有以下不同：
	* spl在设置了配置选项**CONFIG_ARMV8_SPL_EXCEPTION_VECTORS**，则会为其设置异常向量表，否则不为其设置异常向量表 ；
	* uboot默认情况就会设置异常向量表，armv8的异常向量表格式如下：![](https://raw.githubusercontent.com/carloscn/images/main/typora20220722131859.png)
	根据中断触发时cpu正在运行的异常等级、使用的栈寄存器类型以及运行状态，armv8会跳转到不同的中断向量。**由于spl和uboot在启动流程中不会执行比当前更低异常等级的代码**，因此只需要实现当前异常等级下的8个异常向量即可。其对应的向量表定义在arch/arm/cpu/armv8/exceptions.S中。由于根据不同的配置，spl或uboot可运行在el1 – el3异常等级下，因此需要根据**当前实际的异常等级来选择异常向量表基地址寄存器**。
* 若配置了**COUNTER_FREQUENCY选项**，则根据当前正在运行的异常等级，确定是否要设置cpu的system counter的频率。由于system counter的频率是所有异常等级共享的，为了确保该频率不被随意修改，因此约定只有运行于最高异常等级时才允许修改该寄存器。
* 若设置了配置选项CONFIG_ARMV8_SET_SMPEN，则设置S3_1_c15_c2_1以使能cpu之间的数据一致性。
* apply_core_errata用于处理cpu的errata。
* lowlevel_init流程可参考spl启动分析。

## 1.1 SMP多核启动

soc在启动阶段除了一些特殊情况外（如为了加快启动速度，在bl2阶段通过并行加载方式同时加载bl31、bl32和bl33镜像），一般都没有并行化需求。因此只需要一个cpu执行启动流程即可，这个cpu被称为primary cpu，而其它的cpu则统一被称为secondary cpu。为了防止secondary cpu在启动阶段的执行，它们在启动时必须要被设置为一个特定的状态。当primary cpu完成操作系统初始化，调度系统开始工作后，就可以通过一定的机制启动secondary cpu。显然secondary cpu不再需要执行启动流程代码，而只需直接跳转到内核中执行即可。故其启动的关键是如何将内核入口地址告知secondary cpu，以使其能跳转到正确的执行位置[^5]。

随着aarch64架构电源管理需求的增加（如cpu热插拔、cpu idle等），arm设计了一套标准的电源管理接口协议psci。该协议可以支持所有cpu相关的电源管理接口，而且由于电源相关操作是系统的关键功能，为了防止其被攻击，该协议将底层相关的实现都放到了secure空间，从而可提高系统的安全性[^5]。

armv8的从cpu启动包含**psci和spintable**两种方式[^4]。spin-table是一个比较简单的启动方法，差不多意思就是单核启动，其他核睡去的顺序执行；而psci是一个相对比较复杂的启动方法，且psci方式需要由bl31处理。

### 1.1.1 spin-table [^7]
>**spin-table启动方法**
> 一个系统的启动的基本流程是先bootloader然后运行kernel。当对所有CPU上电后，那么所有的CPU都会从bootrom里面开始执行代码，为了防止并发的一些问题，有必要将除了primary cpu以外的cpu拦截下来。使boot的过程是顺序的，而不是并发的。
> 
> 在启动的过程中，bootloader中有一道栅栏，它拦住了除了cpu0外的其他cpu。**cpu0直接往下运行，进行设备初始化以及运行Linux Kernelk，而其他cpu则在栅栏外进入睡眠状态。**cpu0在初始化smp的时候，会在cpu-release-addr里面填入一个地址并唤醒其他 cpu。这时候，在睡眠的这个cpu接受到了信号，醒来的时候先检查下cpu-release-addr这个地址里面的数据是不是不等于0。如果不等于 0，意味着主cpu填入了地址，该它上场了。它就会直接填入该地址执行。

```assembly
#if defined(CONFIG_ARMV8_SPIN_TABLE) && !defined(CONFIG_SPL_BUILD)          （1）
	branch_if_master x0, x1, master_cpu                                 （2）
	b	spin_table_secondary_jump                                   （3）
#elif defined(CONFIG_ARMV8_MULTIENTRY)                                      （4）
	branch_if_master x0, x1, master_cpu                                 （5）
slave_cpu:                                                                     
	wfe                                                                 （6）
	ldr	x1, =CPU_RELEASE_ADDR                                       （7）
	ldr	x0, [x1]
	cbz	x0, slave_cpu                                               （8）
	br	x0                                                          （9）
#endif 
master_cpu:                                                                   
	bl	_main

@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@

ENTRY(spin_table_secondary_jump)
.globl spin_table_reserve_begin
spin_table_reserve_begin:
0:      wfe                                                  （a）
        ldr     x0, spin_table_cpu_release_addr              （b）
        cbz     x0, 0b                                       （c）
        br      x0                                           （d）
.globl spin_table_cpu_release_addr                             
        .align  3
spin_table_cpu_release_addr:                                 （e）
        .quad   0
.globl spin_table_reserve_end
spin_table_reserve_end:
ENDPROC(spin_table_secondary_jump)
```
* （1）若当前从cpu为spin table启动方式，且当前执行的是uboot时。则从cpu将通过wfe进入自旋状态，并等待内核向给定地址填入其启动入口函数，该流程如下：
	 * a 从cpu进入wfe睡眠模式  
	 * b 若该cpu被唤醒，则读取spin_table_cpu_release_addr的值  
	 * c 若内核未向该地址写入其启动的入口函数，则继续返回睡眠  
	 * d 否则，跳转到读取到的入口处开始从cpu的启动流程  
	 * e 定义保存从cpu入口函数的内存地址，该地址在uboot启动时会被填入设备树spintable节点的属性中。内核启动从cpu时，则通过向解析到地址写入入口函数，并唤醒secondary cpu，从而完成其启动
* （2）若当前cpu为主cpu，继续执行冷启动流程
* （3）若当前cpu为从cpu，则进入step 1的spin模式
* （4）若未配置spintable，则从cpu需要spin在一个系统预先定义的地址上，并等待uboot在合适的时机向该地址填入入口函数
* （5）若当前cpu为主cpu，则继续执行冷启动流程
* （6 - 9）该流程与spintable方式类似，也是cpu通过wfe进入睡眠模式，并在唤醒后查询给定地址的值是否已被填入。若被填入则跳转到入口函数开始执行，否则继续进入睡眠模式。

### 1.1.2 psci
除了spin-table的方式，还有psci可以供选择作为secondary cpu的启动[^7]。在arm的uboot源程序里面使用Kconfig的 `ARMV8_SPIN_TABLE` 标签可以使能`SPIN_TABLE`，如果禁止那么则是PSCI启动模式[^8]。PSCI方式依托于ARM64的PSCI架构，PSCI架构不仅仅用于启动，还用于提供的一套电源管理接口，只不过启动被包含在这个架构之中[^6]。本文重点在uboot启动流程，关于SMP多核启动流程，uboot部分和kernel部分有很强的关联，故我们将这部分放在LinuxKernel专题来讲解，请参考 [# 02_LinuxKernel_内核的启动（二）SMP多核处理器启动过程分析 #66](https://github.com/carloscn/blog/issues/66)

## 1.2 `_main`流程分析

### 1.2.1 gd及内存规划

在进入c语言之前，我们需要为其准备好运行环境，以及做好内存规划，这其中除了**栈和堆内存**之外，**还需要为gd结构体分配内存空间**。gd是uboot中的一个global_data类型全局变量，该变量包含了很多全局相关的参数，为各模块之间参数的传递和共享提供了方便。由于该变量在跳转到c流程之前就需要准备好，此时堆管理器尚未被初始化，所以其内存需要通过手工管理方式分配。以下为uboot内存规划相关代码：
```assembly
#if defined(CONFIG_TPL_BUILD) && defined(CONFIG_TPL_NEEDS_SEPARATE_STACK)
	ldr	x0, =(CONFIG_TPL_STACK)
#elif defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
	ldr	x0, =(CONFIG_SPL_STACK)
#elif defined(CONFIG_INIT_SP_RELATIVE)
#if CONFIG_POSITION_INDEPENDENT
	adrp	x0, __bss_start
	add	x0, x0, #:lo12:__bss_start
#else
	adr	x0, __bss_start
#endif
	add	x0, x0, #CONFIG_SYS_INIT_SP_BSS_OFFSET
#else
	ldr	x0, =(CONFIG_SYS_INIT_SP_ADDR)                         （1）
#endif
	bic	sp, x0, #0xf                                           （2）
	mov	x0, sp                                                  
	bl	board_init_f_alloc_reserve                             （3）
	mov	sp, x0                                                 （4）
	mov	x18, x0                                                （5）
	bl	board_init_f_init_reserve                              （6）
```

* （1）以上部分根据不同的配置情况获取uboot的初始栈地址
* （2）为了遵循ABI规范，栈地址需要16字节对齐，该指令将地址做对齐以后设置到栈指针寄存器中，以为系统设置运行栈
* （3）该函数为gd和early malloc分配内存，其代码如下：
	```C
	ulong board_init_f_alloc_reserve(ulong top)
	{
	#if CONFIG_VAL(SYS_MALLOC_F_LEN)
	        top -= CONFIG_VAL(SYS_MALLOC_F_LEN);                         （a）
	#endif
	        top = rounddown(top-sizeof(struct global_data), 16);         （b）
	        return top;
	}
	```
	* a 为早期堆管理器预留内存
	* b 为gd预留内存
* （4）将预留后的内存地址设置为新的栈地址，此时各部分的地址如下：
	<div align='center'>
	<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220722141855.png" width="40%" />
	</div>
* （5）将gd地址保存到x18寄存器中，其可用被于后续gd指针的获取
* （6）该流程主要用于初始化gd，和设置early malloc的堆管理器基地址，其代码如下：
	```C
	void board_init_f_init_reserve(ulong base)
	{
	        struct global_data *gd_ptr;
	
	        gd_ptr = (struct global_data *)base;
	        memset(gd_ptr, '\0', sizeof(*gd));                    （a）
	#if !defined(CONFIG_ARM)
	        arch_setup_gd(gd_ptr);                                 (b）
	#endif
	        if (CONFIG_IS_ENABLED(SYS_REPORT_STACK_F_USAGE))
	                board_init_f_init_stack_protection_addr(base); (c）
	        base += roundup(sizeof(struct global_data), 16);                               
	#if CONFIG_VAL(SYS_MALLOC_F_LEN)
	        gd->malloc_base = base;                               （d）
	#endif
	        if (CONFIG_IS_ENABLED(SYS_REPORT_STACK_F_USAGE))
	                board_init_f_init_stack_protection();         （e）
	}	
      #ifdef CONFIG_ARM64
      #define DECLARE_GLOBAL_DATA_PTR register volatile gd_t *gd asm ("x18")
      #else
      #define DECLARE_GLOBAL_DATA_PTR register volatile gd_t *gd asm ("r9")
      #endif
	```
	* a 获取gd指针，并清空gd结构体内存
	* b 该函数用于非arm架构的gd指针获取，armv8架构则通过前面设置的x18寄存器获取gd指针，其定义如下（arch/arm/include/asm/global_data.h)
	* c 该函数用于获取该栈溢出检测的地址
	* d 设置early malloc的基地址
	* e 初始化栈溢出检测的canary值，该值被设置为SYS_STACK_F_CHECK_BYTE

### 1.2.2 重定位

一般的启动流程会由spl初始化ddr，**然后将uboot加载到ddr中运行**。但这并不是必须的，uboot自身其实也可以作为bl1或bl2启动镜像，此时uboot最初的启动位置不是位于ddr中（如norflash）。由于norflash的执行速度比ddr要慢的多，因此在完成ddr初始化后就需要将其搬移到ddr中，并切换到新的位置继续执行，这个流程就叫做uboot的重定位。

#### 1.2.2.1 重定位的前提

uboot重定位依赖于位置无关代码技术，因此需要在编译和重定位时添加以下支持：
* （1）编译时添加-fpie选项
* （2）**在链接时添加-pie选项，它使得链接器会产生.rel.dyn和.dynsym段的fixup表**。
* （3）链接脚本中添加.rel.dyn和.dynsym段定义，并为重定位代码访问这些段的数据提供符号信息
* （4）在重定位过程中需要根据新的地址fixup .rel.dyn和.dynsym段的数据

#### 1.2.2.2 重定位基本流程

由于**内核需要从内存的低地址开始运行**，为了防止内核三件套（**kernel、dtb和ramdisk**）的加载地址与uboot运行地址重叠，因此**uboot的重定位地址需要被设置到内存顶端附近**。同时我们还需要为一些特定模块预留一些内存空间（比如页表空间、framebuffer等），下图就是uboot规划的重定位后内存布局：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220722143055.png" width="80%" />
</div>

该图中橙色部分都是需要执行重定位操作的，如uboot的代码段、数据段，以及gd、设备树等，它们都是在board_init_r阶段还需要使用的。对于gd和dtb等纯数据的重定位，只需要将数据拷贝到新的地址，并将其基地址指针切换到新地址即可。但对于代码段的重定位我们还需要考虑以下问题：
* （1）位置无关代码需要调整.rel.dyn和.dynsym段
* （2）栈指针需要切换到新的位置
* （3）重定位完成后如何完成pc的平滑切换

**Note**：
* armv8代码重定位准备相关的源码，其位于arch/arm/lib/crt0_64.S中 （ https://github.com/ARM-software/u-boot/blob/v2018.09/arch/arm/lib/crt0_64.S#L92 ）
* armv8的代码重定位流程位于arch/arm/lib/relocate_64.S中 （ https://github.com/ARM-software/u-boot/blob/v2018.09/arch/arm/lib/relocate_64.S#L22 ）

### 1.2.3 board_init_f

board_init_f是uboot重定位前的流程，它包括一些基础模块的初始化和重定位相关的准备工作。以下为该函数在armv8架构下可能的执行流程，图中灰色框表示该流程是可配置的，黄色框表示是必选的。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220722152308.png)

### 1.2.4 board_init_r

board_init_r是uboot重定位后需要执行的流程，它包含基础模块、硬件驱动以及板级特性等的初始化，并最终通过run_main_loop启动os会进入命令行窗口。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220722153140.png)


# 2. uboot的驱动模型

uboot在初始化完成后会为用户提供一个**命令行交互接口**，用户可通过该接口执行uboot定义的命令，以用于查看系统状态，设置环境变量和系统参数等。为了方便对硬件和驱动的管理，uboot还**引入了类似linux内核的设备树和驱动模型特性**。当然，为了增加系统的可配置性、可调试性以及可跟踪性等，它还支持**环境变量、log管理、bootstage统计以及简单的ftrace等功能**。下面我们将对这些特性做一简单的介绍。

## 2.1 设备树

设备树是一种通过**dts文件来描述SoC属性**，通过将设备的具体配置信息与驱动分离，以达到利用一份代码适配多款设备的机制。dts文件包含了一系列层次化结构的节点和属性，它可以通过dtc编译器编译成适合设备解析的二进制dtb文件。

uboot设备树的使用包含以下流程：为**目标板添加dts文件、选择一个运行时使用的dtb文件、使能设备树**。

 **如何为目标板添加一个dts文件**

在`arch/<arch>/dts`目录下，添加一个xxx.dts文件，该文件可以从内核拷贝，或者在uboot dts目录下选择一个其它目标板的dts为基础，再根据实际需求进行修改。修改完成后，在arch/arm/dts/Makefile中为其添加编译选项：

````text
dtb-$(CONFIG_yyy) +=xxx.dtb
````

其中yyy为使用该dts的目标板

**如何为目标板选择dts文件** ？

uboot的设备树文件位于`arch/<arch>/dts`目录下，可通过以下选项为目标板选择一个默认的dts文件：

```text
CONFIG_DEFAULT_DEVICE_TREE="xxx”
```

这是因为与内核不一样，uboot最终的镜像会和dtb打包在一个镜像文件中，因此在编译流程中就需要知道最终被使用的dtb。关于uboot镜像与dtb之间的关系将在 3. uboot的kernel封装 介绍。

**通过编译命令指定dts**

有时在编译时希望使用一个不是默认指定的dts，则可以通过在编译命令中添加DEVICE_TREE=zzz方式指定新的dts文件，其示例如下：

```text
make DEVICE_TREE=zzz
```

**如何使能设备树**

通过配置CONFIG_OF_CONTROL选项即可使能设备树的支持，uboot与dtb可以有以下几种打包组合方式：

* （1）若定义了CONFIG_OF_EMBED选项，则在链接时会为dtb指定一个以__dtb_dt_begin开头的单独的段，dtb的内容将被直接链接到uboot.bin镜像中。官方建议这种方式只在开发和调试阶段使用，而不要用于生产阶段
* （2）若定义了CONFIG_OF_SEPARATE选项，dtb将会被编译为u-boot.dtb文件，而uboot原始镜像被编译为u-boot-nodtb.bin文件，并通过以下命令将它们连接为最终的uboot.bin文件，`cat u-boot-nodtb.bin u-boot.dtb >uboot.bin`

## 2.2 驱动模型DM

Uboot根据Linux的驱动模型架构，也引入了Uboot的驱动模型（**driver model ：DM**）。Uboot驱动模型与**linux的设备模型**比较类似，利用它可以将**设备与驱动分离**。对上可以为同一类设备提供统一的操作接口，对下可以为驱动提供标准的注册接口，从而提高代码的可重用性和可移植性。同时，驱动模型通过树形结构组织uboot中的所有设备，为系统对设备的统一管理提供了方便。

### 2.2.1 驱动模型结构

#### driver 结构体
驱动模型主要用于管理系统中的驱动和设备，uboot为它们提供了以下描述结构体：

driver结构体用于表示一个驱动，其定义如下:

```c
struct driver {
	char *name;
	enum uclass_id id;
	const struct udevice_id *of_match;
	int (*bind)(struct udevice *dev);
	int (*probe)(struct udevice *dev);
	int (*remove)(struct udevice *dev);
	int (*unbind)(struct udevice *dev);
	int (*of_to_plat)(struct udevice *dev);
	int (*child_post_bind)(struct udevice *dev);
	int (*child_pre_probe)(struct udevice *dev);
	int (*child_post_remove)(struct udevice *dev);
	int priv_auto;
	int plat_auto;
	int per_child_auto;
	int per_child_plat_auto;
	const void *ops;	/* driver-specific operations */
	uint32_t flags;
#if CONFIG_IS_ENABLED(ACPIGEN)
	struct acpi_ops *acpi_ops;
#endif
}
```

驱动可以通过以下接口注册到系统中：

```c

#define U_BOOT_DRIVER(__name)						\
	ll_entry_declare(struct driver, __name, driver)

// 其中ll_entry_declare的定义如下：

#define ll_entry_declare(_type, _name, _list)				\
	_type _u_boot_list_2_##_list##_2_##_name __aligned(4)		\
			__attribute__((unused))				\
			__section(".u_boot_list_2_"#_list"_2_"#_name)
```

即其会定义一个struct driver 类型的`_u_boot_list_2_driver_2_#_name`变量，该变量在链接时需要被放在`u_boot_list_2_driver_2_#_name`段中。我们再看下这些section在链接脚本中是如何存放的，以下为armv8架构链接脚本arch/arm/cpu/armv8/u-boot.lds中的定义：
```text
 .u_boot_list : {
     KEEP(*(SORT(.u_boot_list*)));
 }
```
从定义可看到这些以.u_boot_list 开头的section都会被保存在一起，且它们会按照section的名字排序后再保存。这主要是为了便于遍历这些结构体，如我们需要遍历所有已经注册的driver，则可通过以下代码获取driver结构体的起始地址和总的driver数量。

```c
struct driver *drv = ll_entry_start(struct driver, driver); // 获取已注册driver的起始地址
int n_ents = ll_entry_count(struct driver, driver); // 获取已注册driver的数量
```
其中ll_entry_start和ll_entry_coun的定义如下：
```c
#define ll_entry_start(_type, _list)					\
({									\
	static char start[0] __aligned(CONFIG_LINKER_LIST_ALIGN)	\
		__attribute__((unused))					\
		__section(".u_boot_list_2_"#_list"_1");			\              （1-3）
	(_type *)&start;						\
})

#define ll_entry_end(_type, _list)					\
({									\
	static char end[0] __aligned(4) __attribute__((unused))		\
		__section(".u_boot_list_2_"#_list"_3");			\               （1-4）
	(_type *)&end;							\
})

#define ll_entry_count(_type, _list)					\
	({								\
		_type *start = ll_entry_start(_type, _list);		\               
		_type *end = ll_entry_end(_type, _list);		\                   
		unsigned int _ll_result = end - start;			\               （1-5）
		_ll_result;						\
	})
```
（1-3）定义一个`.u_boot_list_2_"#_list"_1`的段，若需要遍历driver，则该段的名字为`.u_boot_list_2_driver_1`，即它位于所有实际driver section之前的位置；
（1-4）定义一个`.u_boot_list_2_"#_list"_3`的段，若需要遍历driver，则该段的名字为`.u_boot_list_2_driver_3`，即它位于所有实际driver section之后的位置；
（1-5）通过以上两个标号就可以很方便地获取驱动的起止地址和计算已注册驱动的总数。

#### uclass_driver结构体

uclass_driver结构体用于表示一个uclass驱动，其定义如下：
```c
struct uclass_driver {
	const char *name;
	enum uclass_id id;
	int (*post_bind)(struct udevice *dev);
	int (*pre_unbind)(struct udevice *dev);
	int (*pre_probe)(struct udevice *dev);
	int (*post_probe)(struct udevice *dev);
	int (*pre_remove)(struct udevice *dev);
	int (*child_post_bind)(struct udevice *dev);
	int (*child_pre_probe)(struct udevice *dev);
	int (*child_post_probe)(struct udevice *dev);
	int (*init)(struct uclass *class);
	int (*destroy)(struct uclass *class);
	int priv_auto;
	int per_device_auto;
	int per_device_plat_auto;
	int per_child_auto;
	int per_child_plat_auto;
	uint32_t flags;
};	
```

其注册和遍历方式与driver完全相同，只是结构体类型和section名有所不同，以下为其定义：

```c
#define UCLASS_DRIVER(__name)						\
	ll_entry_declare(struct uclass_driver, __name, uclass_driver)
```

#### udevice结构体
udevice在驱动模型中用于表示一个与驱动绑定的设备，其定义如下：
```c
struct udevice {
	const struct driver *driver;
	const char *name;
	void *plat_;
	void *parent_plat_;
	void *uclass_plat_;
	ulong driver_data;
	struct udevice *parent;
	void *priv_;
	struct uclass *uclass;
	void *uclass_priv_;
	void *parent_priv_;
	struct list_head uclass_node;
	struct list_head child_head;
	struct list_head sibling_node;
#if !CONFIG_IS_ENABLED(OF_PLATDATA_RT)
	u32 flags_;
#endif
	int seq_;
#if !CONFIG_IS_ENABLED(OF_PLATDATA)
	ofnode node_;
#endif
#ifdef CONFIG_DEVRES
	struct list_head devres_head;
#endif
#if CONFIG_IS_ENABLED(DM_DMA)
	ulong dma_offset;
#endif
} 
```

udevice在驱动模型中用于表示一个与驱动绑定的设备，其定义如下：

```c
struct udevice {
	const struct driver *driver;
	const char *name;
	void *plat_;
	void *parent_plat_;
	void *uclass_plat_;
	ulong driver_data;
	struct udevice *parent;
	void *priv_;
	struct uclass *uclass;
	void *uclass_priv_;
	void *parent_priv_;
	struct list_head uclass_node;
	struct list_head child_head;
	struct list_head sibling_node;
#if !CONFIG_IS_ENABLED(OF_PLATDATA_RT)
	u32 flags_;
#endif
	int seq_;
#if !CONFIG_IS_ENABLED(OF_PLATDATA)
	ofnode node_;
#endif
#ifdef CONFIG_DEVRES
	struct list_head devres_head;
#endif
#if CONFIG_IS_ENABLED(DM_DMA)
	ulong dma_offset;
#endif
} 
```
系统中所有的udevice结构体可以通过parent、child_head和sibling_node连接在一起，并且最终挂到gd的dm_root节点上，这样我们就可以通过gd->dm_root遍历所有的udevice设备。下图是udevice的连接关系，其中每个节点的parent指向其父节点，sibling指向其兄弟节点，而child指向子节点。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220722154526.png" width="66%" />
</div>

由于每个udevice都属于一个uclass，因此除了被连接到gd->dm_root链表之外，udevice还会被挂到uclass的链表中。它们之间的连接关系将在下面介绍uclass时给出。udevice是在驱动模型初始化流程中根据扫描到的设备动态创建的，在uboot中实际的设备可以通过以下两种方式定义：  
* （3-1）devicetree方式：这种方式通过devicetree维护设备信息，uboot在驱动模型初始化时，通过解析设备树获取设备信息，并完成其与驱动等的绑定  
* （3-2）硬编码方式：这种方式可通过下面的宏定义一个设备：

```c
 #define U_BOOT_DRVINFO(__name)						\
		ll_entry_declare(struct driver_info, __name, driver_info)
```

#### uclass结构体

uclass用于表示一类具有相同功能的设备，从而可以为其抽象出统一的设备访问接口，方便其它模块对它的调用。以下为uclass的定义：
```c
struct uclass {
	void *priv_;
	struct uclass_driver *uc_drv;
	struct list_head dev_head;
	struct list_head sibling_node;
} 
```

uclass将所有属于该类的设备挂到其dev_head链表上，同时系统中所有的uclass又会被挂到一张全局链表gd->uclass_root上。

### 2.2.2 驱动模型初始化

驱动模型初始化主要完成udevice、driver以及ucalss等之间的绑定关系，其主要包含以下部分：
* （1）udevice与driver的绑定
* （2）udevice与uclass的绑定
* （3）uclass与uclass_driver的绑定

该流程通过dm_init_and_scan函数实现，它会分别扫描由U_BOOT_DRVINFO以及devicetree定义的设备，为它们分配udevice结构体，并完成其与driver和uclass之间的绑定关系等操作。需要注意的是该函数在board_init_f和board_init_r中都会被调用，其中board_init_f主要是为了解析重定位前需要使用的设备节点，这种类型节点在devicetree中会增加u-boot,dm-pre-reloc属性。

## 2.3 环境变量与命令行

### 2.3.1 环境变量

环境变量可以为uboot提供在运行时动态配置参数的能力，如在命令行通过修改环境变量bootargs可以改变内核的启动参数。它以env=value格式存储，其中每条环境变量之间以’\0’结尾。根据系统的配置参数，uboot在include/env_default.h中为系统定义了一份默认的环境变量：

```c
#ifdef DEFAULT_ENV_INSTANCE_EMBEDDED
env_t embedded_environment __UBOOT_ENV_SECTION__(environment) = {
#ifdef CONFIG_SYS_REDUNDAND_ENVIRONMENT
	1,		
#endif
	{
#elif defined(DEFAULT_ENV_INSTANCE_STATIC)
static char default_environment[] = {
#elif defined(DEFAULT_ENV_IS_RW)
uchar default_environment[] = {
#else
const uchar default_environment[] = {
#endif
#ifndef CONFIG_USE_DEFAULT_ENV_FILE
#ifdef	CONFIG_ENV_CALLBACK_LIST_DEFAULT
	ENV_CALLBACK_VAR "=" CONFIG_ENV_CALLBACK_LIST_DEFAULT "\0"
#endif
#ifdef	CONFIG_ENV_FLAGS_LIST_DEFAULT
	ENV_FLAGS_VAR "=" CONFIG_ENV_FLAGS_LIST_DEFAULT "\0"
#endif
#ifdef	CONFIG_USE_BOOTARGS
	"bootargs="	CONFIG_BOOTARGS			"\0"
#endif
#ifdef	CONFIG_BOOTCOMMAND
	"bootcmd="	CONFIG_BOOTCOMMAND		"\0"
#endif

…

#ifdef	CONFIG_EXTRA_ENV_SETTINGS
	CONFIG_EXTRA_ENV_SETTINGS
#endif
	"\0"
#else
#include "generated/defaultenv_autogenerated.h"
#endif
#ifdef DEFAULT_ENV_INSTANCE_EMBEDDED
	}
#endif
};
```

在该环境变量中，board可通过重新定义CONFIG_EXTRA_ENV_SETTINGS的值设置其自身的默认环境变量，如对于qemu平台，其定义位于include/configs/qemu-arm.h：

```c
#define CONFIG_EXTRA_ENV_SETTINGS \
	"fdt_high=0xffffffff\0" \
	"initrd_high=0xffffffff\0" \
	"fdt_addr=0x40000000\0" \
	"scriptaddr=0x40200000\0" \
	"pxefile_addr_r=0x40300000\0" \
	"kernel_addr_r=0x40400000\0" \
	"ramdisk_addr_r=0x44000000\0" \
		BOOTENV
```

环境变量被修改后可以保存到固定的存储介质上（如flash、mmc等），以便下一次启动后加载最新的值。Uboot通过U_BOOT_ENV_LOCATION宏定义环境变量的存储位置，例如对于mmc其定义如下（env/mmc.c）：

```c
U_BOOT_ENV_LOCATION(mmc) = {
	.location	= ENVL_MMC,
	ENV_NAME("MMC")
	.load		= env_mmc_load,
#ifndef CONFIG_SPL_BUILD
	.save		= env_save_ptr(env_mmc_save),
	.erase		= ENV_ERASE_PTR(env_mmc_erase)
#endif
}
```

环境变量在mmc中的具体存储位置可通过配置选项或devicetree设置，如对于mmc：
* devicetree方式可在/config节点中设置以下属性：
	* u-boot,mmc-env-partition：指定环境变量存储的分区，环境变量会被存储在在该分区的结尾处；
	* u-boot,mmc-env-offset：若未定义u-boot,mmc-env-partition属性，则该参数用于指定环境变量在mmc裸设备上的偏移；
	* u-boot,mmc-env-offset-redundant：指定备份环境变量在mmc设备上的偏移。
* 通过配置参数设置
	* CONFIG_ENV_OFFSET：与u-boot,mmc-env-offset含义相同；
	* CONFIG_ENV_OFFSET_REDUND：与u-boot,mmc-env-offset-redundant含义相同。

下面的选项用于配置环境变量的长度及其保存的设备：
* CONFIG_ENV_SIZE：环境变量的最大长度
* CONFIG_ENV_IS_IN_XXX（如CONFIG_ENV_IS_IN_MMC）：环境变量保存的设备类型
* CONFIG_SYS_MMC_ENV_DEV：环境变量保存的设备编号

uboot对保存在固定介质中的环境变量会使用crc32校验数据的完整性，若数据被破坏了则会使用默认环境变量重新初始化环境变量的值。

### 2.3.2 命令行

uboot在初始化完成后可以通过按键进入命令行窗口，在该窗口可以执行像设置环境变量，下载镜像文件，启动内核等命令，这些命令的支持大大方便了uboot和内核启动相关流程的调试。uboot提供了很多内置命令，如md、mw、setenv、saveenv、tftpboot、bootm等，uboot提供了以下宏用于命令定义（include/command.h）：

#### U_BOOT_CMD

它用于定义一个uboot命令，其定义如下：

```text
#define U_BOOT_CMD(_name, _maxargs, _rep, _cmd, _usage, _help)		\
		U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, NULL)
```
其中参数含义如下:
* _name：命令名
* _maxargs：最多参数个数
* _rep：命令是否可重复。cmd_rep回调会输出自身是否可重复（按下回车之后，上条被执行的命令再次被执行则是可重复的）
* _cmd：命令处理函数
* _usage：使用信息，执行help时显示的简短信息
* _help：帮助信息（执行help name时显示的详细使用信息）

#### U_BOOT_CMD_WITH_SUBCMDS

它用于定义一个带子命令的uboot命令，子命令可以避免主命令处理函数中包含过多的逻辑，还可以为每个子命令可以定义自身的_rep参数，以独立处理其是否可被重复执行的功能。

以下为其定义：
```C
#define U_BOOT_CMD_WITH_SUBCMDS(_name, _usage, _help, ...)		\
	U_BOOT_SUBCMDS(_name, __VA_ARGS__)				\
	U_BOOT_CMDREP_COMPLETE(_name, CONFIG_SYS_MAXARGS, do_##_name,	\
			       _usage, _help, complete_##_name)
```
其固定参数如下：
*  _name：主命令名
* _usage：使用信息，执行help时显示的简短信息
* _help：帮助信息（执行help name时显示的详细使用信息）

#### U_BOOT_SUBCMD_MKENT
可变参数部分可用于定义子命令U_BOOT_SUBCMD_MKENT，其定义如下：
```c
#define U_BOOT_SUBCMD_MKENT(_name, _maxargs, _rep, _do_cmd)		\
	U_BOOT_SUBCMD_MKENT_COMPLETE(_name, _maxargs, _rep, _do_cmd,	\
				     NULL)
```
子命令的参数如下：
* _name：子命令名
* _axargs：子命令的最大参数个数
* _rep：该子命令是否可重复执行
* _do_cmd：子命令的命令处理函数

以wdt命令为例（cmd/wdt.c），其定义了主命令wdt，并且定义了子命令list、dev、start等

```c
static char wdt_help_text[] =
	"list - list watchdog devices\n"
	"wdt dev [<name>] - get/set current watchdog device\n"
	"wdt start <timeout ms> [flags] - start watchdog timer\n"
	"wdt stop - stop watchdog timer\n"
	"wdt reset - reset watchdog timer\n"
	"wdt expire [flags] - expire watchdog timer immediately\n";

U_BOOT_CMD_WITH_SUBCMDS(wdt, "Watchdog sub-system", wdt_help_text,
	U_BOOT_SUBCMD_MKENT(list, 1, 1, do_wdt_list),
	U_BOOT_SUBCMD_MKENT(dev, 2, 1, do_wdt_dev),
	U_BOOT_SUBCMD_MKENT(start, 3, 1, do_wdt_start),
	U_BOOT_SUBCMD_MKENT(stop, 1, 1, do_wdt_stop),
	U_BOOT_SUBCMD_MKENT(reset, 1, 1, do_wdt_reset),
	U_BOOT_SUBCMD_MKENT(expire, 2, 1, do_wdt_expire));
```
若我们需要自定义一个命令，可参考如下流程（以test_cmd命令为例）:
* a 在cmd目录下创建一个源文件test_cmd.c  
* b 在该目录的Makefile中添加编译规则：

```text
   　obj-$(CONFIG_CMD_TEST_CMD) += test_cmd.o
```
* c 在该目录下的Kconfig文件中添加相应的配置项CONFIG_CMD_TEST_CMD  
* d 在test_cmd.c中根据签名的命令定义宏添加命令，并实现其命令处理函数

# 3. uboot的kernel封装

uboot主要用于启动操作系统，以armv8架构下的linux为例，其启动时需要包含kernel、dtb和rootfs三部分。uboot镜像都是以它们为基础制作的，因此在介绍uboot镜像格式之前我们需要先了解一下它们的构成。

## 3.1 内核的几种镜像
### 3.1.1 vmlinux镜像

linux内核编译完成后会在根目录生成原始的内核文件为vmlinux，使用readelf工具可看到其为elf文件格式：
```
`readelf -h vmlinux`
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0xffff800010000000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13681696 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
```

**由于uboot引导的镜像不能包含elf头，因此该镜像不能直接被uboot使用**。

### 3.1.2 image和zImage镜像

Image镜像是vlinux经过objcopy去头后生成的纯二进制文件，对于armv8架构其编译的Makefile如下：
```text
OBJCOPYFLAGS_Image := -O binary -R .note -R .note.gnu.build-id -R .comment –S   （1）
targets := Image Image.bz2 Image.gz Image.lz4 Image.lzma Image.lzo
$(obj)/Image: vmlinux FORCE                                                     
        $(call if_changed,objcopy)                                               （2）

$(obj)/Image.bz2: $(obj)/Image FORCE
        $(call if_changed,bzip2)                                                 （3）

$(obj)/Image.gz: $(obj)/Image FORCE
        $(call if_changed,gzip)                                                  （4）

$(obj)/Image.lz4: $(obj)/Image FORCE
        $(call if_changed,lz4)                                                   （5）

$(obj)/Image.lzma: $(obj)/Image FORCE
        $(call if_changed,lzma)                                                  （6）

$(obj)/Image.lzo: $(obj)/Image FORCE
        $(call if_changed,lzo)     
```

* （1）objcopy命令使用的flag定义  
* （2）以vmlinux为原始文件，通过objcopy命令制作Image镜像：
	* 其命令可扩展如下：`aarch64-linux-gnu-objcopy -O binary -R .note -R .note.gnu.build-id -R .comment -S vmlinux Image`
	* `–O` binary：将输出二进制镜像，即会去掉elf头
	*  `–R` .note：-R选项表示去掉镜像中指定的section，如这里会去掉.note、.note.gnu.build-id和.comment段
	* `–S`：去掉符号表和重定位信息，它与-R选项的功能类似，都是为了减小镜像的size  
	　　因此，执行该命令后生成的Image镜像是去掉elf头，去掉.note等无用的section，以及strip过的二进制镜像。它可以被uboot的booti命令直接启动。但若要使用bootm启动，则还需要将其进一步封装为后面介绍的uimage或bootimg镜像
* （3 – 7）以Image为源文件，调用不同的压缩算法，对镜像进行压缩。若调用gzip命令，则可将其压缩为我们熟悉的zImage镜像。与Image一样，压缩后的镜像也是可以被booti直接启动，且经过封装以后可以被bootm启动的

## 3.2 设备树
设备树是设备树dts源文件经过编译后生成的，其目标文件为二进制格式的dtb文件。其示例编译命令如下：
```text
dtc -I dts -O dtb -o example.dtb example.dts
```
（1） –I：指定输入文件格式  
（2）–O：指定输出文件格式  
（3）–o：指定输出文件名

设备树还支持**dtb overlay机制**，即可以向设备提供一个基础dtb和多个dtbo镜像，并在启动前将它们merge为最终的dtb。在嵌入式Linux下，设备树（device tree）用来描述硬件平台的各种资源，Linux内核在启动过程中，会解析设备树，获取各种硬件资源来初始化硬件[^12]。设备树的**overlay功能是指可以在系统运行期间动态修改设备树**。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220722160848.png)

一般情况下，如上图所示，设备树经过DTC编译器编译为二进制的hello.dtb文件，加载到内存，随Linux内核一起启动后，一般就无法更改了。如果我们想修改设备树，需要修改hello.dts文件文件，重新编译成二进制文件：hello.dtb，然后重新启动内核，重新解析。**有了设备树的overlay功能，省去了设备树的重新编译和内核重启，我们可以直接编写一个设备树插件：overlay.dts，编译成overlay.dtbo后，直接给设备树“打补丁”，在运行期间就可以动态添加节点、修改节点**。

设备树的overlay功能，在很多场合都会用得到，会让我们的开发更加方便：
-   外界插拔设备，无法在设备树中预先描述：耳机
-   树莓派 + FPGA开发板
-   基于I2C的温度传感器
-   管脚的重新配置：PIN multiplexing
-   修改bootcmd、分区

**dtb overlay测试**可以参考文献[^11]。

## 3.3 根文件系统

linux可以支持多种形式的根文件系统，如initrd、initramfs、基于磁盘的根文件系统等。站在启动镜像的角度看其实它们都是制作好的文件系统镜像，内核可以从特定的位置获取并挂载它们。以下是它们在启动时的基本特性：

（1）**initrd**: 它是一种内存文件系统，需要由bootloader预先加载到内存中，并将其内存地址传递给内核。如uboot将initrd加载到地址$initrd_addr处，则bootm参数如下：`bootm  $kernel_addr  $initrd_addr  $fdt_addr`
（2）**initramfs**:  initramfs也是一种内存文件系统，但与initrd不同，它是与内核打包在一起的。因此不需要通过额外的参数
（3）**磁盘rootfs**: 磁盘根文件系统会被刷写到flash、mmc或disk的分区中，在内核启动时可在bootargs添加下面格式的参数，以指定根文件系统的位置`root=/dev/xxx`

因此，以上这些rootfs只有initrd是需要uboot独立加载的，故只有当rootfs为initrd时，uboot镜像打包流程才需要在镜像打包时为其单独考虑。参考[^12]

## 3.4 uImage
### 3.4.1 Legacy uimage格式
uboot最先支持legacy uimage格式的镜像，它是在内核镜像基础上添加一个64字节header生成的。该header信息用于指定镜像的一些属性，如内核的类型、压缩算法类型、加载地址、运行地址、crc完整性校验值等。其格式如下：
```c
typedef struct image_header {
	uint32_t	ih_magic;	/* Image Header Magic Number	*/
	uint32_t	ih_hcrc;	/* Image Header CRC Checksum	*/
	uint32_t	ih_time;	/* Image Creation Timestamp	*/
	uint32_t	ih_size;	/* Image Data Size		*/
	uint32_t	ih_load;	/* Data	 Load  Address		*/
	uint32_t	ih_ep;		/* Entry Point Address		*/
	uint32_t	ih_dcrc;	/* Image Data CRC Checksum	*/
	uint8_t		ih_os;		/* Operating System		*/
	uint8_t		ih_arch;	/* CPU architecture		*/
	uint8_t		ih_type;	/* Image Type			*/
	uint8_t		ih_comp;	/* Compression Type		*/
	uint8_t		ih_name[IH_NMLEN];	/* Image Name		*/
} image_header_t;
```
uboot的bootm命令会解析镜像头中的信息，并根据这些信息执行镜像校验、解压和启动等流程。以下是创建uImage的命令示例：`mkimage -A arm64 -O linux -C none -T kernel -a 0x80008000 -e 0x80008040 -n Linux_Image -d zImage uImage`

但是它也有着一些缺点，如：  
（1）加载流程比较繁琐，如需要分别加载内核、initrd和dtb；
（2）启动参数较多，需要分别制定内核、initrd和dtb的地址；
（3）在支持secure boot的系统中对secure boot的支持不足。
为此，uboot又定义了一种新的镜像格式fit uimage，用于解决上述问题。

### 3.4.2 Fit uimage格式

Fit uimage是使用devicetree语法来定义uimage镜像描述信息以及启动时的各种属性，这些信息被写入一个后缀名为its的源文件中。以下是一个its文件的示例：
```text
/dts-v1/;

/ {
	description = "Various kernels, ramdisks and FDT blobs";
	#address-cells = <1>;

	images {
		kernel-1 {
			description = "vanilla-2.6.23";              （1）
			data = /incbin/("./vmlinux.bin.gz");         （2）
			type = "kernel";                             （3）
			arch = "ppc";                                （4）
			os = "linux";                                （5）
			compression = "gzip";                        （6）
			load = <00000000>;                           （7）
			entry = <00000000>;                          （8）
			hash-1 {
				algo = "md5";                        （9）
			};
			hash-2 {
				algo = "sha1";
			};
		};

		kernel-2 {
			description = "2.6.23-denx";
			data = /incbin/("./2.6.23-denx.bin.gz");
			type = "kernel";
			arch = "ppc";
			os = "linux";
			compression = "gzip";
			load = <00000000>;
			entry = <00000000>;
			hash-1 {
				algo = "sha1";
			};
		};

		kernel-3 {
			description = "2.4.25-denx";
			data = /incbin/("./2.4.25-denx.bin.gz");
			type = "kernel";
			arch = "ppc";
			os = "linux";
			compression = "gzip";
			load = <00000000>;
			entry = <00000000>;
			hash-1 {
				algo = "md5";
			};
		};

		ramdisk-1 {
			description = "eldk-4.2-ramdisk";
			data = /incbin/("./eldk-4.2-ramdisk");
			type = "ramdisk";
			arch = "ppc";
			os = "linux";
			compression = "gzip";
			load = <00000000>;
			entry = <00000000>;
			hash-1 {
				algo = "sha1";
			};
		};

		ramdisk-2 {
			description = "eldk-3.1-ramdisk";
			data = /incbin/("./eldk-3.1-ramdisk");
			type = "ramdisk";
			arch = "ppc";
			os = "linux";
			compression = "gzip";
			load = <00000000>;
			entry = <00000000>;
			hash-1 {
				algo = "crc32";
			};
		};

		fdt-1 {
			description = "tqm5200-fdt";
			data = /incbin/("./tqm5200.dtb");
			type = "flat_dt";
			arch = "ppc";
			compression = "none";
			hash-1 {
				algo = "crc32";
			};
		};

		fdt-2 {
			description = "tqm5200s-fdt";
			data = /incbin/("./tqm5200s.dtb");
			type = "flat_dt";
			arch = "ppc";
			compression = "none";
			load = <00700000>;
			hash-1 {
				algo = "sha1";
			};
		};

	};

	configurations {
		default = "config-1";

		config-1 {
			description = "tqm5200 vanilla-2.6.23 configuration";
			kernel = "kernel-1";
			ramdisk = "ramdisk-1";
			fdt = "fdt-1";
		};

		config-2 {
			description = "tqm5200s denx-2.6.23 configuration";
			kernel = "kernel-2";
			ramdisk = "ramdisk-1";
			fdt = "fdt-2";
		};

		config-3 {
			description = "tqm5200s denx-2.4.25 configuration";
			kernel = "kernel-3";
			ramdisk = "ramdisk-2";
		};
	};
};
```
它包含images和configurations两个顶级节点，images指定该its文件会包含哪些镜像，以及这些镜像的属性信息。configurations用于定义一系列镜像组合信息，如在本例中包含了config-1、config-2和config-3三种镜像组合方式。Its使用default属性指定启动时默认采用的配置信息，若启动时不希望使用默认配置，则可通过在启动参数中动态指定配置序号。下面我们通过kernel-1节点看下image属性的含义：
（1）镜像的描述信息  
（2）镜像文件的路径  
（3）镜像类型，如kernel、ramdisk或fdt  
（4）支持的架构  
（5）支持的操作系统  
（6）其使用的压缩算法  
（7）加载地址  
（8）运行地址  
（9）完整性校验使用的hash算法
configurations的属性比较简单，就是指定某个配置下使用哪一个kernel、dtb和ramdisk镜像。Fit image除了支持完整性校验外，还可支持hash算法 + 非对称算法的secure boot方案，如以下例子：

```text
kernel {
			data = /incbin/("test-kernel.bin");
			type = "kernel_noload";
			arch = "sandbox";
			os = "linux";
			compression = "none";
			load = <0x4>;
			entry = <0x8>;
			kernel-version = <1>;
			signature {
				algo = "sha1,rsa2048";        （1）
				key-name-hint = "dev";       （2）
			};
		};
```

（1）指定sha1为secure boot签名使用的hash算法，rsa2048为其使用的签名算法  
（2）可能使用的验签密钥名

与设备树类似，its文件可以通过mkimage和dtc编译生成itb文件。镜像生成方式如下：`mkimage -f xxx.its xxx.itb`

xxx.itb文件可以直接传给uboot，并通过bootm命令执行，如xxx.itb被加载到0x80000000，则其命令如下：`bootm 0x80000000`

若需要选择非默认的镜像配置，则可通过指定配置序号实现，例如：`bootm 0x80000000#config@2`

## 3.5 boot image

boot image是android定义的启动镜像格式，到目前为止一共定义了三个版本（v0 – v2），其中v0版本包含andr_img_hdr、kernel、ramdisk和second stage，v1版本增加了recovery dtbo/acpio，v2版本又增加了dtb。在这些镜像中second stage是可选的，而recovery dtbo只有在使用recovery分区的非AB系统中才需要，且它们都需要page对齐（通常为2k）。以下是boot image镜像的基本格式：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220722162207.png" width="44%" />
</div>

andr_img_hdr镜像头用于描述这些镜像的信息，如其长度、加载地址等，其定义如下：

```c
struct andr_img_hdr {
    /* Must be ANDR_BOOT_MAGIC. */
    char magic[ANDR_BOOT_MAGIC_SIZE];

    u32 kernel_size; /* size in bytes */
    u32 kernel_addr; /* physical load addr */

    u32 ramdisk_size; /* size in bytes */
    u32 ramdisk_addr; /* physical load addr */

    u32 second_size; /* size in bytes */
    u32 second_addr; /* physical load addr */

    u32 tags_addr; /* physical addr for kernel tags */
    u32 page_size; /* flash page size we assume */

    u32 header_version;
    u32 os_version;

    char name[ANDR_BOOT_NAME_SIZE]; /* asciiz product name */

    char cmdline[ANDR_BOOT_ARGS_SIZE];

    u32 id[8]; /* timestamp / checksum / sha1 / etc */

    char extra_cmdline[ANDR_BOOT_EXTRA_ARGS_SIZE];

    u32 recovery_dtbo_size;   /* size in bytes for recovery DTBO/ACPIO image */
    u64 recovery_dtbo_offset; /* offset to recovery dtbo/acpio in boot image */
    u32 header_size;

    u32 dtb_size; /* size in bytes for DTB image */
    u64 dtb_addr; /* physical load address for DTB image */
} __attribute__((packed));
```

可以使用mkbootimg.py脚本制作boot image镜像，该脚本参数比较简单，就是指定与镜像头中定义的相关参数。例如：

```text
mkbootimg.py \
    	--base aaa \
        --kernel kernel/arch/arm64/boot/Image.gz \
        --kernel_offset bbb \
        --second kernel/arch/arm64/boot/ccc.dtb \
        --second_offset ddd \
        --board test_board \
        -o boot.img
```


## 3.6 boot flow
bootm是uboot用于启动操作系统的命令，它的主要流程包括根据镜像头获取镜像信息，解压镜像，以及启动操作系统。以下为其主要执行流程：
![](https://raw.githubusercontent.com/carloscn/images/main/typorado_bootm-do_bootm_states.km-2.svg)
以上流程最终会调用特定os的启动函数，例如需要启动armv8架构的linux，则其调用的接口为arch/arm/lib/bootm.c中的do_bootm_linux。以下为其执行流程：
![](https://raw.githubusercontent.com/carloscn/images/main/typorado_bootm_linux.svg)

# Reference
[^1]:[01_Embedded_ARMv7/v8 non-secure Boot Flow #61](https://github.com/carloscn/blog/issues/61)
[^2]:[02_Embedded_ARMv8 ATF Secure Boot Flow (BL1/BL2/BL31) #65](https://github.com/carloscn/blog/issues/65)
[^3]:[# 聊聊SOC启动（五） uboot启动流程一](https://zhuanlan.zhihu.com/p/520060653)
[^4]:[arm64 中的 spin-table 和 psci 两种启动多核流程分析](https://blog.csdn.net/eric43/article/details/81154430)
[^5]:[# linux cpu管理（二） spin-table启动](https://zhuanlan.zhihu.com/p/537049863)
[^6]:[# linux cpu管理（三） psci启动](https://zhuanlan.zhihu.com/p/537381492)
[^7]:[ARM多核处理器启动过程分析](https://blog.csdn.net/qianlong4526888/article/details/27695173)
[^8]:[[2/2] arm64: add better spin-table support](https://patches.linaro.org/project/u-boot/patch/1466167909-15345-2-git-send-email-yamada.masahiro@socionext.com/)
[^9]:[u-boot kconfig](https://github.com/u-boot/u-boot/blob/master/arch/arm/cpu/armv8/Kconfig#L40)
[^10]:[# Linux内核编程12期：设备树overlay与ConfigFS文件系统](https://blog.csdn.net/zhaixuebuluo/article/details/124825619)
[^11]:[# 树莓派设备树覆盖（dtb overlay）](https://blog.csdn.net/abc4764/article/details/112759220)
[^12]:[# ramfs,rootfs,initramfs,initrd](https://blog.csdn.net/rikeyone/article/details/52048972)