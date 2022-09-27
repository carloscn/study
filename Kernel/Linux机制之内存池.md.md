# Linux机制之内存池

通常的进程发起申请内存的动作之后，会在系统的空闲内存区寻找合适大小的内存块（底层分配函数__alloc_node_mask），如果满足就直接分配，如果不满足就会向上查找。如果过大就会进行分裂，一部分分给申请进程，一部分放入空闲区。释放时需要找到这个块对应的伙伴，如果伙伴也为空闲，就进行合并，放入高阶空闲链表，如果不空闲就放入对应链表。同时对于多线程申请和释放内存，需要加锁。这样的默认的分配方式考虑到了系统中的大部分情况，具有通用性，但是无可避免的会产生内部碎片，而且加锁，解锁的开销也很大。**程序可以通过系统的内存分配方法预先分配一大块内存来做一个内存池，之后程序的内存分配和释放都由这个内存池来进行操作和管理，当内存池不足时再向系统申请内存**。

我们通常使用malloc等函数来为用户进程分配内存。它的执行过程通常是由用户程序发起malloc申请内存的动作，在**标准库找到对应函数，对不满128k的调用brk()系统调用来申请内存（申请的内存是堆区内存），接着由操作系统来执行brk系统调用**。

我们知道malloc是在标准库，真正的申请动作需要操作系统完成。所以由应用程序到操作系统就需要3层。内存池是专为应用程序提供的专属的内存管理器，它属于应用程序层。所以程序申请内存的时候就不需要通过标准库和操作系统，明显降低了开销。

# 1. 内存池分类 [^1]

对于线程安全来说，内存池可以分**为单线程内存池和多线程内存池**。单线程内存池整个生命周期只被一个线程使用，因而不需要考虑互斥访问的问题；**多线程内存池有可能被多个线程共享，因此则需要在每次分配和释放内存时加锁**。相对而言，单线程内存池性能更高，而多线程内存池适用范围更广。

从可分配内存大小来说，可以分为固定内存池和可变内存池。所谓固定内存池是指应用程序每次从内存池中分配出来的内存单元大小事先已经确定，是固定不变的；而可变内存池则每次分配的内存单元大小可以按需变化，应用范围更广，而性能比固定内存池要低。

# 2. 内存池的工作原理

## 2.1 原理

固定内存池的设计如图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220913113826.png)

固定内存池的内存块实际上是由链表连接起来的，前一个总是后一个的2倍大小。当内存池的大小不够分配时，向系统申请内存，大小为上一个的两倍，并且使用一个指针来记录当前空闲内存单元的位置。当我们要需要一个内存单元的时候，就会随着链表去查看每一个内存块的头信息，如果内存块里有空闲的内存单元，将该地址返回，并且将头信息里的空闲单元改成下一个空闲单元。

当应用程序释放某内存单元，就会到对应的内存块的头信息里修改该内存单元为空闲单元。

## 2.2 线程安全考虑

对于单线程来说，不需要考虑互斥访问的问题，但是对于多线程来说内存池可能会被多个线程共享，所以需要给内存池加上一个锁，来进行保护。

如果出现程序中有大量线程申请释放内存，那么这种方案下锁的竞争将会非常激烈，线程这样的场景下使用该方案不会有很好的性能。

所以就有了一种方法的诞生----------->TLS线程本地存储（线程局部存储）。

TLS的作用是能将数据和执行的特定的线程联系起来。它可以让程序中由所有线程使用的全局变量，都产生自己的副本。也就是这个变量每个线程都有自己的私有，它们之间的操作互不干扰，不会影响。

在这样的背景下，我们可以设计出线程本地存储+内存池的方式，这样线程都有了自己的内存池，且彼此之间不会产生影响，也就不需要加锁。

## 2.3 内存池设计

### 提前创建

假设服务器程序比较简单，处理请求的时候只使用一种对象，提前创建出一些需要的对象，比如数据结构，我们可以用的时候拿出来，不用的时候还回去就可以，只需要标记使用的和未使用的。这种比较简单。

### 可申请不同大小的内存

这种方法使得用户程序在请求过程中只申请内存，之后当处理完请求之后才一次性释放所有内存，这样可以降低内存申请和释放的开销，而且能减少内部碎片。

除了这两种还有其他的设计方法，可以自己设计。

**因为内存池并不是一种通用的内存管理模式，它是一种比较特定的，固定某种场景去使用，属于私人定制**。

# 3. Linux kernel的内存池

在内核中有不少地方内存分配不允许失败。内存池作为一个在这些情况下确保分配的方式，内核开发者创建了一个已知为内存池(或者是 “mempool” )的抽象，内核中内存池真实地只是相当于后备缓存，它尽力一直保持一个空闲内存列表给紧急时使用，而在通常情况下有内存需求时还是从公共的内存中直接分配，这样的做法虽然有点霸占内存的嫌疑，但是可以从根本上保证关键应用在内存紧张时申请内存仍然能够成功。

