# Linux应用调试（三）工具之Valgrind

援引[Linux应用调试（一）方法、技巧和工具 - 综述.md](https://gist.github.com/carloscn/4037f1ffd881e8eac29e8511e6ca1431) ：*软件工具->Linux User-> 动态 -> valgrind*。

C/C++相比其他高级编程语言，具有指针的概念，指针即是内存地址。C/C++可以通过指针来直接访问内存空间，效率上的提升是不言而喻的，是其他高级编程语言不可比拟的；比如访问内存一段数据，通过指针可以直接从内存空间读取数据，避免了中间过程函数压栈、数据拷贝甚至消息传输等待。指针是C/C++的优势，但也是一个隐患，指针是一把双刃剑，由于内存交给了程序员管理，这是存在隐患的；人总会有疏忽的时候，如果申请了内存，一个疏忽**忘记释放**了，未释放的内存将不能被系统再申请使用，即是一直占用又不能使用，俗称内存泄露；久而久之，系统长时间运行后将申请不到内存，系统上所有任务执行失败，只能重启系统。内存交给了程序员管理，除了内存泄露的情况外，还可能引起其他的问题，总结起来包括如下几点：

-   **访问没有申请内存的空指针（空指针）**
-   **访问已释放内存的指针（野指针）**
-   **内存越界访问**
-   **内存泄露，申请了内存没有释放**
-   **重复释放内存**

虽然动态内存（这里默认指的是堆上的动态内存，由程序员管理；栈上的动态内存由系统管理）存在一定的隐患，可靠性取决于程序员的认真和细心，但因为其具有优良的灵活性和高效率以及内存复用，在实际应用中，对于一些动态数据交互场合，动态内存往往是首选。因此，一方面除了程序员本身的谨慎使用，另一方面也涌现出各类“内存异常“的检测工具，毕竟当代码量数以万计时，依靠人力去检查是不切实际的。

# 1 Valgrind简介
Valgrind是一款基于linux平台开源的**内存检测工具集合**，功能强大，使用广泛。利用Valgrind工具，在检测出文章开头提及的动态内存隐患，即是Valgrind工具中的内存检测组件Memcheck， Memcheck支持的功能包括：
-   使用空指针
-   使用野指针
-   内存空间访问越界
-   内存空间未释放
-   内存空间重复释放
-   内存空间申请和释放不匹配

目前Valgrind支持很多平台[^1]：
-   **x86/Linux:** up to and including SSSE3, but not higher -- no SSE4, AVX, AVX2. This target is in maintenance mode now..
-   **AMD64/Linux:** up to and including AVX2. This is the primary development target and tends to be well supported.
-   **PPC32/Linux, PPC64/Linux, PPC64LE/Linux:** up to and including Power8.
-   **S390X/Linux:** supported.
-   **ARM/Linux:** supported since ARMv7.
-   **ARM64/Linux:** supported for ARMv8.
-   **MIPS32/Linux, MIPS64/Linux:** supported.
-   **X86/FreeBSD, AMD64/FreeBSD**: supported since FreeBSD 11.3.
-   **X86/Solaris, AMD64/Solaris, X86/illumos, AMD64/illumos**: supported since Solaris 11.
-   **X86/Darwin (10.5 to 10.13), AMD64/Darwin (10.5 to 10.13):** supported.
-   **ARM/Android, ARM64/Android, MIPS32/Android, X86/Android:** supported.

编译到ARM平台可以参考[^2]。

## 1.1 Valgrind延伸工具

Valgrind功能强大，是一个工具集合，除了Memcheck工具外，还提供Cachegrind、Helgrind、Callgrind、Massif工具。

#### Memcheck

Memcheck 是一个内存错误检测器。它可以检测C和C++程序中常见的下列问题：

-   访问不应该访问的内存，例如溢出堆外、溢出堆栈顶部以及在释放内存后访问内存。
-   使用未定义值，即未初始化的值，或从其他未定义值派生的值。
-   堆内存释放不正确，例如堆块的双重释放，或者 malloc/new/new[]与free/delete/delete[] 的使用不匹配。
-   memcpy 和相关函数中的 src 和 dst 指针区域存在重叠。
-   向内存分配函数的 size 参数传递一个可疑值（可能是负值）。
-   内存泄漏。

使用形式为：

`./valgrind --tool=memcheck --leak-check=full [memcheck options] your-program [program options]`

#### Cachegrind
与gprof类似的分析工具，但它对程序的运行观察更是入微，能给我们提供更多的信息。和gprof不同，它不需要在编译源代码时附加特殊选项，但加上调试选项是推荐的。Callgrind收集程序运行时的一些数据，建立函数调用关系图，还可以有选择地进行cache模拟。在运行结束时，它会把分析数据写入一个文件。callgrind_annotate可以把这个文件的内容转化成可读的形式。

`./valgrind --tool=cachegrind [cachegrind options] your-program [program options]`

#### Cachegrind
Cachegrind 模拟程序如何与计算机的缓存层次结构和（可选）分支预测器交互。它模拟具有独立的一级指令和数据缓存（I1和D1）的机器，后面接着是统一的二级缓存（L2）。这完全符合许多现代机器的配置。然而，一些现代机器有三到四级缓存。对于这些机器（在 Cachegrind 可以自动检测缓存配置的情况下），Cachegrind 模拟第一级和最后一级缓存。这种选择的原因是，最后一级缓存对运行时的影响最大，因为它屏蔽了对主内存的访问。此外，一级缓存通常具有低关联性，因此模拟它们可以检测到代码与该缓存交互不良的情况（例如，按行长为2的幂次遍历矩阵列）。

`./valgrind --tool=cachegrind [cachegrind options] your-program [program options]`

#### Helgrind

POSIX pthreads 中的主要抽象是：一组共享公共地址空间的线程、线程创建、线程 join、线程退出、互斥（锁）、条件变量（线程间事件通知）、读写锁、自旋锁、信号量和屏障。

Helgrind 可以如下检测三类错误：

1.  POSIX pthreads API 的错误使用。
2.  由锁顺序问题引起的潜在死锁。
3.  存在数据竞争，即在没有足够的锁定或同步的情况下访问内存。

`./valgrind --tool=helgrind [helgrind options] your-program [program options]`

#### Callgrind
Callgrind 是一个分析工具，它将程序运行中函数之间的调用历史记录记录为调用图。默认情况下，收集的数据由执行的指令数量、指令与源行的关系、函数之间的调用方/被调用方关系以及调用次数组成。或者，缓存模拟或分支预测（类似于 Cachegrind）可以生成有关应用程序运行时行为的进一步信息。

`./valgrind --tool=callgrind [callgrind options] your-program [program options]`

会生成一个文件：`callgrind.out.<pid>`

该文件可以拿到服务器上用 **callgrind_annotate**（其位置就是在 valgrind/output/bin 下） 去解析：

```bash
./callgrind_annotate [options] callgrind.out.<pid>
```

#### Massif
Massif 是一个堆分析器。它测量程序使用的堆内存量。这既包括有用的空间，也包括对齐目的分配的额外字节。它还可以测量程序堆栈的大小，尽管默认情况下它不会这样做。堆分析可以帮助您减少程序使用的内存量。在具有虚拟内存的现代计算机上，这提供了以下好处：

-   它可以加快程序的速度——更小的程序可以更好地与机器的缓存交互，避免分页。
-   如果程序使用大量内存，这将减少耗尽机器交换空间的机会。

此外，有些空间泄漏是传统的泄漏检查程序（如 Memcheck）无法检测到的，这是因为内存从来没有真正丢失过，因为仍有指针指向它，但它没有使用。存在该种泄漏的程序可能会不必要地增加它们在一段时间内使用的内存量。Massif 可以帮助识别这些泄漏。重要的是，Massif 不仅告诉你程序使用了多少堆内存，还提供了非常详细的信息，指示程序的哪些部分负责分配堆内存。

使用形式为：

```bash
./valgrind --tool=massif [massif options] your-program [program options]
```

会生成一个文件：`massif.out.<pid>`

该文件可以拿到服务器上用 **ms_print**（其位置就是在 valgrind/output/bin 下） 去解析：

```bash
./ms_print [options] massif.out.<pid>
```

## 1.2 Valgrind使用
Valgrind内存检测的原理是，Valgrind实现一个程序运行的虚拟环境，待检测的应用程序在Valgrind的虚拟环境中运行，Valgrind能够实时监测程序中内存申请、使用、释放的情况，并记录一些正常运行的有效信息和可能存在内存异常的信息，输出到终端或者用户指定的文件中。

### 1.2.1 Valgrind安装

#### ubuntu

Valgrind工具实用且使用广泛，大多数主流linux系统发行版都已集成Valgrind工具，在终端输入`“valgrind”`，如系统没有安装Valgrind工具会提示以下安装信息。

`sudo apt install valgrind`

安装完成，输入`“valgrind”`命令，提示以下信息，表示安装成功。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911190907.png)

