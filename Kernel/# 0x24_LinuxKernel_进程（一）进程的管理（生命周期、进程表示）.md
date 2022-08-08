# 0x24_LinuxKernel_进程（一）进程的管理（生命周期、进程表示）

在UNIX操作系统下运行的应用程序、服务器以及其他程序都被称为**进程**。每个进程都在CPU的虚拟内存上分配地址空间。每个进程的地址空间都是独立的，因此进程和进程不会意识对方的存在，觉得自己是CPU上唯一运行的进程。从处理器的角度，系统上有几个CPU，就最多能运行几个进程。但Linux是一个多任务的系统，内核为了支持多任务需要在不同的进程之间切换，这样从时间维度的划分造成了多进程同时运行的假象。

内核借助CPU的帮助，负责进程切换，欺骗进程独占了CPU。这就需要在切换进程之前保存进程的所有状态及相关要素，并将进程置于IDLE状态。在进程切换回来之前，这些资源和信息需要恢复到CPU和内存上面。这些资源被成为**进程上下文**。

内核除了要保存进程一些必要信息之外。还需要合理分配每一个进程的时间片，通常重要的进程得到CPU的时间多一点，次要的进程时间少一点。这个时间分配的过程成为**调度**。

本文对于进程的表述的结构如下所示：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86%E4%B8%8E%E8%B0%83%E5%BA%A61.svg" width="100%" /> </div>


# 1. 用户空间定义
这部分介绍一些从用户空间视角来观摩进程的概念。

## 1.1 进程树
Linux对进程的表述采用了一种层次结构，每个进程都依赖于一个父进程。内核启动`init`程序作为第一个进程，该进程负责进一步的系统初始化操作，并显示视提示符和登陆界面。因此`init`进程被称为进程树的***根***，**所有的进程都直接或者间接起源自该进程**。如下使用*pstree*程序的输出所示。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220807124816.png" width="100%" /> </div>

这个树形结构的进程上面和新进程的创建方式密切相关。UNIX操作系统有两种创建新进程的机制分别是`fork()`和`exec()`。

`fork()`可以创建当前进程的一个副本且有独立的PID号码（父进程和子进程只有PID号码不同）。Linux使用了一个中所周的的技术来使`fork`更高效，那就是写时复制（copy on write）操作。

`exec()`将一个新程序加载到当前的内存中执行，旧程序的内存页面被逐出，其内容被替换为新数据，然后开始执行新的程序。

## 1.2 线程

除了重量级进程（有时候也成为UNIX进程），还有一个轻量级*进程*（线程）。本质上，一个进程包含若干个线程，这些线程使用着同样的地址空间，共享同样的数据和资源，因此除了使用同步的方法保护共享的资源之外，没有额外的通信机制了。这也是线程和进程的差别。

Linux使用`clone()`的方法创建线程。其工作方式类似于`fork`，但启用了精准的检查，以确认那些资源与父进程共享、哪些资源为线程独立创建。

## 1.3 用户空间进程编程

linux提供一些关于进程的系统调用，例如`fork()`、`getpid()`、`getppid()`、`wait()`、`waitpid()`，可以创建进程，对父进程和子进程流程的一些控制。

