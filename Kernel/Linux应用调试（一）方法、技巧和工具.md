# Linux应用调试（一）方法、技巧和工具 - 综述

如果把调试系统化，截止到目前可以总结为：

![](https://raw.githubusercontent.com/carloscn/images/main/typora%E8%B0%83%E8%AF%95_%E7%BB%BC%E8%BF%B0.svg)

搞电子都知道，电路不是焊接出来的，是调试出来的。程序员也一定认同，程序不是写出来的，是调试出来的。那么调试工具就显得尤为重要，linux作为笔者重要的开发平台，在linux中讨论调试工具主要是为那些入门者提供一些帮助。**调试工具能让我们能够监测、控制和纠正正在运行的程序**。我们在运行一些程序的时候，可能被卡住或出现错误，或者运行过程或结果，没能如我们预期，此时，最迫切需要明白究竟发生了什么。为了修复程序，剖析和了解程序运行的细节， 调试工具就成为了我们的必备工具，工于善其事，必先利其器。在Linux下的用户空间调试工具主要有系统工具和专门调试工具：`print `打印语句，这是新手最常用的，也是最不提倡使用的；查询 (`/proc, /sys` 等)系统的虚拟文件查看，这个方法有局限性；**跟踪 `(strace/ltrace)`工具**使用这个比较普遍，值得提倡；`Valgrind` (`memwatch`)内存排除工具，在内存排除方面比较独到，是内存排错的法宝；`GDB`大名鼎鼎的程序调试工具，这个是个全能的工具，没有完不成的，只有你不知道的。

然动态内存存在隐患，但有时候又不得不使用，内存泄露也就成为一个必须杜绝的敏感问题；有需求就会有相应的解决方法，目前已存在许多优秀的内存泄露检测工具，除了我们常用的Valgrind外，还有mtrace、dmalloc、memwatch等优秀工具。

|工具	| 描述 |
|------|-------|
|valgrind|一个强大开源的程序检测工具集合|
|mtrace	|GNU扩展, 用来跟踪malloc, mtrace为内存分配函数（malloc, realloc, memalign, free）安装hook函数|
|dmalloc|用于检查C/C++内存泄露(leak)的工具，即检查是否存在直到程序运行结束还没有释放的内存,以一个运行库的方式发布|
|memwatch|和dmalloc一样，它能检测未释放的内存、同一段内存被释放多次、位址存取错误及不当使用未分配之内存区域|
|mpatrol|一个跨平台的 C++ 内存泄漏检测器|
|dbgmem	| |
|Electric Fence| |

# 1. 调试常用方法

## 1.1 'print' 语句（采样法）
这是一个基本的调试问题的方法。 我们在程序中怀疑的地方插入print语句来了解程序的运行流程控制流和变量值的改变。 这是一个最简单的技术， 它的缺点。 需要进行程序编辑，添加'print'语句，必须重新编译，重新运行来获得输出。若需要调试的程序比较大，这将是一个耗时费力的方法。

## 1.2 查询

### 1.2.1 查看线程信息

在某些情况下，我们需要弄清楚在一个运行在内核中的进程的状态和内存映射。为了获得这些信息，我们不需要在内核中插入任何代码。 相反，可以用 `/proc `文件系统。在`/proc`的伪文件系统，保留系统启动运行就收集的运行时信息 (cpu信息, 内存容量等)。

`ls -l /proc`的输出结果，通过对 系统中运行的每一个进程在`/proc`文件系统中有一个以进程id命名的项。每个进程的细节信息可以在进程id对应的目录下的文件中获得。也可以`ls /proc/pid`的输出

**解释/proc文件系统内的所有条目超出了本文的范围。一些有用的列举如下**：

-   `/proc/cmdline` -> 内核命令行
-   `/proc/cpuinfo` -> 关于处理器的品牌，型号信息等
-   `/proc/filesystems` -> 文件系统的内核支持的信息
-   `/proc/<pid>/cmdline` -> 命令行参数传递到当前进程
-   `/proc/<pid>/mem `-> 当前进程持有的内存
-   `/proc/<pid>/status` -> 当前进程的状态

### 1.2.2 查新系统信息
`uname -a`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912093420.png)