## 3.1 数据结构
```C
typedef struct mempool_s {
    spinlock_t lock; /*保护内存池的自旋锁*/
    int min_nr; /*内存池中最少可分配的元素数目*/
    int curr_nr; /*尚余可分配的元素数目*/
    void **elements; /*指向元素池的指针*/
    void *pool_data; /*内存源，即池中元素真实的分配处*/
    mempool_alloc_t *alloc; /*分配元素的方法*/
    mempool_free_t *free; /*回收元素的方法*/
    wait_queue_head_t wait; /*被阻塞的等待队列*/
} mempool_t;

```
## 3.2 函数

### 3.2.1 create
内存池的创建函数`mempool_create`

```C
mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn,
                mempool_free_t *free_fn, void *pool_data)
{
    return mempool_create_node(min_nr,alloc_fn,free_fn, pool_data,-1);
}

```
这个函数指定了，内存池大小，分配方法，释放发法，分配源。创建完成之后，会从分配源（pool_data）中分配内存池大小（min_nr）个元素来填充内存池。

### 3.2.2 destory
内存池的释放函数`mempool_destory`

```C
void mempool_destroy(mempool_t *pool)
{
    while (pool->curr_nr) {
        void *element = remove_element(pool);
        pool->free(element, pool->pool_data);
    }
    kfree(pool->elements);
    kfree(pool);
}

```
他是依次将元素对象从池中移除，再释放给pool_data，最后释放池对象。

### 3.2.3 alloc

内存池分配对象的函数：`mempool_alloc`。`mempool_alloc`的作用是从指定的内存池中申请/获取一个对象。

```C
void *mempool_alloc(mempool_t *pool, gfp_t gfp_mask)
{
	void *element;
	unsigned long flags;
	wait_queue_entry_t wait;
	gfp_t gfp_temp;

	VM_WARN_ON_ONCE(gfp_mask & __GFP_ZERO);
	might_sleep_if(gfp_mask & __GFP_DIRECT_RECLAIM);

	gfp_mask |= __GFP_NOMEMALLOC;	/* don't allocate emergency reserves */
	gfp_mask |= __GFP_NORETRY;	/* don't loop in __alloc_pages */
	gfp_mask |= __GFP_NOWARN;	/* failures are OK */

	gfp_temp = gfp_mask & ~(__GFP_DIRECT_RECLAIM|__GFP_IO);

repeat_alloc:

	element = pool->alloc(gfp_temp, pool->pool_data);//先从后备源中申请内存
	if (likely(element != NULL))
		return element;

	spin_lock_irqsave(&pool->lock, flags);
	if (likely(pool->curr_nr)) {
		element = remove_element(pool);/*从内存池中提取一个对象*/
		spin_unlock_irqrestore(&pool->lock, flags);
		/* paired with rmb in mempool_free(), read comment there */
		smp_wmb();
		/*
		 * Update the allocation stack trace as this is more useful
		 * for debugging.
		 */
		kmemleak_update_trace(element);
		return element;
	}

	/*
	 * We use gfp mask w/o direct reclaim or IO for the first round.  If
	 * alloc failed with that and @pool was empty, retry immediately.
	 */
	if (gfp_temp != gfp_mask) {
		spin_unlock_irqrestore(&pool->lock, flags);
		gfp_temp = gfp_mask;
		goto repeat_alloc;
	}

	/* We must not sleep if !__GFP_DIRECT_RECLAIM */
	if (!(gfp_mask & __GFP_DIRECT_RECLAIM)) {
		spin_unlock_irqrestore(&pool->lock, flags);
		return NULL;
	}

	/* Let's wait for someone else to return an element to @pool */
	init_wait(&wait);
	prepare_to_wait(&pool->wait, &wait, TASK_UNINTERRUPTIBLE);//加入等待队列

	spin_unlock_irqrestore(&pool->lock, flags);

	/*
	 * FIXME: this should be io_schedule().  The timeout is there as a
	 * workaround for some DM problems in 2.6.18.
	 */
	io_schedule_timeout(5*HZ);

	finish_wait(&pool->wait, &wait);
	goto repeat_alloc;
}
EXPORT_SYMBOL(mempool_alloc);

```

函数先从后备源中申请内存，当从后备源无法成功申请到时，才会从内存池中申请内存使用，因此可以发现内核内存池（mempool）其实是一种后备池，在内存紧张的情况下才会真正从池中获取，这样也就能保证在极端情况下申请对象的成功率，但也不一定总是会成功，因为内存池的大小毕竟是有限的，如果内存池中的对象也用完了，那么进程就只能进入睡眠，也就是被加入到pool->wait的等待队列，等待内存池中有可用的对象时被唤醒，重新尝试从池中申请元素。

