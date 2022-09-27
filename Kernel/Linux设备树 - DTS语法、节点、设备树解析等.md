# Linux设备树- DTS语法、节点、设备树解析等
设备树的故事还要从Linus的一个merge的commit讲起[^1]：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923124525.png)

Linus已经受够了合并这些关于驱动的“鬼东西”，而这些也是由于ARM的丰富的生态促使各个厂家不得不更新基于自己架构的驱动，这些东西不断merge进Linux内核让内核越来越臃肿。Linus强调这是平台的概念，而不是kernel。

我们在arm架构下面依旧可以找到一些型号比较老旧的架构，虽然linux4.1.15已经引入了设备树的概念，但是为了兼容老旧的芯片架构这里也不得不让这些架构保留在内核中了。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923125025.png)

引入设备树之后，厂家都采用了最新的设备树来管理设备，设备树是一种方法-数据分离的实现方式，可以在Linux内核编译完成之后动态的扩充设备。Arm 社区开始引入之前 powerPC 架构就采用的设备树，将描述这些板级信息的文件与 Linux 内核代码分离，**Linux 4.x 版本几乎都支持设备树，所有开发板的设备树文件统一放在`arch/arm/boot/dts`目录中[^2]**。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923125356.png)

最新的设备都已经采用设备树的管理方式。我们曾经以为设备树很复杂，但是始终要明白一点，所有的设计都是希望设计和采用一种更符合人性的方式“舒服、便于使用、好管理、去掉重复工作”。设备树也是同样的。本节的目的：

* 了解设备树的概念和性质
* 了解设备树底层的工作原理
* 了解设备树的属性和如何使用

# 1. 设备树基本概念

## 1.1 设备树设计思想
**设备树（Device Tree），是一种数据结构，用来动态描述设备的ARCH级信息（SoC上）、Board级（SPI/I2C等）等设备**。Linus指出痛点是因为ARM生态的问题，ARM的生态鼓励厂家自己做SoC，不像是intel固定几个架构，层出不穷的ARM SoC，选择的CPU数量、型号、中断控制器、SRAM配置，这些每个厂家甚至厂家内部的产品可能都有很大差异。更不用说，在PCB板级，CPU外围配置的各种内存、传感器、显卡这些部件了。设备树的使命就是能够来把这些信息描述清楚。如图所示，一个精简的设备树案例[^3]。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923130239.png)

这里面要描述CPU有多少CORE，描述了DDR，描述了I2C下面要挂载多少设备，等等等等，设备树能描述的信息远比这些丰富。想一想，这些信息需要以hardcode的方式写入到内核中，实际上可能开发者压根就不感兴趣。设备树可以把这些信息从内核中分离出来，让开发者单独去描述。而且要知道，对于一款芯片的开发者在生态上会有很多的工作者，例如，在ARM厂更注重架构的描述，到了SoC厂商，开发者更注重SoC周围的设备描述，到了应用SoC的OEM，更注重Board层级的设备。设备树就像是一个笔记，从上游开发者传递到下游开发者，而每一位开发者只需要基于上游开发者做增加即可，大大减小了各个层级开发者的关注成本。到这里，我们应该能体会到了设备树的设计思想了。

好吧，我们开始来安利设备树如何实现和使用。

## 1.2 DTS/DTC/DTB

### 1.2.1 工作流程

设备树本质还是以文件作为载体进行描述，这里面设计了三种文件用于完成设备树的作业：
* DTS： 设备树源文件 (ASIC File)
* DTC： 设备树的编译器，把设备树源文件编译成内核可识别的二进制文件，这个工具内核源码已经自带`scripts/dtc`目录下。
* DTB： 编译之后的设备树的二进制文件，喂给内核使用。

整个工作流程如图所示：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220923131504.png" width="60%" />
</div>

Linux的设备树在系统中体现可以通过`tree -a /proc/device-tree`来查看。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923141204.png)


### 1.2.2 DTC编译过程

我们在编译内核的时候可以使用`make all`或者`make dtbs`就可以对设备树进行编译。前者是编译所有的文件包括zImage，.ko驱动以及设备树；后者是仅仅编译设备树文件。

我们看下DTC的makefile文件：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923132118.png)