#### ARM embedded Linux[^3]
##### 下载Code
下载地址：https://valgrind.org/downloads/ ，我这里下载的是[valgrind 3.19.0 (tar.bz2)](https://sourceware.org/pub/valgrind/valgrind-3.19.0.tar.bz2)

解压：`tar -xvf valgrind-3.19.0.tar.bz2`

或者直接从GITHUB上拉：`git clone https://sourceware.org/git/valgrind.git --depth=1`

##### 修改code
1.  运行`./autogen.sh`
2. 修改 configure，将 **armv7*** 改成 **arm*** 如下：
    ![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911193648.png)
3. `./configure CC=arm-linux-gnueabihf-gcc CPP=arm-linux-gnueabihf-cpp CXX=arm-linux-gnueabihf-g++ --prefix=/opt/valgrind --host=arm-linux-gnueabihf` ，注意：***--prefix=/opt/Valgrind指定的目录要与开发板上放置的目录一致，不然运行valgrind时可能会出现“valgrind: failed to start tool 'memcheck' for platform 'arm-linux': No such file or directory”错误***。
4. `make -j8`
5. `sudo make install`

##### 移植

1.  将生成的文件拷贝到共享目录的相应位置，板子上布局如下：
     ![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911213917.png)
     `scp -r /opt/valgrind root@192.168.31.210:/opt`