### 3.2.4 free
内存池回收对象的函数：`mempool_free`
```C
void mempool_free(void *element, mempool_t *pool)
{
	unsigned long flags;

	if (unlikely(element == NULL))
		return;

	/*
	 * Paired with the wmb in mempool_alloc().  The preceding read is
	 * for @element and the following @pool->curr_nr.  This ensures
	 * that the visible value of @pool->curr_nr is from after the
	 * allocation of @element.  This is necessary for fringe cases
	 * where @element was passed to this task without going through
	 * barriers.
	 *
	 * For example, assume @p is %NULL at the beginning and one task
	 * performs "p = mempool_alloc(...);" while another task is doing
	 * "while (!p) cpu_relax(); mempool_free(p, ...);".  This function
	 * may end up using curr_nr value which is from before allocation
	 * of @p without the following rmb.
	 */
	smp_rmb();

	/*
	 * For correctness, we need a test which is guaranteed to trigger
	 * if curr_nr + #allocated == min_nr.  Testing curr_nr < min_nr
	 * without locking achieves that and refilling as soon as possible
	 * is desirable.
	 *
	 * Because curr_nr visible here is always a value after the
	 * allocation of @element, any task which decremented curr_nr below
	 * min_nr is guaranteed to see curr_nr < min_nr unless curr_nr gets
	 * incremented to min_nr afterwards.  If curr_nr gets incremented
	 * to min_nr after the allocation of @element, the elements
	 * allocated after that are subject to the same guarantee.
	 *
	 * Waiters happen iff curr_nr is 0 and the above guarantee also
	 * ensures that there will be frees which return elements to the
	 * pool waking up the waiters.
	 */
	if (unlikely(pool->curr_nr < pool->min_nr)) {
		spin_lock_irqsave(&pool->lock, flags);
		if (likely(pool->curr_nr < pool->min_nr)) {//当前可分配的是否小于内存大小，
			add_element(pool, element);
			spin_unlock_irqrestore(&pool->lock, flags);
			wake_up(&pool->wait);
			return;
		}
		spin_unlock_irqrestore(&pool->lock, flags);
	}
	pool->free(element, pool->pool_data);
}
EXPORT_SYMBOL(mempool_free);

```

其实原则跟mempool_alloc是对应的，释放对象时先看池中的可分配元素如果小于池中最少的可分配元素，那么久需要把元素放到内存池中。相反就要把它放到后备源中。

用户程序的内存池通常是特殊的，适用于特定场景的专属内存管理法。内核态的内存池则是保证系统中的一些**关键应用在内存紧缺的时候确保能够申请内存成功**。他们的用途不同，但是做法确是如出一辙。

# 4. Linux 用户空间内存池 [^2]

Linux用户空间也是有可以使用内存池的，可以使用`libtalloc_pools`的三方库函数。考虑到一个设计如何使用了数以千计的malloc()函数，这个就会造成很多碎片。所以这个talloc的库可以帮助我们使用内存池原理来减少碎片，提高性能。

使用这个库在ubuntu上需要安装`sudo apt install libtalloc-dev`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220913121101.png)

使用这个库：

```C
#include <talloc.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// sudo apt install libtalloc-dev
// gcc test_tmalloc.c -ltalloc

int main(void)
{
    size_t i = 0;
    /* allocate 1KiB in a pool */
    TALLOC_CTX *pool_ctx = talloc_pool(NULL, 1024);
    mempool_alloc();
    /* Take 512B from the pool, 512B is left there */
    char *ptr = (char *)talloc_size(pool_ctx, 512);
    memset(ptr, '1', 512);
    for (i = 0; i < 512; i++) {
        printf("%c, ", ptr[i]);
    }
    /* 1024B > 512B, this will create new talloc chunk outside
    the pool */
    void *ptr2 = talloc_size(ptr, 1024);

    /* The pool still contains 512 free bytes
    * this will take 200B from them. */
    void *ptr3 = talloc_size(ptr, 200);

    /* This will destroy context 'ptr3' but the memory
    * is not freed, the available space in the pool
    * will increase to 512B. */
    talloc_free(ptr3);

    /* This will free memory taken by 'pool_ctx'
    * and 'ptr2' as well. */
    talloc_free(pool_ctx);
}
```

编译：`gcc test_tmalloc.c -ltalloc`

运行之后： 

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220913121220.png)


# Ref：
[^1]:[内存池（memory pool）](https://blog.csdn.net/weixin_48101150/article/details/121336318)
[^2]:[libtalloc_pools (3) - Linux Man Pages](https://www.systutorials.com/docs/linux/man/3-libtalloc_pools/)