编译的时候会先编译出DTC，内部依赖这些文件。编译出DTC之后，使用DTC编译DTS文件。那么Linux内核源码中使用`make dtbs`如何找到DTS文件的，以及我们怎么指定我们自己的DTS文件呢？我们来查看一下`arch/arm/boot/dts`中的makefile：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923132525.png)

可以看到Makefile最终是通过`CONFIG_XXXX`这个配置选项来包含进要生成那些dtb文件的。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923132827.png)

在menuconfig上面，是通过这个使能的。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923133015.png)

如果我们自己定义了设备树文件，就需要在我们配置的架构上面加入我们的文件：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923133321.png)

## 1.3 DTS语法

### 1.3.1 头文件
DTS是支持类似于C语言的#include功能的：

```DTS
12 #include <dt-bindings/input/input.h>
13 #include "imx6ull.dtsi"
```
第 12 行，使用“#include”来引用“input.h”这个.h 头文件。
第 13 行，使用“#include”来引用“imx6ull.dtsi”这个.dtsi 头文件。

在.dts 设备树文件中，可以通过“#include”来引用.h、.dtsi 和.dts 文件。只是，我们在编写设备树头文件的时候最好选择.dtsi 后缀。一般.dtsi 文件用于描述 SOC 的内部外设信息，比如 CPU 架构、主频、外设寄存器地址范围，比如 UART、IIC 等等。比如 imx6ull.dtsi 就是描述 I.MX6ULL 这颗 SoC 内部外设情况信息的。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923133909.png)

### 1.3.2 主题语法
我们再来看一下设备树的主体，用一张图来表示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923133449.png)

设备树是采用树形结构来描述板子上的设备信息的文件，每个设备都是一个节点，叫做设备节点，每个节点都通过一些属性信息来描述节点信息，属性就是键—值对。

每个节点都有不同属性，不同的属性又有不同的内容，属性都是键值对，值可以为空或任意的字节流。设备树源码中常用的几种数据形式如下所示：

**①、字符串**
	`compatible = "arm,cortex-a7";`
	上述代码设置 compatible 属性的值为字符串`“arm,cortex-a7”`。

**②、32 位无符号整数**
	`reg = <0>;`
	上述代码设置 reg 属性的值为 0，reg 的值也可以设置为一组值，比如：
	`reg = <0 0x123456 100>;`

**③、字符串列表**
	属性值也可以为字符串列表，字符串和字符串之间采用“,”隔开，如下所示：
	`compatible = "fsl,imx6ull-gpmi-nand", "fsl, imx6ul-gpmi-nand";`
	上述代码设置属性 compatible 的值为`“fsl,imx6ull-gpmi-nand”`和`“fsl, imx6ul-gpmi-nand”`。

#### 特殊节点
在根节点“/”中有两个特殊的子节点：`aliases` 和 `chosen`，我们接下来看一下这两个特殊的子节点。

**chosen 子节点**
chosen 并不是一个真实的设备，chosen 节点主要是为了 uboot 向 Linux 内核传递数据，重点是 `bootargs` 参数。一般.dts 文件中 chosen 节点通常为空或者内容很少。`bootargs`这个并不是在设备树阶段被添加的，而是uboot帮忙进行添加的，在uboot的代码里可以找到这一步骤。(在uboot里可以查找fdt_chosen这个函数)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923141915.png)

输入 cat 命令查看 bootargs 这个文件的内容：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923141944.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923142046.png)

### 1.3.3 标准属性
设备树的标准属性包含：
* **compatible**： 描述兼容性，平台驱动会以此做比较
* **model**：描述设备模块信息
* **status**：设备状态，okay/disabled/fail/fail-xxx
* **`#address-cells`和`#size-cells`**：描述子节点地址信息
* **reg**：描述设备地址空间资源信息
* **range**：地址映射转换表（子地址、父地址、长度）
* **name**：记录节点名称，name属性已经被弃用。
* **device_type**：IEEE1275会用这个属性，用于描述FCode。只用于CPU或mem节点。

我们这里以几个重要的性质拿出来讲讲。

#### compatible
compatible 属性也叫做“兼容性”属性，这是非常重要的一个属性！compatible 属性的值是一个字符串列表，compatible 属性用于将设备和驱动绑定起来。字符串列表用于选择设备所要使用的驱动程序，compatible 属性的值格式如下所示：

`"manufacturer,model"`