2. 在device端执行：`export VALGRIND_LIB=/opt/valgrind/libexec/valgrind`，不配置VALGRIND_LIB路径，还会报*valgrind: failed to start tool 'memcheck' for platform 'arm-linux': No such file or directory*的错误，也可以放在.bashrc里面。注意2，*是libexec的路径，而不是lib路径*。
3. 接着就可以运行了：`./valgrind --tool=memcheck --leak-check=yes ./test.elf `
     ![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911224425.png)
4. 上面显示fatal error at startup。Valgrind 需要链接 not stripped 过的 `ld*.so/libc*.so/libdl*.so`，这些可以从相应toolchain release package里面找，`/opt/cross-compile/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib` 找如下几个：
    ![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911225035.png)
    `scp -r ld-* libc* libdl* root@192.168.31.210:/opt/valgrind/lib/valgrind`
    配置LD_LIBRARY  `export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/valgrind/lib/valgrind`
5. 可以通过设置 LD_LIBRARY_PATH 去指定 libc*.so/libdl*.so 的位置，但是对于动态装载库 `ld*.so` 则无效。**动态装载库的位置既不是由系统配置指定，也不是由环境参数决定，而是由 ELF 可执行文件决定**。在动态链接的 ELF 可执行文件中，有一个专门的 section 叫 **.interp**，该 section 可通过 **readelf** 查看：
    ![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911225601.png)
   通过 **patchelf** 直接改写 ELF 可执行文件中的 .interp section：
    ![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911230027.png)


### 1.2.2 使用说明

