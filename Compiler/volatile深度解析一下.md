# Compiler optimization and the volatile keyword

【每天学个编译器小知识】

面试的时候，很多面试官，en，会问，volatile有啥用啊？今天我们就来了基于ARM架构，聊聊在ARMv7/v8架构上，volatile到底是什么东西。

在编译器配置较高优化级别的时候，可能程序中会出现一些问题，这些问题在较低的优化不太明显，volatile易失性限定符可以告诉编译器，不要对该变量做过多的优化。这种优化可以通过以下场景复现：

* 在轮询的时候，代码可能会卡在循环中
* 多线程代码出现奇怪的行为
* 一些人为的延迟代码被优化掉。

volatile标记会被编译器识别，可以在实现外部随时修改变量，例如操作系统、其他执行线程（中断、信号处理）或硬件，这样就间接给这个变量增加了一个保护，外部随时修改的变量是可能被外面更改的，因此每次在代码中引用该值的时候，需要从内存中去读取，而不是缓存到寄存器中。（将某个变量缓存到寄存器里面是ARM处理器的一种优化手段）。在实现睡眠和计时延迟的上下文，需要将变量声明为volatile告诉编译器需要特定类型的行为。

The two versions of the routine differ only in the way that `buffer_full` is declared. The first routine version is incorrect. Notice that the variable `buffer_full` is not qualified as volatile in this version. In contrast, the second version of the routine shows the same loop where `buffer_full` is correctly qualified as volatile.

| Nonvolatile version of buffer loop                           | Volatile version of buffer loop                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20220413213827285](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220413213827285.png) | ![image-20220413213842311](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220413213842311.png) |

[Table 13](https://developer.arm.com/documentation/dui0472/c/compiler-coding-practices/compiler-optimization-and-the-volatile-keyword?lang=en) shows the corresponding disassembly of the machine code produced by the compiler for each of the sample versions in [Table 8](https://developer.arm.com/documentation/dui0472/c/compiler-coding-practices/optimization-of-loop-termination-in-c-code?lang=en), where the C code for each implementation has been compiled using the option `-O2`.

![image-20220413214136517](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220413214136517.png)

如果采用-O2的优化等级，还是看出差异了，左边的r1寄存器被load了r1的值，之后就没有再load了，然后就在L1.12的分支里面无限循环。而在加了volatile的版本，每次都要LDR这个值。

要注意以下情况需要volatile：

* 访问内存映射的外围设备
* 在多个线程中共享的全局变量
* 在中断程序或者信号处理中访问的全局变量