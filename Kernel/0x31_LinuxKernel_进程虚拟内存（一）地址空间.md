# 0x31_LinuxKernel_进程虚拟内存（一）地址空间

我们在ARM的博客里面了解到从CPU的角度来看内存管理，里面涉及了很多细节，包括整个的内存分层机制，cache-TLB-ddr如何工作的，并且在ARM上面提供了寄存器里面包含了内存的地址。可以参考以下内容：

-   [13_ARMv8_内存管理（一）-内存管理要素](https://github.com/carloscn/blog/issues/53) 
-   [14_ARMv8_内存管理（二）-ARM的MMU设计](https://github.com/carloscn/blog/issues/54) 
-   [15_ARMv8_内存管理（三）-MMU恒等映射及Linux实现](https://github.com/carloscn/blog/issues/55) 

Linux系统内核如何使用ARM的MMU机制呢？Linux内核又是如何给用户提供一些使用内存的接口或者方法规则呢？这将是本章的重点，I will work you through memory management between the linux kernel and the linux userspace.

The memory management of linux kernel is a very complex mechanism. To easily understand the mechanism, we get it from the perspective of the linux userspace.  笨叔书里面写的这句话不错：

*   如果从Linux的使用者角度来看内存管理，经常使用的`free`命令；
*   如果从Linux编程者角度来看，主要就是使用分配函数`malloc`,`mmap`等
*   如果从Linux内核角度来看，那么又可以分角度了：
    *   系统模块角度
    *   进程角度

我们一步步的揭开Linux的内存管理的面纱，和我们的ARM的MMU机制联合在一起。

# 1. 非内核角度来看内存管理

本节站在Linux使用者角度 + 编程者角度来看内存管理。

## 1.2 使用者角度

从使用者的角度来看`free`是一个查看linux内存的工具，他表示当前系统中已经使用的内存和空闲的内存情况。

![image-20220831160204023](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220831160204023.png)

*   `total`：总共内存数目，如果是`-m`参数也按照兆字节显示；
*   `used`：表示已经使用的内存
*   `free`：未被分配的**物理内存**大小
*   `shared`：共享内存大小，主要用于进程之间的通信
*   `buff/cache`：buff指的是buffers，用来给设备作为缓冲；cache为page cache，用来给文件作为缓冲，提高文件访问速度。
*   `available`：当内存短缺的时候，系统可以回收buffers和page cache。那么公式`available = free + buffers + page cache - 不可回收部分`（不可回收部分在page cache里面包含内存共享段、tmpfs、ramfs等）

## 1.3 从应用编程者角度

Linux的userspace空间提供了malloc()这类的函数，是可以让用户直接访问到内存的最直接的接口。malloc()是从虚拟内存分配内存的函数。除此之外，C库中还提供了一下管理内存的接口，他们是：

*   `void *malloc(size_t size)`
*   `void free(void *ptr)`
*   `void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset)`
*   `int munmap(void *addr, size_t len)`
*   `int getpagesize(void)`
*   `int mprotect(const void *addr, size_t len, int prot)`
*   `int mlock(const void* addr, size_t len)`
*   `int munlock(const void *addr, size_t len)`
*   `int madvise(void* addr, size_t len, int advice)`
*   `void *mremap(void *old_addr, size_t old_size, size_t new_size, int flags....)`
*   `int rmap_file_pages(void *addr, size_t size, int prot, ssize_t pgoff, int flags)`

我们从Linux用户空间来看内存，需要注意以下几种情况：

**分配大量的内存**

**现象一：**假如我们的Linux系统的物理内存只有2GB大小，我们申请了一个比物理内存还要大的空间，例如4GB，我们会发现一眨眼的功夫就malloc完毕（如果你不使用的话）。

**现象二：**如果我们无限度的分配内存，就会发现执行的很慢，最后可能还会收到`Out of Memory：killed process xxxx`的log输出（在一些操作系统里只可能退出不会输出任何内容）。

**这两个现象我们可以浅显的解释一下从编程者角度遇到的问题：**

*   因为MMU的存在因此可以分配比物理内存还大的空间，得益于MMU的换页原理和swap技术。

*   malloc在开始分配而没有使用的时候感觉速度特别快，那是因为延迟分配策略。如果不使用这个内存，将不会提取该页面。我们在malloc之后使用memset对这个内存进行初始化，用户侧只是两句话的事儿，实际上linux内核内部执行了非常复杂的逻辑。

    >1.  用户态程序使用malloc接口，分配虚拟地址。
    >2.  0用户程序访问该虚拟地址，比如memset。
    >3.  硬件（MMU）需要将虚拟地址转换为物理地址。
    >4.  硬件读取页表。
    >5.  硬件发现相应的页表项不存在，硬件自动触发缺页异常。
    >6.  硬件自动跳转到page fault的处理程序(内核实现注册好)
    >7.  内核中的page fault处理程序执行，在其中分配物理内存，然后修改页表(创建页表项)
    >8.  异常处理完毕，返回程序用户态，继续执行memset相应的操作。

*   超过一定数量会被Kill，这是Linux的内存耗尽（术语：OOM）保护机制。是Linux保护自己资源的一种手段。

这里就引入了一个话题：既然Linux系统提供了如此丰富的内存保护机制，在资源耗尽前甚至可以直接杀死进程，而且有MMU的存在，那我们是否要对malloc的返回值进行合法性检查呢？答案是肯定的，肯定要检查。那我们检查哪一类错误呢？

通俗来讲，如果我们使用了malloc申请了内存，而我们并没有合法的使用这部分内存（越界访问），此时非法的内存访问可能不会第一时间让程序崩溃，但实际上已经破坏了malloc的分配结构，等下一次进行malloc分配的时候，那么malloc就由于结构性的破坏而无法使用了，因此，我们还是需要对申请的内存进行合法性检查。**malloc失败很难因为空间耗尽，而是因为结构性破坏，这个问题会非常难调试（也不一定会出现），通常需要一些辅助的调试工具**。

除了malloc，还有calloc和realloc这些函数。calloc结构数组分配，并初始化为0，malloc没有初始化功能；realloc函数改变之前分配的长度（记得实用realloc重新返回的指针，旧的指针不要用了）

## 1.4 从内存布局角度看内存管理

要了解Linux的内存机制，必然要掌握Linux的内存布局。在内核代码：`arch/arm/mm/init.c:472:void __init mem_init(void)`函数内部，可以把Linux的内存布局打印出来。

```C
/*
 * mem_init() marks the free areas in the mem_map and tells us how much
 * memory is free.  This is done after various parts of the system have
 * claimed their memory after the kernel image.
 */
void __init mem_init(void)
{
#ifdef CONFIG_HAVE_TCM
	/* These pointers are filled in on TCM detection */
	extern u32 dtcm_end;
	extern u32 itcm_end;
#endif

	set_max_mapnr(pfn_to_page(max_pfn) - mem_map);

	/* this will put all unused low memory onto the freelists */
	free_unused_memmap();
	free_all_bootmem();

#ifdef CONFIG_SA1111
	/* now that our DMA memory is actually so designated, we can free it */
	free_reserved_area(__va(PHYS_OFFSET), swapper_pg_dir, -1, NULL);
#endif

	free_highpages();

	mem_init_print_info(NULL);

#define MLK(b, t) b, t, ((t) - (b)) >> 10
#define MLM(b, t) b, t, ((t) - (b)) >> 20
#define MLK_ROUNDUP(b, t) b, t, DIV_ROUND_UP(((t) - (b)), SZ_1K)

	pr_notice("Virtual kernel memory layout:\n"
			"    vector  : 0x%08lx - 0x%08lx   (%4ld kB)\n"
#ifdef CONFIG_HAVE_TCM
			"    DTCM    : 0x%08lx - 0x%08lx   (%4ld kB)\n"
			"    ITCM    : 0x%08lx - 0x%08lx   (%4ld kB)\n"
#endif
			"    fixmap  : 0x%08lx - 0x%08lx   (%4ld kB)\n"
			"    vmalloc : 0x%08lx - 0x%08lx   (%4ld MB)\n"
			"    lowmem  : 0x%08lx - 0x%08lx   (%4ld MB)\n"
#ifdef CONFIG_HIGHMEM
			"    pkmap   : 0x%08lx - 0x%08lx   (%4ld MB)\n"
#endif
#ifdef CONFIG_MODULES
			"    modules : 0x%08lx - 0x%08lx   (%4ld MB)\n"
#endif
			"      .text : 0x%p" " - 0x%p" "   (%4td kB)\n"
			"      .init : 0x%p" " - 0x%p" "   (%4td kB)\n"
			"      .data : 0x%p" " - 0x%p" "   (%4td kB)\n"
			"       .bss : 0x%p" " - 0x%p" "   (%4td kB)\n",

			MLK(UL(CONFIG_VECTORS_BASE), UL(CONFIG_VECTORS_BASE) +
				(PAGE_SIZE)),
#ifdef CONFIG_HAVE_TCM
			MLK(DTCM_OFFSET, (unsigned long) dtcm_end),
			MLK(ITCM_OFFSET, (unsigned long) itcm_end),
#endif
			MLK(FIXADDR_START, FIXADDR_END),
			MLM(VMALLOC_START, VMALLOC_END),
			MLM(PAGE_OFFSET, (unsigned long)high_memory),
#ifdef CONFIG_HIGHMEM
			MLM(PKMAP_BASE, (PKMAP_BASE) + (LAST_PKMAP) *
				(PAGE_SIZE)),
#endif
#ifdef CONFIG_MODULES
			MLM(MODULES_VADDR, MODULES_END),
#endif

			MLK_ROUNDUP(_text, _etext),
			MLK_ROUNDUP(__init_begin, __init_end),
			MLK_ROUNDUP(_sdata, _edata),
			MLK_ROUNDUP(__bss_start, __bss_stop));

#undef MLK
#undef MLM
#undef MLK_ROUNDUP

	/*
	 * Check boundaries twice: Some fundamental inconsistencies can
	 * be detected at build time already.
	 */
#ifdef CONFIG_MMU
	BUILD_BUG_ON(TASK_SIZE				> MODULES_VADDR);
	BUG_ON(TASK_SIZE 				> MODULES_VADDR);
#endif

#ifdef CONFIG_HIGHMEM
	BUILD_BUG_ON(PKMAP_BASE + LAST_PKMAP * PAGE_SIZE > PAGE_OFFSET);
	BUG_ON(PKMAP_BASE + LAST_PKMAP * PAGE_SIZE	> PAGE_OFFSET);
#endif

	if (PAGE_SIZE >= 16384 && get_num_physpages() <= 128) {
		extern int sysctl_overcommit_memory;
		/*
		 * On a machine this small we won't get
		 * anywhere without overcommit, so turn
		 * it on by default.
		 */
		sysctl_overcommit_memory = OVERCOMMIT_ALWAYS;
	}
}
```

图片可以表示如下：

![img](https://raw.githubusercontent.com/carloscn/images/main/typora20151208112232028.png)

ARM64架构处理器采用的48位物理寻址机制，最大可以寻找256T的内存，足以应付现在的物理内存。在Linux架构中，会把内存空间分为用户空间和系统空间。

![image-20220901163552735](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220901163552735.png)

从上面的图片可以





