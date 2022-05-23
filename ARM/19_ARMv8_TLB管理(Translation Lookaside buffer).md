# *19_ARMv8_TLB管理(Translation Lookaside buffer)*

# 0. 引言

TLB这块我们之前在学习MMU的时候有过涉及。TLB作为一个MMU的旁路器件辅助CPU以最快的速度能找出虚拟地址和物理地址之间的对应关系，之前的文章只是泛泛的介绍了TLB的功能，并没有一个比较感性的认识。这一节内容我们来重新认识一下TLB， 并且将TLB和cache的结构做对比，最后我们引入一个2017年在CPU界发生的重大漏洞“CPU熔断”，这个漏洞在ARM上就是在TLB上面做了一些工作来解决这个安全漏洞的。

-----

# 1. TLB Basic

TLB是一个很小的高速缓存，是MMU的一个辅助机。TLB包含的项（TLB Entry）数量比较少，每个TLB项包含页帧号（VPN），物理也帧号（PFN）以及一些属性。当处理器要访问一个虚拟地址的时候，首先会从TLB中查找是不是包含这个地址，如果命中皆大欢喜；如果不命中TLB将会访问MMU，从MMU中获取目标虚拟地址对应的物理地址（规则按照多级页表查询的方法），并组成TLB项，替换到自己的TLB列表内。

## 1.1 TLB性能影响因素

在ARMv8中，最高等级的页表支持4级，如果没有TLB，那么每次内存访问将要进行4级页表的索引，这样大大的降低了效率。我们不妨引入一个有趣的实验，在Cortex-A9硬件下，测试TLB的影响。这个例子来源于StackOverflow，使用三星Galaxy S3做了测试，相关配置如下：

*   I-cache和D-cache各配套一个TLB，我们非正式定义为I-TLB（32-entries or 64-entries），D-TLB（32-entries）
*   在L2 cache上还有个main TLB，我们命名为M-TLB

这个论文里面[^2]在比较老的机器上CRAY-T3D和DEC workstation上面对cache进行探究，即便是比较老了，但是还具备参考意义，可以好好读一读。