其中 manufacturer 表示厂商，model 一般是模块对应的驱动名字。比如 imx6ull-alientekemmc.dts 中 sound 节点是 I.MX6U-ALPHA 开发板的音频设备节点，I.MX6U-ALPHA 开发板上的音频芯片采用的欧胜(WOLFSON)出品的 WM8960，sound 节点的 compatible 属性值如下：

`compatible = "fsl,imx6ul-evk-wm8960","fsl,imx-audio-wm8960";`

一般驱动程序文件都会有一个 OF 匹配表，此 OF 匹配表保存着一些 compatible 值，如果设备节点的 compatible 属性值和 OF 匹配表中的任何一个值相等，那么就表示设备可以使用这个驱动。比如在文件 imx-wm8960.c 中有如下内容：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923134951.png)

#### 设备树匹配方法

设备树的匹配方法是依赖于设备驱动的`dt_compat`属性。

没有设备树之前也是需要检测兼容性的，但是使用的machine id，使用设备树之后方法就换了。当Linux内核引入设备树以后就不再使用`MACHINE_START`了，而是换为了`DT_MACHINE_START`。`DT_MACHINE_START`也定义在文件`arch/arm/include/asm/mach/arch.h`里面，定义如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923135855.png)

在 `DT_MACHINE_START` 里面直接将.nr 设置为~0。**说明引入设备树以后不会再根据 machine id 来检查 Linux 内核是否支持某个设备了**。

打开文件 arch/arm/mach-imx/mach-imx6ul.c：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923140112.png)

machine_desc 结构体中有个`.dt_compat`成员变量，此成员变量保存着本设备兼容属性，设置`.dt_compat = imx6ul_dt_compat`，imx6ul_dt_compat 表里面有`"fsl,imx6ul"`和`"fsl,imx6ull"`这两个兼容值。只要某个设备(板子)根节点“/”的 compatible 属性值与imx6ul_dt_compat 表中的任何一个值相等，那么就表示 Linux 内核支持此设备。`imx6ull-alientekemmc.dts` 中根节点的 `compatible`属性值如下：

`compatible = "fsl,imx6ull-14x14-evk"`, `"fsl,imx6ull"`;

其中`fsl,imx6ull`与 imx6ul_dt_compat 中的`fsl,imx6ull`匹配，因此 I.MX6U-ALPHA 开发板可以正常启动 Linux 内核。如果将 imx6ull-alientek-emmc.dts 根节点的 compatible 属性改为其他的值，比如：

`compatible = "fsl,imx6ull-14x14-evk"`, **`"fsl,imx6ullll"`**

重新编译DTS，并且启动Linux内核，kernel就没有办法启动了。在 uboot 输出 Starting kernel…以后就再也没有其他信息输出了。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923140422.png)

匹配过程可以如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923140543.png)

#### Linux内核解析dtb
Linux 内核在启动的时候会解析 DTB 文件，然后在/proc/device-tree 目录下生成相应的设备树节点文件。接下来我们简单分析一下 Linux 内核是如何解析 DTB 文件的：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923142154.png)

## 1.4 绑定信息文档
设备树是用来描述板子上的设备信息的，不同的设备其信息不同，反映到设备树中就是属性不同。那么我们在设备树中添加一个硬件对应的节点的时候从哪里查阅相关的说明呢？在Linux 内核源码中有详细的.txt 文档描述了如何添加节点，这些.txt 文档叫做绑定文档，路径为：Linux 源码目录`/Documentation/devicetree/bindings`，

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923142350.png)

比如我们现在要想在 I.MX6ULL 这颗 SoC 的 I2C 下添加一个节点，那么就可以查看Documentation/devicetree/bindings/i2c/i2c-imx.txt，此文档详细的描述了 I.MX 系列的 SOC 如何在设备树中添加 I2C 设备节点，文档内容如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923142435.png)

算是一个SoC厂给的一些示例可以供我们参考的。

## 1.5 设备树常用 OF 操作函数
设备树描述了设备的详细信息，这些信息包括数字类型的、字符串类型的、数组类型的，我们在编写驱动的时候需要获取到这些信息。比如设备树使用 reg 属性描述了某个外设的寄存器地址为 0X02005482，长度为 0X400，我们在编写驱动的时候需要获取到 reg 属性的，该如何获取呢？

Linux内核给我们提供一些接口函数来提取设备树上的信息。这一系列的函数都有一个统一的前缀“of_”，所以在很多资料里面也被叫做 OF 函数。这些 OF 函数原型都定义在 include/linux/of.h 文件中