### 1.2.3 查看文件占用空间
命令：`size file-name`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912093736.png)
*   text，代码段，代码和常量区
*   data，数据段，存放已经初始化的全局变量、全局或局部静态变量
*   bss，bss段，存放未初始化的全局变量（一般以0填充）
*   dec， decimal，text、data、bss三者之和的十进制格式
*   hex，hexadecimal，text、data、bss三者之和的十六进制格式

### 1.2.4 查看文件所需动态库
命令：`ldd file-name`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912093912.png)

### 1.2.5 查看文件符号表
命令： `nm file-name |grep [fun/variable]`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912094053.png)

### 1.2.6 文件空间优化
命令：`strip file-name`  
该命令可以优化删除执行文件的一些符号信息、调试信息（gdb），节约空间。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220912094259.png)

## 1.3 跟踪

`strace`的和`ltrace`是两个在Linux中用来追踪程序的执行细节的跟踪工具。

### strace:

**strace拦截和记录系统调用及其接收的信号**。对于用户，它显示了系统调用、传递给它们的参数和返回值。strace的可以附着到已在运行的进程或一个新的进程。它作为一个针对开发者和系统管理员的诊断、调试工具是很有用的。它也可以用来当做一个通过跟踪不同的程序调用来了解系统的工具。这个工具的好处是不需要源代码，程序也不需要重新编译。

使用strace的基本语法是：

`> strace 命令`

strace有各种各样的参数。可以检查看strace的手册页来获得更多的细节。strace的输出非常长，我们通常不会对显示的每一行都感兴趣。我们可以用`-e expr`选项来过滤不想要的数据。用 '-p pid' 选项来绑到运行中的进程，用'-o'选项，命令的输出可以被重定向到文件。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911111607.png)

strace过滤成只有系统调用的输出。

### ltrace:

`ltrace`跟踪和记录一个进程的动态（运行时）库的调用及其收到的信号。它也可以跟踪一个进程所作的系统调用。它的用法是类似与`strace`。

`> ltrace command`
* '-i' 选项在调用库时打印指令指针。
* '-S' 选项被用来现实系统调用和库调用
* 所有可用的选项请参阅ltrace手册。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911111721.png)

ltrace捕捉'STRCMP'库调用的输出。

### Valgrind
Valgrind是一套调试和分析工具。它的一个被广泛使用的默认工具——'Memcheck'——可以拦截malloc()，new()，free()和delete()调用。换句话说，它在检测下面这些问题非常有用：
-   内存泄露
-   重释放
-   访问越界
-   使用未初始化的内存
-   使用已经被释放的内存等。

它直接通过可执行文件运行。

Valgrind也有一些缺点，因为它增加了内存占用，会减慢你的程序。它有时会造成误报和漏报。它不能检测出静态分配的数组的访问越界问题。

为了使用它，首先请[下载](http://valgrind.org/downloads.html)并安装在你的系统上。可以使用操作系统上的包管理器来安装。

使用命令行安装需要解压缩和解包下载的文件：

`tar -xjvf valgring-x.y.z.tar.bz2`  (where x.y.z is the version number you are trying to install)

进入新创建的目录（的valgrind-XYZ）内运行以下命令：
1.  `./configure`
2.  `make`
3.  `make install`

让我们通过一个小程序(test.c)来理解valgrind怎么工作的:
```C
#include <stdio.h>

void f(void) {
    int x = malloc(10 * sizeof(int));
    x[10] = 0;
}

int main() {
    f();
    return 0;    // 没有对内存进行free，产生了内存泄漏。
}
```
编译程序:
`gcc -o test -g test.c`
现在我们有一个可执行文件叫做'test'。我们现在可以用valgrind来检测内存错误：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911112113.png)

valgrind显示堆溢出和内存泄漏的输出，正如我们在上面看到的消息，我们正在试图访问函数f未分配的内存以及分配尚未释放的内存。

