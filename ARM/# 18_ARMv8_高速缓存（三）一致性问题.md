# *18_ARMv8_高速缓存（三）多核与一致性要素*

探究cache的coherency，还要从ARM的多核处理器的历史讲起，为什么会引入这个问题，这个问题给业界带来什么挑战，我作为一个软件工程师，在多核系统上应该如何开发？

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513130401726.png" width="80%" />
</div>

2004年ARM登上头版，宣布已经集成多核系统，实际上，2003年在学术界已经提出了这个架构的的模型了。2005年的时候ARM宣布帮助英伟达生产下一代的多核系统，如图所示。2006年Cortex-A8出世，不过Cortex- A8是一个单核的CPU，没有在多核层面的coherency的问题，不过在这个系统里面有DMA，会存在DMA和Cache之间的一致性问题；Cortex-A9的时候，加入了MPCore的设计，因此在core和core之间存在硬件一致性的问题，通常做法实现MESI总线协议（SCU，如下图），我们定义为**多核之间的一致性问题**；Cortex-A15中引入了大小核的概念，也就是把多个大核放在一起统称一个cluster，把所有的小核放在一起统称为一个cluster（簇），MESI总线协议可以保证在一个cluster内的一致性，而cluster和cluster之间的一致性需要另外引入硬件机制来保证，还有cluster和DMA、GPU之间也会引入一致性问题，ARM有现成的IP（CCI-400，CCI-500等），我们把cluster和cluster的一致性、cluster与gpu或者dma的一致性问题称为：**系统之间的一致性性问题**；

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513125850380.png" width="100%" />
</div>

多核面向以下产品，这些产品多任务，且有着更高的性能。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513133907106.png" width="50%" />
</div>

我们再来看一组数据，基于多核系统，在有cache和没有cache下的性能对比，红色是有L1 cache的，蓝色是没有L1 cache，分别进行memset和jpeg压缩，能看到cache在多核上的性能提高了52%左右。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513134129475.png" width="50%" />
</div>

题外话就这么多，不如这一节的正题，我们在这个部分需要了解，缓存一致性问题怎么演变过来的。

*   多核系统multiple-cores processors。
*   为什么需要cache一致性。
*   cache coherency在业界有什么解决方案。
*   MESI协议

*note, 本章是【要素】更多的是理论和模型探讨，在高速缓存（四）会对ARMv8架构的一致性问题进行探究。*

# 1. 系统间的一致性

以下这个结构图来源于ARM 64 Bit  Juno r2 ARM Development Platform 评估板，根据这个图，一共有5个我们比较关系的cluster：

*   **cluster 1**: 

    Cortex-A72，包含一个两个core0 和 core1，为方便表述我们用 a72.core0, a72.core1表述。每个a72.core都有一个48KB的L1 I-Cache，32KB的L1 D-Cache；共享L2 Cache 2MB[^2]。

*   **cluster 2**: 

    Cortex-A52，包含4个core。a53.core0 - a53.core3表示。每个a53.core都有一个L1-32KB，L2 Cache 1MB[^2]。

*   **cluster 3**: 

    Mali-T624, GPU，我们直接用mali.shader0 - mali.shader3 表示。L2 cache是128KB[^2]。

*   **cluster 4**: 

    Cortex-M3，控制器，非多核结构，m3表示。

*   **cluster 5**:

    DMA，用dma表示。


![image-20220513143244839](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513143244839.png)

ARM的SoC越来越复杂，从multi-core演变为multi-cluster，每个cluster里面的core都有自己的L1 cache，每个cluster都有一个可以sharing的L2 cache。cluster通过ACE（AXI coherent Extension，AMBA 4中定义）连接到CCI-400（缓存一致性控制器）。在这个系统内部，除了CPU还有GPU，还有MCU，还有带DMA功能的外设，这些设备都有访问内存的功能，因此它们也必须通过ACE接口来连接到CCI-400上面，使整个系统层级的存储系统，保持一致性。

**综上所述，一致性问题存在整个系统中，是由于cluster之间的隔离引起，为了保证一致性，在SoC设计的时候将这些clusters通过ACE连接到CCI-400控制器上，也就是说，系统间的一致性，是由CCI-400完成的，从软件工程师的角度，我们不需要维护系统的一致性。**

## 1.1 系统一致性举例