这些函数是：
1、of_find_node_by_name 函数
2、of_find_node_by_type 函数
3、of_find_compatible_node 函数
4、of_find_matching_node_and_match 函数
5、of_find_node_by_path 函数
6、of_get_parent 函数
7、of_get_next_child 函数
8、of_find_property 函数
9、of_property_count_elems_of_size 函数
....等等等

这些函数有很强的可读性。参考，关于of函数的使用：
https://github.com/wifialan/drivers/blob/master/device_tree_i2c/device_tree_node_transfer

# 2、设备树实例[^2]

## **i.MX6ULL 内部框图** 

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620.png)

## **如何寻找开发板对应的设备树文件** 

直接查看`arch/arm/boot/dts/Makefile`文件，在该文件中查找即可，比如 imx6ull 相关部分如下：

```javascript
dtb-$(CONFIG_SOC_IMX6ULL) += \
        imx6ull-14x14-ddr3-arm2.dtb \
        imx6ull-14x14-ddr3-arm2-adc.dtb \
        imx6ull-14x14-ddr3-arm2-cs42888.dtb \
        imx6ull-14x14-ddr3-arm2-ecspi.dtb \
        imx6ull-14x14-ddr3-arm2-emmc.dtb \
        imx6ull-14x14-ddr3-arm2-epdc.dtb \
        imx6ull-14x14-ddr3-arm2-flexcan2.dtb \
        imx6ull-14x14-ddr3-arm2-gpmi-weim.dtb \
        imx6ull-14x14-ddr3-arm2-lcdif.dtb \
        imx6ull-14x14-ddr3-arm2-ldo.dtb \
        imx6ull-14x14-ddr3-arm2-qspi.dtb \
        imx6ull-14x14-ddr3-arm2-qspi-all.dtb \
        imx6ull-14x14-ddr3-arm2-tsc.dtb \
        imx6ull-14x14-ddr3-arm2-uart2.dtb \
        imx6ull-14x14-ddr3-arm2-usb.dtb \
        imx6ull-14x14-ddr3-arm2-wm8958.dtb \
        imx6ull-14x14-evk.dtb \
        imx6ull-14x14-evk-btwifi.dtb \
        imx6ull-14x14-evk-emmc.dtb \
        imx6ull-14x14-evk-gpmi-weim.dtb \
        imx6ull-14x14-evk-usb-certi.dtb \
        imx6ull-9x9-evk.dtb \
        imx6ull-atk-emmc.dtb \
        imx6ull-9x9-evk-btwifi.dtb \
        imx6ull-9x9-evk-ldo.dtb
```

可以看到所有 imx6ull 的开发板，其中我们移植的开发板为`imx6ull-atk-emmc.dtb`，本文就以该设备树文件为例，讲述设备树语法。

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160441899.png)

## **1. skeleton 描述文件** 

查看文件`arch/arm/boot/dts/skeleton.dtsi`，内容非常简洁，只定义了根节点：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160409730.png)

## **2. imx6ull 芯片级描述文件（通用）** 

不同的 imx6ull 开发板都是使用 imx6ull 这颗处理器芯片，**而 imx6ull soc 芯片级的描述是固定的，通常这个也是由芯片厂商提供**。

查看`imx6ull.dtsi`文件，整体框架如下：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160409768.png)

接下来我们逐个分析。

### **2.1. 根节点的补充**

该文件引用的`skeleton.dtsi`文件中，**已经定义了根节点，如果再次定义根节点，其中的内容将作为对根节点的补充**。

在该描述文件中，挂在根节点上的子节点有：aliases、cpus、intc、clocks、soc。

（1）aliases 节点

aliases 节点用来定义一个或多个别名属性，按照约定，该节点应该在根节点上。

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160409809.png)

（2）cpus 节点**所有的设备树都需要 cpus 节点**，用来描述系统的 CPU 信息。

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160454580.png)

i.MX6ULL 是单核处理器，因此只有一个`/cpus/cpu*`子节点，用来表示某一个具体 CPU 核的信息，其中有以下属性：

-   compatible：
-   device_type：描述设备类型
-   reg
-   clock-latency
-   operating-points
-   fsl,soc-operating-points
-   fsl,low-power-run
-   clocks
-   clock-names

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160409891.png)

（3）intc 节点