###  GDB
GDB是来自自由软件基金会的调试器。它对定位和修复代码中的问题很有帮助。当被调试的程序运行时，它给用户控制权去执行各种动作， 比如:
-   启动程序
-   停在指定位置
-   停在指定的条件
-   检查所需信息
-   改变程序中的数据 等。

你也可以将一个崩溃的程序coredump附着到GDB并分析故障的原因。GDB提供很多选项来调试程序。 然而，我们将介绍一些重要的选择，来感受如何开始使用GDB。如果你还没有安装GDB，可以在这里下载：[GDB官方网站](https://www.gnu.org/software/gdb/download/)。

```C
#include <stdio.h>
int divide()
{
    int x=5, y=0;    // 除0异常
    return x / y;
}

int main()
{
    divide();
}
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911112400.png)

展示GDB用法的例子：

#### 调用 GDB:

通过在命令行中执行'gdb'来启动gdb:
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911112427.png)

调用 gdb，调用后, 它将等待终端命令并执行，直到退出。如果一个进程已经在运行，你需要将GDB连接到它上面，可以通过指定进程ID来实现。假设程序已经崩溃，要分析问题的原因，则用GDB分析core文件。

#### 启动程序:

一旦你在GDB里面，使用'run'命令来启动程序进行调试。

#### 给程序传参数:

使用'set args'给你的程序传参数，当程序下次运行时将获得该参数。'show args'将显示传递给程序的参数。

#### 检查堆栈:

每当程序停止，任何人想明白的第一件事就是它为什么停止，以及怎么停在那里的。该信息被称为反向跟踪。由程序产生每个函数调用和局部变量，传递的参数，调用位置等信息一起存储在堆栈内的数据块种，被称为一帧。我们可以使用GDB来检查所有这些数据。 GDB从最底层的帧开始给这些帧编号。

-   bt: 打印整个堆栈的回溯
-   bt 打印n个帧的回溯
-   frame : 切换到指定的帧，并打印该帧
-   up : 上移'n'个帧
-   down : 下移'n'个帧 ( n默认是1)

#### 检查数据:

程序的数据可以在里面GDB使用'print'命令进行检查。例如，如果'x'是调试程序内的变量，'print x'会打印x的值。

#### 检查源码:

源码可以在GDB中打印。默认情况下，'list'命令会打印10行代码。

-   list : 列出'linenum'行周围的源码
-   list : 从'function'开始列出源码
-   disas : 显示该函数机器代码

#### 停止和恢复程序:

使用GDB，我们可以在必要的地方设置断点，观察点等来停止程序。

-   break : 在'location'设置一个断点。当在程序执行到这里时断点将被击中，控制权被交给用户。
-   watch : 当'expr'被程序写入而且它的值发生变化时GDB将停止
-   catch : 当'event'发生时GDB停止
-   disable : 禁用指定断点
-   enable : 启用指定断点
-   delete : 删除 断点/观察点/捕获点。 如果没有传递参数默认操作是在所有的断点
-   step: 一步一步执行程序
-   continue: 继续执行程序，直到执行完毕

#### 退出 GDB:

用'quit'命令还从GDB中退出。

GDB还有更多的可用选项。里面GDB使用help选项了解更多详情。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220911112533.png)

### 总结

在这篇文章中，我们已经看到不同类型的Linux用户空间的调试工具。总结以上所有内容，如下是什么时候使用该什么的快速指南：

-   基本调试，获得关键变量 - print 语句
-   获取有关文件系统支持，可用内存，CPU，运行程序的内核状态等信息 - 查询 /proc 文件系统
-   最初的问题诊断，系统调用或库调用的相关问题，了解程序流程 – strace / ltrace
-   应用程序内存空间的问题 – valgrind
-   检查应用程序运行时的行为，分析应用程序崩溃 – gdb

# 调试案例：
* [linux下的程序调试方法汇总](https://www.pianshen.com/article/596279446/)
* [定位多线程内存越界问题实践总结](https://www.cnblogs.com/djinmusic/archive/2013/02/04/2891753.html)
* [【代码质量】如何使用Valgrind检测内存泄漏](https://acuity.blog.csdn.net/article/details/106503929)