这有个DMA控制器的场景[^3]，这个会影响CPU（cache）与memory的一致性。此时假设CPU的cache使用write-back策略，CPU更新了cache的数据，还没有clean到内存里，这个时候CPU想向外设依靠DMA write这个数据，DMA拿到的数据是一个和cache不一致的数据，这就造成了一致性问题；同样，内存从DMA read了数据并没有把数据刷到cache里面，CPU在不知情的情况下拿着旧数据做操作，这也会出现问题。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora16cchdma.png" width="80%" />
</div>

## 1.2 一致性解决方案

解决一致性问题通常有一些方案：

*	直接关闭cache （不可能）
*	要求软件编程者维护cache一致性 （最好）
*	要求硬件设计者维护cache一致性 （居中）

### 1.2.1 关闭cache（最差）

关闭cache不用想了，只要知道关闭cache不会存在一致性问题就好了。但为了解决一致性问题而关闭cache，得不偿失。在cache关闭情况下，每一次CPU都要主动从内存读写数据，这个一是性能降低，二是功耗也升高。

从软件硬件角度解决一致性问题是现在业界流行的方法。

### 1.2.2 硬件角度（中等）

处理器可以从硬件角度解决一致性问题，对于软件编程者来说这是一件很开心的事情，因为不需要自己维护cache了，减少了很大的工作量。可这样增加了硬件机制和复杂度。

#### 系统层级

对于系统级别的一致性问题，我们上面也有提到，大部分SoC设计也采用了这样的方法，即把cluster和cluster通过**ACE总线接入CCI-400/CCI-500**的硬件模块上，软件设计可以无痛的不需要考虑系统级的一致性问题，大胆的使用DMA搬运数据。

#### 核间层级

对于多核之间的一致性问题，在硬件上也可以解决，通常做法就是在多核内部实现一个**MESI协议**的控制器（Snoop Control Unit），这个控制器在硬件设计上有自己的状态机，会在总线上监听所有核心的状态。但并不是所有的处理器都会制作一个这样的硬件来维护一致性。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513182348414.png" width="70%" />
</div>


### 1.2.3 软件角度（最优）

从软件角度处理一致性问题，是现在综合成本最优的方案。在软件上，编程人员需要在合适的时候使用clean、invalidate和flush。缺点当然是会增加软件设计的复杂度，且debug和trace成本比较高，出问题比较隐晦，甚至需要一帧一帧的分析数据，这也可能是个随机错误，对！海森堡bug；另外，软件clean和invalidate还有flush都是让cache和内存打交道的行为，这样会增加功耗成本，频繁清理cache，并不是一个好的办法。因此，刷cache是一个非常依赖于开发者的经验，和QA团队测试力度的解决方案。

**一致性的问题解决角度，大部分系统并不是完全依赖硬件或者软件，这对谁都不公平。So traded off，很多软件硬件各顶半边天，系统层级的交给硬件来解决，而cluster内部的一致性需要软件开发人员自己来维护。**

# 2. 多核间的一致性与cache伪共享

把上面的图简化为下面的图，图片来源[^2]。**多核之间一致性是属于cluster内部的**，比如a72.core0和a72.core1之间的一致性问题，而不一致的地方发生在自己私有的L1 cache和共享的L2 cache上面。

![image-20220513162837630](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220513162837630.png)

## 2.1 MESI协议（硬件角度解决）

核间的cache保持一致性，如果依赖硬件机制的话，就是要增加SCU机制，而SCU机制使用的是MESI协议。这部分的确是设计SCU机制IP的人去做，而并非我们软件工程师的任务，但我觉得我们有必要来理解一下MESI协议的思想，当没有SCU这个机制的时候，实际上是需要我们软件工程师实现一个软件版本的精简的SCU的。MESI协议对我们理解cache的状态很有好处，所以这里也展开这个话题讨论一下（不会太细致，只是思想借鉴）。



# 2. Ref

[^1]:[ARM Versatile Express Juno r2 Development Platform (V2M-Juno r2) Technical Reference Manual Version 2.0 - Juno r2 ARM Development Platform SoC](https://developer.arm.com/documentation/100114/0200/Introduction?lang=en)
[^2]:[64 Bit Juno r2 ARM® Development Platform](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Juno%20r2%20datasheet.pdf)
[^3]:[Flushing Cached Data during DMA Operations](https://github.com/MicrosoftDocs/windows-driver-docs/blob/staging/windows-driver-docs-pr/kernel/flushing-cached-data-during-dma-operations.md)
[^4]:[]()
[^5]:[]()
[^6]:[]()
[^7]:[]()
[^8]:[]()
[^9]:[]()
[^10]:[]()