（4）clocks 节点

（5）soc 节点

soc 节点中，描述了 i.MX6ULL 片上的总线和全部外设：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160502532.png)

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160409966.png)

### **2.2. aips2 总线节点分析**

```javascript
aips2: aips-bus@02100000 {
   compatible = "fsl,aips-bus", "simple-bus";
   #address-cells = <1>;
   #size-cells = <1>;
   reg = <0x02100000 0x100000>;
   ranges;

   //一堆外设子节点，省略...
};
```

复制

aips2 节点的属性有：

-   compatible：兼容性
-   \#address-cells：**子节点**reg 属性中地址字段所占用的单元格数量，占用 1 个 u32
-   size-cells：**子节点**reg 属性值的长度所占用的单元格的数量，占用 1 个 u32
-   reg：寄存器起始地址 0x02100000，长度 0x100000
-   ranges：空

### **2.3. i2c 控制器节点分析**

i2c 控制器是挂在 aips2 总线上的，对应到设备树中，i2c 控制器节点挂在 aips2 节点上，描述代码如下：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160512250.png)

以 i2c1 节点为例，标签是 i2c1，节点名称是 i2c，寄存器起始地址是 0x021a0000，有如下属性：

-   \#address-cells：**子节点**reg 属性中地址字段所占用的单元格数量，占用 1 个 u32
-   size-cells：**子节点**reg 属性值的长度所占用的单元格的数量，占用 0 个 u32
-   compatible：兼容性，fsl,imx6ul-i2c 和 fsl,imx21-i2c
-   reg：寄存器，起始地址是 0x021a0000，长度是 0x4000
-   interrupts：中断，不了解 A7 的中断控制器，看不懂
-   clocks：时钟源，clks 节点的 IMX6UL_CLK_I2C1 这个时钟
-   status：节点状态，禁用

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160410047.png)

## **3. imx6ull ATK 开发板描述文件** 

`imx6ull-atk-emmc.dts`这个文件的大概框架如下。

### **3.1. 版本**

```javascript
/dts-v1/;
```

复制

### **3.2. 根节点的补充**

框架如下：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160519261.png)

其中 chosen 节点是 uboot 用来向内核传递参数，内容如下：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160410131.png)

### **3.3. 子节点的补充**

**根节点之后，使用引用符`&`来对`imx6ull.dtsi`文件中定义的子节点进行补充，用来描述开发板的具体配置，这个也是主要需要适配修改的文件。**

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160410170.png)

### **3.4. 磁力计 mag3110 节点分析**

在 NXP 官方开发板上，磁力计 mag3110 是接在 i2c1 总线控制器上的，对应到设备树中，磁力计节点挂在 i2c1 控制器节点上，如下。

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160410208.png)

可以看到，i2c1 节点的补充描述中，就描述了 i2c1 控制器上所连接的设备。

i2c1 节点的补充属性有：

-   clock-frequency：i2c 控制器时钟频率，100khz
-   pinctrl-names：
-   pinctrl-0：
-   status：设备状态，就绪

**i2c 控制器上接了两个设备，一个是 mag3110 磁力计，一个是 fxls8471 加速度计。注意，在描述节点时，@后面的地址变为了 i2c 总线的设备地址，mag3110 的 i2c 从机地址是 0e，fxls8471 的 i2c 从机地址是 1e**。

![img](https://raw.githubusercontent.com/carloscn/images/main/typora1620-20220923160529363.png)

至此，imx6ull 设备树分析完成。

# 3. 设备树实例[^4][^5]

参考[^4][^5] : I2C总线的设备树实例

# Ref
[^1]:[Linus Torvalds - Re: [GIT PULL] omap changes for v2.6.39 merge window](https://lkml.org/lkml/2011/3/17/492)
[^2]:[设备树实例解析](https://cloud.tencent.com/developer/article/2008640)
[^3]:[OSD335x Lesson 2: Linux Device Tree](https://octavosystems.com/app_notes/osd335x-design-tutorial/osd335x-lesson-2-minimal-linux-boot/linux-device-tree/)
[^4]:[Linux 设备树学习——基于i2c总线分析](https://blog.csdn.net/multimicro/article/details/103129546?spm=1001.2014.3001.5502)
[^5]:[wifialan - **device_tree_i2c**](https://github.com/wifialan/drivers/tree/master/device_tree_i2c)