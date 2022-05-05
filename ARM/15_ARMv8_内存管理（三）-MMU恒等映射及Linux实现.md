# *15_ARMv8_内存管理（三）-MMU恒等映射及Linux实现*

-   以ELF作为引子展开MMU的作用，为了和ELF做一些知识关联工作。[13_ARMv8_内存管理与MMU（一）-内存管理要素]
-   以《操作系统概念》的虚拟内存管理一节，为MMU做一些理论和基础知识的储备。[13_ARMv8_内存管理与MMU（一）-内存管理要素]
-   以《ARMv8体系架构》的MMU为例子，了解在CPU层级MMU的实现和配置。[13_ARMv8_内存管理与MMU（二） - ARMv8架构虚拟内存MMU]
-   以《Linux内核》的内存管理为例子，来研究MMU怎么实现的。[Linux_Kernel_MMU设计（一）]

# 1. 恒等映射

## 1.1 boot stage MMU

在讨论恒等映射之前，我们我们先来弄清楚一个问题，**为什么在uboot阶段需要关闭MMU[^1]？**

uboot阶段关闭MMU是一种推荐做法，这个在uboot的文档[^3]和Linux文档[^2]上都有体现。不过，uboot阶段是可以开启MMU的，可能是为了一些测试或一些可执行文件在boot环境[^2]，也有文献表明在一些加解密安全特性下，在boot环境考虑使能MMU[^4]。

如果在boot阶段使能MMU，要从两个维度考虑，第一个，**页表映射引入的问题**[^5]；第二个，**d-cache关闭和i-cache开启**[^1]的功能。

在某些处理器里面，并不是像ARMv8体系架构一样，**cache隶属于MMU下面的一个模块，而是cache和MMU是分立的两个机制**，MMU的功能仅仅是页表映射。因此这就引发了一些讨论，我们从上一个内存管理（二）里面提到ARM将内存划分，device memory和normal memeory，device memory就是一般性的存储空间，比如ROM, RAM, Flash；而device memory就是外围设备映射的空间，这个分水岭是以cacheing功能做的划分，如果此时我们并**没有打开MMU，而开了D-Cache**，这有可能那块内存刚好被映射到device memory上面，如果刚好是一些比如fifo读敏感的设备，那么就会发生很致命的错误。在这种情况下就必须要使能MMU，让MMU页表的映射（MMU内存有相应的属性和合法性检查），这就会被视为安全的访问。简而言之，**在没有MMU的保护下，cacheing设备内存是一个非常错误的操作。**

I-Cache打开没有问题，因为指令部分不会从device memory这个位置去取指，而是从normal memory去取指，就算是没有MMU的属性保护，也不会发生错误。**如果MMU在boot阶段关闭，且开了I-Cache**，这里需要注意的是，当使能MMU的时候一定要对cache内的数据进行invalidate操作，否则cache内部可能会记录在MMU没有映射之前VMA和PHY恒等映射的地址，而不是MMU映射后的地址。还要注意在JTAG的情境下，我们会把程序直接load到ram里面，此时i-cache没有周期性的从ram刷新数据，所以我们运行程序的时候，可能就出现，程序可能完全是ram中的，可能是ram的也可能是cache旧的，或者完全是cache中旧的，因此要注意i-cache使能情况下，jtag的cache一致性。

所以，综上考虑， MMU的推荐配置通常是：

*   MMU关闭
*   D-Cache关闭
*   I-Cache开启



# 3. Ref

[^1]:[ARM Bootloader: Disable MMU and Caches](https://stackoverflow.com/questions/21262014/arm-bootloader-disable-mmu-and-caches)
[^2]:[[PATCH 1/2\] ARM: allow booting with MMU enabled](https://www.mail-archive.com/barebox@lists.infradead.org/msg32223.html)
[^3]:([Does armv7 u-boot use MMU?](https://stackoverflow.com/questions/25152073/does-armv7-u-boot-use-mmu))[https://stackoverflow.com/a/25173623/14574212]
[^4]:[ARM Bootloader: Disable MMU and Caches- second comments form artless noise](https://stackoverflow.com/questions/21262014/arm-bootloader-disable-mmu-and-caches)
[^5]:[Why must I enable the MMU to use the D-Cache but not for the I-Cache? ](https://www.cnblogs.com/pengdonglin137/p/10221932.html)
[^6]:()[]
[^7]:()[]
[^8]:()[]
[^9]:()[]