### 1.3.1 fork进程
https://github.com/carloscn/clab/blob/master/linux/test_pid/test_pid.c

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int main(int argc, char *argv[])
{
    pid_t pid = fork();

    while(1) {
        if (0 == pid) {
            debug_log("Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("I'm Child process!\n");
            sleep(1);
        } else if (pid > 0) {
            debug_log("-------------------> Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("-------------------> I'm Parent process!\n");
            sleep(2);
        } else {
            debug_log("fork() failed \n");
        }
    }

    return 0;
}

```
两个进程会竞争一个终端输出：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220807180928.png)
但是后台是被fork了两个进程：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220807181014.png)

### 1.3.2 zombie进程
fork进程之后，子进程的处理程序中直接退出，而父进程处理程序中没有任何等待子进程的操作。这时候子进程就会成为*僵尸进程*。僵尸进程会被init 0收养，等待父进程退出之后，僵尸进程会被init 0进程释放掉。
https://github.com/carloscn/clab/blob/master/linux/test_pid/test_pid_zombie.c
```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

int main(int argc, char *argv[])
{
    pid_t pid = fork();
    u_int16_t count = 0;

    while(1) {
        if (0 == pid) {
            debug_log("Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("I'm Child process\n");
            // just exit child process, and parent process don't call waitpid(),
            // which to create zombie process
            sleep(1);
            count ++;
            if (count > 10) {
                debug_log("child will exit.\n");
                exit(0);
            }
        } else if (pid > 0) {
            // debug_log("-------------------> wait for child exiting.\n");
            // WNOHANG is non block mode.
            // WUNTRACED is block mode.
            // mask the next line code, the zombie process will be generated.
            // pid_t t_pid = waitpid(pid, NULL, WUNTRACED);
            debug_log("-------------------> Current PID = %d, PPID = %d\n", getpid(), getppid());
            debug_log("-------------------> I'm Parent process!\n");
            sleep(2);
        } else {
            debug_log("fork() failed \n");
        }
    }
    return 0;
}

```
子进程退出之后，它成为僵尸进程。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220807181556.png)

父进程使用`waitpid`可以观测到子进程是否已经退出，以及时回收进程资源。

# 进程的管理和调度

## 进程优先级

进程优先级粗暴分为实时进程和非实时进程：
* **硬实时进程：**有严格的时间限制/**Linux不支持硬实时**/一些linux旁支版本RTLinux支持硬实时，主要原因是调度器没有做进内核中，内核作为一个独立的进程处理一些不仅要的任务。
* Linux的任务优先满足吞吐量，所以弱化了进程调度。但是这些年人们也在降低内核延迟上面做了很多研究，比如提出可抢占机制、实时互斥内核锁还有完全公平调度器。
* **软实时进程**：类似与写CD这种工作种类的进程，就算是写进程被暂时中断也不会造成宕机之类的风险操作。
* **普通进程**： 没有具体的时间约束限制，但是会分配优先级来区分重要性。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220807183447.png" width="50%" /> </div>

进程优先级模型转换为时间片长度，优先级越高的占用的时间片越多。这种方案被称为**抢占式多任务处理(preemptive multitasking)**，即各个进程都被分配到一定的时间片用来执行。时间到了之后，内核会从当前进程收回控制权，让不同的进程运行。抢占的时候所有的CPU寄存器的内容和页表都会保存起来，因此其结果并不会因为进程的切换而丢失。

## 进程的生命周期

* 生命周期分为：运行、等待、休眠和终止。
* 进程的状态机由调度器来改变状态。
* 不在于周期范围内的进程:“僵尸进程”：子进程被KILL信号（SIGTERM和SIGKILL）杀死，父进程没有调用wait4()。正常流程是，子进程被KILL，父进程调用wait4系统调用，通知内核子进程已死。
* 处于僵尸进程状态进程：
  * 还会在进程表中（ps/top能刷出进程）
  * 占用很少的资源
  * 重启后才能刷掉该进程
* 抢占式多任务处理
  * 进程执行分为内核态和用户态，最大区别在于，内存地址区域访问划分不同。
  * 进内核态方法一：如果用户态进程进入内核态：访问共享数据，文件系统空间，必须通过**系统调用**，才能进入到内核态。
  * 进内核态方法二：中断触发进入内核态。
  * 中断触发和系统调用可以使用户态进入到内核态，但用户态是主动调用的，中断是外部触发的。
  * 用户态 < 核心态 < 中断
* 进程抢占层次：
  * 普通进程总会被抢占（kernel preemption）
  * 进程处于内核态（或者普通进程处于系统调用期间）无法被任何进程抢占，但中断可以抢占内核态，因此要求**中断不能占用太长时间**。
  * 优点：减少等待时间，让进程更“平滑”的执行；缺点：增加内核复杂程度。

## 进程的表示

Linux有一套自己对于进程管理的方式，还有调度器，调度器可以理解为真正去执行和指挥进程如何运行的东西，而管理方式可以说是调度器去管理进程的一个笔记计划表（task_structure）在include/sched.h中，这里有如何管理进程，进程命名，编号法等等。

### task结构体

```c
<sched.h>
stcuct task_struct {
// 非常多的成员
};
```

成员非常多，弄清楚费劲，可以分类：

* 状态和执行信息，如待决信号、进程的二进制格式种类、进程PID、父进程地址、优先级和CPU时间。
* 分配的虚拟内存信息。
* 身份凭据，如用户ID、组ID和权限等。
* 使用的文件（包括程序代码的二进制文件）
* 线程信息记录，CPU的运行时间数据。
* 与其他进程通信有关的信息。
* 该进程所用的信号处理，用于响应到来的信号。

### 进程状态机

```C
<sched.h>
stcuct task_struct {
	volatile long state;
};
```

* TASK_RUNNING: 处于可运行状态，未必处于CPU cover时间也有可能是等待调度器的状态。
* TASK_INTERRUPTIBLE: 针对某时间或者其他资源的睡眠进程设置的。
* TASK_UNINTERRUPTIBLE：用于内核指示而停用的睡眠进程。不能由外部唤醒，只能由内核亲自唤醒。
* TASK_STOPPED：进程特意停止运行，例如，由调度器暂停。
* TASK_TRACED：本来不是进程状态，用于停止的进程（ptrace机制）与常规停止进程区分开。
* EXIT_ZOMBIE：僵尸状态
* EXIT_DEAD：wait系统调用已发出，解除僵尸状态。

### 进程资源限制

```C
<sched.h>
<resource.h>
struct rlimit {
    unsigned long rlim_cur;
    unsigned long rlim_max;
}
struct task_struct {
	struct rlimite limit;
};
```

* rlim_cur: 进程当前的资源限制，成为软限制(soft limit)
* rlim_max:该限制的最大容许值， 硬限制（hard limit）

用户进程通过setrlimit()系统调用来增减当前限制，最大值不能超过rlim_max；getrlimits()用于检查当前限制。那么设定的资源是什么呢？

![](/img/bVcVKjc)

`cat /proc/self/limits`来查看当前系统设定进程的资源限制。

![](/img/bVcVKiX)

Linux系统启动的时候会设定好当前资源限制的属性，在`include/asm-generic/resource.h`中定义进程的资源限制，在linux启动的时候通过init进程完成配置。

![](/img/bVcVKi6)

在init_task.h中挂载该数组。

![image-20211101105750497](/img/bVcVKi7))

然后会在各个进程中使用该变量：

![](/img/bVcVKja)

### 进程类型

进程是由（二进制代码应用程序）、（单线程）、分配给应用程序的资源（内存、文件）。新进程是使用（fork）或者（exec）系统调用产生的。

* fork生成当前进程的副本，称为子进程，复制为两份一样的独立的进程，资源是分开的，资源都是copy的两份，不再系统上做任何关联。
* exec从一个可执行二进制加载另一个应用程序，来替代当前运行的进程。exec不是创建新的进程，首先使用fork复制一份旧的程序，然后调用exec在系统上创建一个应用程序。

* 旧版本的clone调用，原理和fork一致，可以共享父进程的一些资源。

### 命名空间

Linux的全局管理特性，比如PID、UID和系统调用uname返回系统的信息都是全局调用。因此就导致资源和重用的问题，在虚拟化中亟需解决。把全局资源通过命名空间抽象出来，划分不同的命名空间对应不同的资源分配，可实现虚拟化环境。

父命名空间生成多个子命名空间，子命名空间有映射关系到父命名空间。

#### 命名空间的数据结构

```c
#include <nsproxy.h>
struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns; //uts: 内核名称、版本、底层体系结构
	struct ipc_namespace *ipc_ns; //ipc: 进程间通信有关的信息
	struct mnt_namespace *mnt_ns; //已装在的文件系统的视图 stcuct mnt_namespace
	struct pid_namespace *pid_ns; //有关进程ID的信息 struct pid_namespace
	struct user_namespace *user_ns; // 保存用于限制每个用户资源使用的信息
	struct net *net_ns;// 包含所有网络相关的命名空间参数
};
```

创建新进程的使用使用fork可以建立一个新的命名空间，所以在fork的时候需要标记创建新的命名空间的类别。

```C
#include <sched.h> 
#define CLONE_NEWUTS 0x04000000 /* 创建新的utsname组 */ 
#define CLONE_NEWIPC 0x08000000 /* 创建新的IPC命名空间 */ 
#define CLONE_NEWUSER 0x10000000 /* 创建新的用户命名空间 */ 
#define CLONE_NEWPID 0x20000000 /* 创建新的PID命名空间 */ 
#define CLONE_NEWNET 0x40000000 /* 创建新的网络命名空间 */
```

在task_struct里面有有关于命名空间的定义：

```C
struct task_struct {
    //...
	struct nsproxy *nsproxy;
    //...
};
```

![image-20211102103721609](/img/bVcVLO0))

struct  task_struct里面挂的是stcut nsproxy的指针，只要挂上不同命名空间的指针，就算是赋予不同的命名空间了。

NOTE: 命名空间的支持必须在编译的时候启动 General setup -> Namespaces support，而且必须逐一指定需要支持的命名空间。如果内核编译的时候没有指定命名空间的支持，默认的命名空间的作用则类似于不启用命名空间，所有的属性相当于全局的。

![](/img/bVcVLO1)

#### UTS命名空间和用户命名空间

```C
<kernel/nsproxy.c> 
struct nsproxy init_nsproxy = INIT_NSPROXY(init_nsproxy); 

<init_task.h> 
#define INIT_NSPROXY(nsproxy) { \ 
    .pid_ns = &init_pid_ns, \ 
    .count = ATOMIC_INIT(1), \ 
    .uts_ns = &init_uts_ns, \ 
    .mnt_ns = NULL, \ 
    INIT_NET_NS(net_ns) \ 
    INIT_IPC_NS(ipc_ns) \ 
    .user_ns = &init_user_ns, \ 
}
```

##### UTS命名空间

UTS命名空间是Linux内核Namespace（命名空间）的一个子系统，主要用来完成对容器HOSTNAME和domain的隔离，同时保存内核名称、版本、以及底层体系结构类型等信息。

```C
<utsname.h> 
struct uts_namespace { 
    struct kref kref; 
    struct new_utsname name; 
};
// 在proc中可以看到这些值
struct new_utsname { 
    char sysname[65];    //  cat /proc/sys/kernel/ostype
    char nodename[65];   
    char release[65];    //  cat /proc/sys/kernel/osrelease
    char version[65];    //  cat /proc/sys/kernel/version
    char machine[65]; 
    char domainname[65]; // cat /proc/sys/kernel/domainname
};

init/version.c 
struct uts_namespace init_uts_ns = { 
... 
    .name = { 
        .sysname = UTS_SYSNAME, 
        .nodename = UTS_NODENAME, 
        .release = UTS_RELEASE, 
        .version = UTS_VERSION,
        .machine = UTS_MACHINE, 
        .domainname = UTS_DOMAINNAME, 
	}, 
};
```

就一个名字和kref，引用计数器，可以跟踪内核中有多少地方使用了struct uts_namespace的实例。

##### 用户命名空间

用户命名空间在数据结构管理方面类似于UTS：在要求创建新的用户命名空间时，则生成当前用户命名空间的一份副本，并关联到当前进程的nsproxy实例。但用户命名空间自身的表示要稍微复杂一些：

```C
<user_namespace.h> 
    struct user_namespace { 
    struct kref kref; 
    struct hlist_head uidhash_table[UIDHASH_SZ]; 
    struct user_struct *root_user; 
};

<kernel/user_namespace.c> 
static struct user_namespace *clone_user_ns(struct user_namespace *old_ns) 
{ 
    struct user_namespace *ns; 
    struct user_struct *new_user; 
    ... 
    ns = kmalloc(sizeof(struct user_namespace), GFP_KERNEL); 
    ... 
    ns->root_user = alloc_uid(ns, 0); 
    /* 将current->user替换为新的 */ 
    new_user = alloc_uid(ns, current->uid); 
    switch_uid(new_user); 
    return ns; 
}
```

#### 

