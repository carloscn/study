# 01_ARMv7 boot Flow

从学习uboot开始，就误以为一个板子的开启，上电后的所有操作都是uboot来做的，那时候只关注于uboot能把内核引导成功；工作之后又接触到了secure boot，而在我们的secure boot设计中有好几级引导，我知道了boot并不是只限于uboot，我们可以根据工程的需要设计多级的boot；再当学到SoC上面的知识的时候，在boot阶段除了引导之外还有很多很多工作要做。因此，我决定把boot这块所有的相关的内容整理出来，顺便把secure boot的一些做法写出来，看看在boot哪个阶段，我们可以用secure boot。

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

通常状况的SPL是一个非常小的程序（64KB）以下，运行在SoC内部的SRAM上面。SPL通常存储在启动媒介里面，**因此这部分程序并不是由soc厂家负责开发的**。在uboot源程序中时机已经包含了这部分程序。


# 3.Ref

[^1]:[AM335x Sitara Processors datasheet.pdf](https://github.com/carloscn/doclib/blob/master/man/arm/ti/AM335x%20Sitara%20Processors%20datasheet.pdf)
[^2]:[**AM335x and AMIC110 Sitara Processors TRM.pdf**](https://github.com/carloscn/doclib/blob/master/man/arm/ti/AM335x%20and%20AMIC110%20Sitara%20Processors%20TRM.pdf)
[^3]:[**armv8m_architecture_memory_protection_unit_100699_0100_00_en.pdf**](https://github.com/carloscn/doclib/blob/master/man/arm/armv8/armv8m_architecture_memory_protection_unit_100699_0100_00_en.pdf)
[^4]:[OP-TEE基本的从芯片设计到给客户的安全问题浅析](https://blog.csdn.net/zxpblog/article/details/107410971)
[^5]:[An Exploration of ARM TrustZone Technology](https://genode.org/documentation/articles/trustzone)
[^6]:[U-Boot 之五 详解 U-Boot 及 SPL 的启动流程](https://blog.csdn.net/ZCShouCSDN/article/details/121925283)
[^7]:[Embedded Linux Booting Process (Multi-Stage Bootloaders, Kernel, Filesystem)](https://www.youtube.com/watch?v=DV5S_ZSdK0s)
[^8]:[**i.MX 8QuadMax Applications Processor Reference Manual.pdf**](https://github.com/carloscn/doclib/blob/master/man/embedded/nxp/i.MX%208QuadMax%20Applications%20Processor%20Reference%20Manual.pdf)









