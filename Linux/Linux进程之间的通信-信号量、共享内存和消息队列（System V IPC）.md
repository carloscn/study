# Linux进程之间的通信-信号量、共享内存和消息队列（System V and POSIX IPC）

* 信号量 (sem) ： 管理资源的访问
* 共享内存 (shm)： 高效的数据分享
* 消息队列 (msg)：在进程之间简易的传数据的方法

 IPC（Inter-Process Communication，进程间通讯）包含三种通信方式，信号量、共享内存和消息队列。在linux编程里面可以有两个不同的标准，一个是SYSTEM-V标准，一个是POSIX标准。以下是两个标准之间的区别[^1]。简单的说，POSIX更轻量，常面向于线程；SYSTEM-V更重一些，需要深陷Linux内核之中，面向于进程。

## 1. POSIX和SYSTEM-V的区别

Following table lists the differences between System V IPC and POSIX IPC[^2].

| SYSTEM V                                                     | POSIX                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| AT & T introduced (1983) three new forms of IPC facilities namely message queues, shared memory, and semaphores. | Portable Operating System Interface standards specified by IEEE to define application programming interface (API). POSIX covers all the three forms of IPC |
| SYSTEM V IPC covers all the IPC mechanisms viz., pipes, named pipes, message queues, signals, semaphores, and shared memory. It also covers socket and Unix Domain sockets. | Almost all the basic concepts are the same as System V. It only differs with the interface |
| Shared Memory Interface Calls shmget(), shmat(), shmdt(), shmctl() | Shared Memory Interface Calls shm_open(), mmap(), shm_unlink() |
| Message Queue Interface Calls msgget(), msgsnd(), msgrcv(), msgctl() | Message Queue Interface Calls mq_open(), mq_send(), mq_receive(), mq_unlink() |
| Semaphore Interface Calls semget(), semop(), semctl()        | Semaphore Interface Calls Named Semaphores sem_open(), sem_close(), sem_unlink(), sem_post(), sem_wait(), sem_trywait(), sem_timedwait(), sem_getvalue() Unnamed or Memory based semaphores sem_init(), sem_post(), sem_wait(), sem_getvalue(),sem_destroy() |
| Uses keys and identifiers to identify the IPC objects.       | Uses names and file descriptors to identify IPC objects      |
| NA                                                           | POSIX Message Queues can be monitored using select(), poll() and epoll APIs |
| Offers msgctl() call                                         | Provides functions (mq_getattr() and mq_setattr()) either to access or set attributes 11. IPC - System V & POSIX |
| NA                                                           | Multi-thread safe. Covers thread synchronization functions such as mutex locks, conditional variables, read-write locks, etc. |
| NA                                                           | Offers few notification features for message queues (such as mq_notify()) |
| Requires system calls such as shmctl(), commands (ipcs, ipcrm) to perform status/control operations. | Shared memory objects can be examined and manipulated using system calls such as fstat(), fchmod() |
| The size of a System V shared memory segment is fixed at the time of creation (via shmget()) | We can use ftruncate() to adjust the size of the underlying object, and then re-create the mapping using munmap() and mmap() (or the Linux-specific mremap()) |

我从文献里面得到几个进程持续性概念，我觉得这个角度分类比较好。直接抄文献：从IPC的持续性角度而言，可以把进程通信分为以下几类[^3]：

>* 随进程持续 (Process-Persistent IPC)
>
>  * IPC对象一直存在，直到最后拥有他的进程被关闭为止，典型的IPC有pipes（管道）和FIFOs（先进先出对象）
>
>  * Pipe, FIFO, Posix的mutex（互斥锁）, condition variable（条件变量）, read-write lock（读写锁），memory-based semaphore（基于内存的信号量） 以及 fcntl record lock，TCP和UDP套接字，Unix domain socket
>
>* 随内核持续 (Kernel-Persistent IPC)
>
>  * IPC对象一直存在直到内核被重启或者对象被显式关闭为止，在Unix中这种对象有System V 消息队列，信号量，共享内存。（注意Posix消息队列，信号量和共享内存被要求为至少是内核持续的，但是也有可能是文件持续的，这样看系统的具体实现）。
>  * Posix的message queue（消息队列）, named semaphore（命名信号量）, System V Message queue, semaphore, shared memory。
>
>* 随文件系统持续  (FileSystem-Persistent IPC)
>
>  * 除非IPC对象被显式删除，否则IPC对象会一直保持（即使内核才重启了也是会留着的）。如果Posix消息队列，信号量，和共享内存都是用内存映射文件的方法，那么这些IPC都有着这样的属性。
>  * 要注意的是，虽然上面所列的IPC并没有随文件系统的，但是我们就像我们刚才所说的那样，Posix IPC可能会跟着系统具体实现而不同（具有不同的持续性），举个例子，写入文件肯定是一个文件系统持续性的操作，但是通常来说IPC不会这样实现。很少有IPC会实现文件系统持续，因为这会降低性能，不符合IPC的设计初衷。
>
>**System V IPC不是随进程持续的，是随内核持续的。**

**上面是摘录的，下面谈下我的理解：**我们在Linux编程里面，关于线程可以使用pthread_mutex, spinlock这些工具，这些工具都是在一个进程中的，守护的是进程内部的资源，因此作者提到随进程持续的概念；而两个无关进程之间对于访问同一个资源，比如文件，也是可能会有临界区，只是相比于进程内部的临界区，扩展到了系统内部的临界区，因此这里有随内核持续的概念。这也是为什么POSIX是一个轻量级的常用于线程的，而System V IPC是一个深陷内核的常用于进程的标准。





## Reference

[^1]: [System V IPC vs POSIX IPC - Stack Overflow](https://stackoverflow.com/questions/4582968/system-v-ipc-vs-posix-ipc)
[^2]:[System V & Posix (tutorialspoint.com)](https://www.tutorialspoint.com/inter_process_communication/inter_process_communication_system_v_posix.htm)
[^3]:[UNIX 进程间通讯（IPC）概念（Posix，System V IPC）](https://www.cnblogs.com/Philip-Tell-Truth/p/6284475.html)