![image-20220518100008470](https://raw.githubusercontent.com/carloscn/images/main/typoratyporaimage-20220518100008470.png)

在三星的这个芯片中，有32个TLB entry，随着enties的数量升高，所要查询时间越高，甚至在在260个entry之后，性能出现了极具的衰减，因此可以得到结论，**并不是TLB越大性能越好**。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typorasP6Bo.png" width="50%" />
</div>

## 1.2 TLB在SoC位置

ARMv8-A的体系结构手册没有约定TLB项的结构，我们在cortex-A72的手册中来找到一些参考的设计，在其他的SoC中，TLB的位置并不是一样的。下图是Cortex-A72 cluster 4个cores中的TLB的位置[^3]（25页），所有的TLB都是和cache紧挨着的。每个core上面针对D-cache存在一个d-tlb，对于i-cache有一个配套的i-tlb，还有与L2-cache配套的L2-TLB。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220518103036227.png" width="70%" />
</div>

根据介绍[^3]，Cortex-A72的每一个核的TLB大小为：

*   **L1-I-TLB：48-entry fully-associative (4KB/64KB/1MB page size)，PIPT结构**
*   **L1-D-TLB: 32-entry fully-associative (4KB/64KB/1MB page size)，PIPT结构**
*   **L2-TLB: 4-way set-associative unified 1024-entry。**

这里引入这个就是让我们有一个定量的概念，**一级的TLB的项目并不是很多，48/32这种；而二级TLB能达到1024个entry**。**在L1 TLB采用的是全相联的方式，可以视为一个1维的**，没有什么可说的；

而L2的cache采用了组相联的方式，如图所示：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220518120153162.png" width="62%" />
</div>
## 1.3 ASID

芯片工程师和操作系统设计人员共同努力需要解决的问题。在芯片层级上，引入了上面结构提到的**ASID (Address space ID)**。

### 1.3.1 进程切换的效率问题

#### 单核场景

首先感谢[^5]提供两种思路。我们先看一下单核时候在多进程进行切换的时候TLB的变化，从进程的角度来看，这里有三类空间：

*   进程1的空间（我们定义，用户空间为S_P1U，内核空间为S_P1K）
*   进程2的空间（我们定义，用户空间为S_P2U，内核空间为S_P2K）
*   kernel的空间 （S_K）

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220518130727871.png" width="50%" />
</div>

CPU上面运行了这些P1和P2还有K的进程，对于K进程而言还算是没有什么问题所有的进程都是共享这部分，而对于P1和P2都有自己的用户地址空间。比如，现在这样的状况，从进程的视角来看，P1和P2有着一样的虚拟地址空间，但是在进程的TLB上，却有着不一样的映射。在CPU上面正常的P1和P2的映射关系如图，他们分别映射不一样的位置。假如P1切到P2没有刷新TLB的时候，CPU就会在P2的时候使用了P1的地址，这就出现了TLB的同名问题（TLB其歧义问题[^4]）

>##### TLB的歧义问题
>
>我们知道不同的进程之间看到的虚拟地址范围是一样的，所以多个进程下，不同进程的相同的虚拟地址可以映射不同的物理地址。这就会造成歧义问题。例如，进程A将地址0x2000映射物理地址0x4000。进程B将地址0x2000映射物理地址0x5000。当进程A执行的时候将0x2000对应0x4000的映射关系缓存到TLB中。当切换B进程的时候，B进程访问0x2000的数据，会由于命中TLB从物理地址0x4000取数据。这就造成了歧义。如何消除这种歧义，**我们可以借鉴VIVT数据cache的处理方式，在进程切换时将整个TLB无效**。切换后的进程都不会命中TLB，但是会导致性能损失。

如果我们粗暴的flush整个TLB这个会带来性能上极大的损失，所以有没有一种方法可以降低TLB的刷新几率呢？当然有，对于内核空间而言，这部分空间无论是哪个进程，他们都是一致的，所以这部分就算是切换掉，也不需要flush；而对于用户空间而言，对于他们来说是私有的，因此就需要flush操作。**因此，在ARM上将TLB区分为全局性的TLB和局部性TLB。**

为了支持标识TLB的类别，则使用ASID（Address Space ID）来进行编址。有了ASID的硬件方案，进程切换的时候根据ASID来判断自己是否需要flush TLB。在ARMv8-A还有ARMv9-A中使用寄存器TTBRx_TL1[^6]来维护ASID的值。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220518133308332.png" width="40%" />
</div>

ARMv8有两个TTBR，一个是0，一个是1，我们在使用的时候使用任意一个都可以，只不过需要在TCR寄存器的A1的域来选择那个TTBR生效。ASID并不是进程的PID，ASID使用的bitmap模式管理，具体如何管理参考[^7]。如果是8位的bitmap，那么就有256个号码同时存在可以区分。实际上L2的页表应该如下所示：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220518134230224.png" width="62%" />
</div>

#### 多核场景

进程调度多进程，可能存在的情况比单核的时候要复杂，因为TLB在每个core上面都有，而且TLBs是相互隔离的不共享任何数据的。大体可分为两种情况：

*   多个进程被调度一个CPU管理，那么TLB的问题如单核场景是一致的。
*   多个进程被调度多个CPU管理，那么在切换多核之前，系统要对TLB进行flush，为新切入的进程准备一个干净的TLB。如下图所示 。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220518135637026.png" width="50%" />
</div>

在多核系统有个问题需要注意[^5]：

>不过，对于多核系统，这种情况有一点点的麻烦，其实也就是传说中的TLB shootdown带来的性能问题。在多核系统中，如果cpu支持PCID并且在进程切换的时候不flush tlb，那么系统中各个cpu中的tlb entry则保留各种task的tlb entry，当在某个cpu上，一个进程被销毁，或者修改了自己的页表（也就是修改了VA PA映射关系）的时候，我们必须将该task的相关tlb entry从系统中清除出去。这时候，你不仅仅需要flush本cpu上对应的TLB entry，还需要shootdown其他cpu上的和该task相关的tlb残余。而这个动作一般是通过IPI实现（例如X86），从而引入了开销。此外PCID的分配和管理也会带来额外的开销，因此，OS是否支持PCID（或者ASID）是由各个arch代码自己决定（对于linux而言，x86不支持，而ARM平台是支持的）。

## 1.4 TLB指令

*   使所有的TLB表失效
*   使ASID对应的某一个TLB项失效
*   使ASID对应的所有的TLB失效
*   使虚拟地址对应的所有TLB失效

`TLB <type><level>{IS} {<xt>}`[^8]

## 1.5 TLB案例

这部分在Linux内核机制部分，不是本文的重点，我们在这里做一些简单的论述。2018年的时候，各大媒体开始渲染“幽灵”和“熔断”的CPU漏洞，说Intel有重大的bug，后来ARM和AMD也相继确认的确有这样的漏洞影响。Meltdown（熔断）对应编号恶意数据缓存加载 CVE-2017-5754 ，Spectre （幽灵）对应编号边界检查绕过 CVE-2017-5753 、分支目标注入 CVE-2017-5715[^9] 。从测试现象来看，就是在用户空间的程序可以无权限的访问kernel space的内存，因此就导致很多在kernel space上的机密数据泄露出来。

>这次漏洞利用了 CPU 执行中对出现故障的处理。由于现在 CPU 为了提供性能，引入了乱序执行和预测执行。
>
>-   乱序执行是指 CPU 并不是严格按照指令的顺序串行执行，而是根据相关性对指令进行分组并行执行，最后汇总处理各组指令执行的结果。
>-   预测执行是 CPU 根据当前掌握的信息预测某个条件判断的结果，然后选择对应的分支提前执行。
>
>这两种执行在遇到异常时，CPU 会丢弃之前执行的结果，将 CPU 的状态恢复到乱序执行或预测执行前的正确状态，然后继续执行正确的指令，从而保证了程序能够正确连续的执行。但是问题在于，CPU 恢复状态时并不会清除 CPU 缓存中的内容，而这两组漏洞正是利用了这一设计上的缺陷进行侧信道攻击。**乱序执行对应的利用即为 Meltdown ，而预测执行对应的利用即为 Spectre 。**

CPU架构级的漏洞很难修补，只能通过软件来进行一些弥补，因此Linux kernel方案就是采用KPTI的方案：

*   KPTI方案
*   BBM机制

# 2. REF

[^1]:[Measurement of TLB effects on a Cortex-A9](https://stackoverflow.com/questions/30530300/measurement-of-tlb-effects-on-a-cortex-a9)
[^2]:[**Empirical_Evaluation_of_the_cray-t3d_a_compiler_perspecitve.pdf**](https://github.com/carloscn/doclib/blob/master/paper/software/cache/Empirical_Evaluation_of_the_cray-t3d_a_compiler_perspecitve.pdf)
[^3]:[cortex_a72_mpcore_trm_100095_0003_06_en.pdf](https://github.com/carloscn/doclib/blob/master/man/arm/armv8/cortex_a72_mpcore_trm_100095_0003_06_en.pdf)
[^4]:[TLB原理 ](https://zhuanlan.zhihu.com/p/108425561)
[^5]:[进程切换分析（2）：TLB处理 ](http://www.wowotech.net/process_management/context-switch-tlb.html)
[^6]:[Arm Armv9-A Architecture Registers - TTBR1_EL1, Translation Table Base Register 1 (EL1) ](https://developer.arm.com/documentation/ddi0601/2021-12/AArch64-Registers/TTBR1-EL1--Translation-Table-Base-Register-1--EL1-?lang=en)
[^7]:[BitMap的原理介绍与实现 ](https://blog.csdn.net/wf19930209/article/details/79120000)
[^8]:[Arm A-profile A64 Instruction Set Architecture  - TLBI](https://developer.arm.com/documentation/ddi0602/2022-03/Base-Instructions/TLBI--TLB-Invalidate-operation--an-alias-of-SYS-?lang=en)
[^9]:[“幽灵”和“熔断”，究竟是什么？（事件汇总） ](https://zhuanlan.zhihu.com/p/32810180)