-   待检测的程序编译时需加入`"-g"`，保留调试信息
-   检测命令格式：`valgrind --tool=memcheck --leak-check=full ./file`，`–leak-check=full`表示检测所有内存泄露
-  检测记录除了输出到终端，还可以输出到文件，这样可以监测程序长时间运行后出现异常，通过查阅文件记录分析问题
-   更多的使用说明或者忘记命令时，可以执行`valgrind -h`查看帮助信息

# 2. 示例
## 2.1 检测程序使用空指针
```C
#include <stdio.h>
// valgrind --tool=memcheck --leak-check=full ./test_null_pointer.elf
int main(int argc, char * argv [ ])
{
	int *p = NULL;

    printf("address [0x%p]\r\n", p);
	*p = 0;

    return 0;
}
```
编译：
`gcc -g test_null_pointer.c -o test_null_pointer.elf`
运行检测:
`valgrind --tool=memcheck --leak-check=full ./test_null_pointer.elf`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912091219.png)

## 2.2 检测内存越界
```C
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char * argv [ ])
{
	int *p = NULL;

	p = malloc(4);
	if (p == NULL)
	{
		perror("malloc failed");
	}
    printf("address [0x%p]\r\n", p);
	p[4] = 0;
	free(p);

    return 0;
}
```
编译：
`gcc -g test_over_array.c -o test_over_array.elf`
运行检测:
`valgrind --tool=memcheck --leak-check=full ./test_over_array.elf`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912091447.png)

## 2.3 检测程序内存空间未释放
```C
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char * argv [ ])
{
	int *p = NULL;

	p = malloc(4);
	if (p == NULL)
	{
		perror("malloc failed");
	}
    printf("address [0x%p]\r\n", p);

    return 0;
}
```
编译：
`gcc -g test_mem_leak.c -o test_mem_leak.elf`
运行检测:
`valgrind --tool=memcheck --leak-check=full ./test_mem_leak.elf`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912091744.png)

## 2.4 double-free
```C
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char * argv [ ])
{
	int *p = NULL;

	p = malloc(4);
	if (p == NULL)
	{
		perror("malloc failed");
	}
    printf("address [0x%p]\r\n", p);
	free(p);
	free(p);

    return 0;
}
```
编译：
`gcc -g test_double_free.c -o test_double_free.elf`
运行检测:
`valgrind --tool=memcheck --leak-check=full ./test_double_free.elf`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912091935.png)

## 2.5 释放对象错误
```CPP
#include <stdio.h>
#include <stdlib.h>
#include <iostream>

int main(int argc, char * argv [ ])
{
	int *p = NULL;

	p = (int*)malloc(sizeof(int));
	if (p == NULL)
	{
		perror("malloc failed");
	}
    printf("address [0x%p]\r\n", p);
	delete p;

    return 0;
}
```
编译：
`g++ -g test_wrong_free_obj.cpp -o test_wrong_free_obj.elf`
运行检测：
`valgrind --tool=memcheck --leak-check=full ./test_wrong_free_obj.elf`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912092253.png)

# 3. 小结
内存泄露检测包括动态内存使用的规范性，根本的解决办法是程序员保持良好的编码习惯，使用动态内存时谨慎考虑，保证申请与释放的必然性。因为，一些隐晦的问题可能需要在特定条件下才会引起内存泄露，依赖于检测工具也是需要长时间运行软件才能发现。

Valgrind—memcheck工具更多是用于检测内存泄露，空指针、野指针、内存越界、重复释放等问题，会引系统段错误（Valgrind），使用GDB结合系统产生的core dump文件，也能快速定位到调用位置。而内存泄露不会立即导致系统异常，只有运行一定时间后系统申请不到内存时才会引起异常。因此，借助Valgrind—memcheck工具来检测内存泄露是一个高效的方法之一。

# Ref
[^1]:[# Supported Platforms](https://valgrind.org/info/platforms.html)
[^2]:[# [在ARM Linux 使用 Valgrind](https://www.cnblogs.com/xuanyuanchen/p/5761315.html)](https://www.cnblogs.com/xuanyuanchen/p/5761315.html)
[^3]:[Valgrind arm平台交叉编译使用介绍](https://juejin.cn/post/6844904196051845127)