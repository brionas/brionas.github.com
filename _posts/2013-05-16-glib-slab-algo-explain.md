---
layout: post
title: "glib的slab算法实现学习"
description: slab提出来是为了解决内部内存碎片的问题，在linux内核中与buddy system一起来解决内核内存管理。但是要看懂slab在linux内核中的实现当前有些困难，我们不如拿些容易阅读的代码来了解slab算法的运作过程。GLIB库实现非常clear，可以做为slab算法的实现学习的入门。
category: "memory_management"
tags: [slab, glib, 源码阅读]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/
refer_post_addr: http://zeroli.github.io/memory_management/2013/03/24/glib-slab-algo-explain/
---
{% include JB/setup %}

slab提出来是为了解决内部内存碎片的问题，在linux内核中与buddy system一起来解决内核内存管理。但是要看懂slab在linux内核中的实现当前有些困难，我们不如拿些容易阅读的代码来了解slab算法的运作过程。GLIB库实现非常clear，可以做为slab算法的实现学习的入门。  
slab在GLIB中的实现相关文件是gslice.h/c，但是在这个实现文件中，有更复杂的，支持多线程更多的magazine caching算法。

##0. slab算法的运作机理
关于slab算法的运作机理，可以查看[wiki](http://en.wikipedia.org/wiki/Slab_allocation)


###slab结构体定义
{% highlight cpp %}
struct _ChunkLink {
	ChunkLink *next;
	ChunkLink *data;
};
struct _SlabInfo {
	ChunkLink *chunks;
	guint n_allocated;
	SlabInfo *next, *prev;
};
{% endhighlight %}
![](http://zeroli.github.io/assets/image/1345812003_5463.png)

从结构体可以知道，每个slab维护着前面slab的link prev和后面的slab的link prev，可知它们链接起来就是一个双向环形链表(slab ring)。除此之外，还维护着一个chunk指针，用来指向free chunk的单向链表。

###allocator结构体定义
{% highlight cpp %}
	...
	/* slab allocator */
	GMutex slab_mutex;
	SlabInfo **slab_stack; /* array of MAX_SLAB_INDEX (allocator) */
	guint color_accu;
} Allocator;
{% endhighlight %}
从数据结构可知，全局变量allocator会维护着一个指针数组，成员为SlabInfo*指针，指向一个环形slab双向链表，也即前面介绍的slab ring。每个成员的slabinfo*指向的slab ring所管理的chunks，size都必须是相同的，例如第一个slab ring里面的chunk全部都是8 (bytes)，第二个slab ring里全部都是8\*2 (bytes)，以此类推，一直到最大的chunk size。初始化时，所有的成员都会初始化为null，代表当前slab system没有free list。

###slab内存请求操作的过程
{% highlight cpp %}
static gpointer
slab_allocator_alloc_chunk (gsize chunk_size)
{
	ChunkLink *chunk;
	guint ix = SLAB_INDEX (allocator, chunk_size); // 找到chunk_size所对应的slab slot
	/* ensure non-empty slab */
	// 如果没有free list，则预先分配一页来填充此slot（一页可能会有很多chunks）
	if (!allocator->slab_stack[ix] || !allocator->slab_stack[ix]->chunks)
	allocator_add_slab (allocator, ix, chunk_size);
	/* allocate chunk */
	// 分配第一个chunk给client，后面的chunks挂接到当前slab上
	chunk = allocator->slab_stack[ix]->chunks;
	allocator->slab_stack[ix]->chunks = chunk->next;
	allocator->slab_stack[ix]->n_allocated++;
	/* rotate empty slabs */
	// 如果当前所有的chunks用完，旋转slab ring，定位到下一个slab
	if (!allocator->slab_stack[ix]->chunks)
	allocator->slab_stack[ix] = allocator->slab_stack[ix]->next;
	return chunk;
}
{% endhighlight %}
当client首次请求一定量的的内存时，slab system首先会将请求的size up align to 8的倍数，作为chunk size，然后再分配一页内存。去除slab info 结构体和color(padding)，之后那一页内剩下的共有n个chunk size。拿出1个chunk返回给client,剩下的(n-1)个chunk size挂接到当前的slab节点上。

**allocator_add_slab函数**
{% highlight cpp %}
static void
allocator_add_slab (Allocator *allocator,
guint ix,
gsize chunk_size)
{
	ChunkLink *chunk;
	SlabInfo *sinfo;
	gsize addr, padding, n_chunks, color = 0;
	// 计算出当前系统一页的内存大小
	gsize page_size = allocator_aligned_page_size (allocator, SLAB_BPAGE_SIZE (allocator, chunk_size));
	/* allocate 1 page for the chunks and the slab */
	gpointer aligned_memory = allocator_memalign (page_size, page_size - NATIVE_MALLOC_PADDING);
	guint8 *mem = aligned_memory; // 对齐到page 的虚拟内存地址
	guint i;
	if (!mem)
	{
		const gchar *syserr = "unknown error";
		#if HAVE_STRERROR
		syserr = strerror (errno);
		#endif
		mem_error ("failed to allocate %u bytes (alignment: %u): %s\n",
		(guint) (page_size - NATIVE_MALLOC_PADDING), (guint) page_size, syserr);
	}
	/* mask page address */
	addr = ((gsize) mem / page_size) * page_size;
	/* assert alignment */
	mem_assert (aligned_memory == (gpointer) addr);
	/* basic slab info setup */
	// 从下面知道每次分配一页时，slab_info总是在那个内存页的末尾
	sinfo = (SlabInfo*) (mem + page_size - SLAB_INFO_SIZE);
	sinfo->n_allocated = 0;
	sinfo->chunks = NULL;
	/* figure cache colorization */
	n_chunks = ((guint8*) sinfo - mem) / chunk_size;
	padding = ((guint8*) sinfo - mem) - n_chunks * chunk_size;
	if (padding)
	{
		color = (allocator->color_accu * P2ALIGNMENT) % padding;
		allocator->color_accu += allocator->config.color_increment;
	}
	/* add chunks to free list */
	// 将第一个free chunk的内存地址定为跳过mem之后color (padding)的地址
	chunk = (ChunkLink*) (mem + color);
	sinfo->chunks = chunk;
	// 将连续内存空间，用单向链表链接起来
	for (i = 0; i < n_chunks - 1; i++)
	{
		chunk->next = (ChunkLink*) ((guint8*) chunk + chunk_size);
		chunk = chunk->next;
	}
	chunk->next = NULL; /* last chunk */
	/* add slab to slab ring */
	allocator_slab_stack_push (allocator, ix, sinfo);
}

{% endhighlight %}

**一些有意思的MACRO定义：**
{% highlight cpp %}
/* optimized version of ALIGN (size, P2ALIGNMENT) */
#if     GLIB_SIZEOF_SIZE_T * 2 == 8  /* P2ALIGNMENT */
#define P2ALIGN(size)   (((size) + 0x7) & ~(gsize) 0x7)
#elif   GLIB_SIZEOF_SIZE_T * 2 == 16 /* P2ALIGNMENT */
#define P2ALIGN(size)   (((size) + 0xf) & ~(gsize) 0xf)
#else
#define P2ALIGN(size)   ALIGN (size, P2ALIGNMENT)
#endif

#define P2ALIGNMENT             (2 * sizeof (gsize)) 
#define NATIVE_MALLOC_PADDING   P2ALIGNMENT            /* per-page padding left for native malloc(3) see [1] */
#define SLAB_INFO_SIZE          P2ALIGN (sizeof (SlabInfo) + NATIVE_MALLOC_PADDING)
#define SLAB_BPAGE_SIZE(al,csz) (8 * (csz) + SLAB_INFO_SIZE)
{% endhighlight %}
如果要求分配8个字节的内存，csz=8，所以每次加载chunks，分配的内存必须能够包含8个chunks，加上SLAB_INFO_SIZE的大小。
{% highlight cpp %}
static void
allocator_slab_stack_push (Allocator *allocator,
                           guint      ix,
                           SlabInfo  *sinfo)
{
	/* insert slab at slab ring head */
	if (!allocator->slab_stack[ix])
	{
		sinfo->next = sinfo;
		sinfo->prev = sinfo;
	}
	else
	{
		  SlabInfo *next = allocator->slab_stack[ix], *prev = next->prev;
		  next->prev = sinfo;
		  prev->next = sinfo;
		  sinfo->next = next;
		  sinfo->prev = prev;
	}
	allocator->slab_stack[ix] = sinfo;
}

static gsize
allocator_aligned_page_size (Allocator *allocator,
                             gsize      n_bytes)
{
	gsize val = 1 << g_bit_storage (n_bytes - 1);
	val = MAX (val, allocator->min_page_size);
	return val;
}
{% endhighlight %}
`gsize val = 1 << g_bit_storage (n_bytes - 1);`用来将n_bytes表示的数字up align到2的n次幂的倍数。如果up align之后的数比min_page_size(4k=2^12)要大，那么函数返回的值就是val。也就是说allocator_aligned_page_size并不一定返回4k page size，或者说只要chunk size大于等于512，8 * 512 + 16 + 8 > 4096 (order >= 64)。
然而从以下的MACRO定义可知：
{% highlight cpp %}
#define MAX_SLAB_CHUNK_SIZE(al) (((al)->max_page_size - SLAB_INFO_SIZE) /8 )
#define MAX_SLAB_INDEX(al)      (SLAB_INDEX (al, MAX_SLAB_CHUNK_SIZE (al)) + 1)
{% endhighlight %}
allocator中的slabinfo slot个数是不会超过64的(只有63，而且chunk size=504)。也就是说`allocator_aligned_page_size`返回的永远会是一个page的size(4k)，而且一个page里最少会有8个chunks。

但是如何分配那一页内存呢？
{% highlight cpp %}
static gpointer
allocator_memalign (gsize alignment,  gsize memsize)          
{
	gpointer aligned_memory = NULL;
    gint err = ENOMEM;
#if     HAVE_COMPLIANT_POSIX_MEMALIGN
    err = posix_memalign (&aligned_memory, alignment, memsize);
#elif   HAVE_MEMALIGN
    errno = 0;
    aligned_memory = memalign (alignment, memsize);
    err = errno;
#elif   HAVE_VALLOC
    errno = 0;
    aligned_memory = valloc (memsize);
    err = errno;
#else
    /* simplistic non-freeing page allocator */
    mem_assert (alignment == sys_page_size);
    mem_assert (memsize <= sys_page_size);
    if (!compat_valloc_trash)
    {
        const guint n_pages = 16;
        guint8 *mem = malloc (n_pages * sys_page_size);
        err = errno;
        if (mem)
        {
            gint i = n_pages;
            guint8 *amem = (guint8*) ALIGN ((gsize) mem, sys_page_size);
            if (amem != mem)
            i--;        /* mem wasn't page aligned */
            while (--i >= 0)
            g_trash_stack_push (&compat_valloc_trash, amem + i * sys_page_size);
        }
    }
    aligned_memory = g_trash_stack_pop (&compat_valloc_trash);
#endif
    if (!aligned_memory)
    errno = err;
    return aligned_memory;
}
{% endhighlight %}
假如程序走到最后一个else语句，那么这个slab系统就有点复杂了。compat_valloc_trash是一个全局的静态指针`static GTrashStack *compat_valloc_trash = NULL; `这个函数首先会向系统一次性请求分配16 \* page_size(4k)大小的连续内存空间，然后将这些内存用GTrashStack串联起来，（要知道GTrashStack结构知道一个next域指向下一个区域，本身并不消耗内存），串联的单元就是一个页。这个就是`g_trash_stack_push`干的事情。  
之后调用g\_trash\_stack\_pop将第一个页的内存分配出去。

###slab内存释放的操作
slab内存释放的操作由slab_allocator_free_chunk来完成，chunk_size是up allign到8的倍数的内存大小。
{% highlight cpp %}
static void
slab_allocator_free_chunk (gsize chunk_size, gpointer mem)
{
	ChunkLink *chunk;
	gboolean was_empty;
	guint ix = SLAB_INDEX (allocator, chunk_size); // 找到chunk size所属的slab slot序号
	// 计算chunk size所决定的page的大小(从slab分配出去的chunk都有所属的page)
	gsize page_size = allocator_aligned_page_size (allocator, SLAB_BPAGE_SIZE (allocator, chunk_size));
	// 计算释放chunk的地址所在page的内存首地址
	gsize addr = ((gsize) mem / page_size) * page_size;
	/* mask page address */
	guint8 *page = (guint8*) addr;
	// slab info结构体在一个page的最后24字节
	SlabInfo *sinfo = (SlabInfo*) (page + page_size - SLAB_INFO_SIZE);
	/* assert valid chunk count */
	// 这个slab应该有chunks被分配出去，不然这个slab应该被回收给系统，稍后知道为什么。
	mem_assert (sinfo->n_allocated > 0);
	/* add chunk to free list */
    was_empty = sinfo->chunks == NULL;
	// 把这个chunk插入到free chunk list头
	chunk = (ChunkLink*) mem;
	chunk->next = sinfo->chunks;
	sinfo->chunks = chunk;
	sinfo->n_allocated--;
	/* keep slab ring partially sorted, empty slabs at end */
	// 如果之前所有的chunks都分配出去了，这时有了free chunk，就应该让slab slot知道
	if (was_empty)
    {
		/* unlink slab */
		SlabInfo *next = sinfo->next, *prev = sinfo->prev;
		next->prev = prev;
		prev->next = next;
		if (allocator->slab_stack[ix] == sinfo)
		allocator->slab_stack[ix] = next == sinfo ? NULL : next;
		/* insert slab at head */
		// 让slab slot直接指向这个有free chunk的slab，以方便下次分配这样的chunk时，直接获取。
		allocator_slab_stack_push (allocator, ix, sinfo);
    }
	/* eagerly free complete unused slabs */
	// 以下操作就是为什么要做mem_assert (sinfo->n_allocated > 0);原因
	// 当这个slab所有的chunk都free时，就可以将这个slab所在页面返回给系统，或者TrashStack内存池。
	if (!sinfo->n_allocated)
    {
		/* unlink slab */
		SlabInfo *next = sinfo->next, *prev = sinfo->prev;
		next->prev = prev;
		prev->next = next;
		if (allocator->slab_stack[ix] == sinfo)
		allocator->slab_stack[ix] = next == sinfo ? NULL : next;
		/* free slab */
		allocator_memfree (page_size, page);  // 返回给系统或者内存池
    }
}
{% endhighlight %}
将所有chunk都free的slab返回给系统或者内存池：
{% highlight cpp %}
static void
allocator_memfree (gsize memsize, gpointer mem)             
{
#if     HAVE_COMPLIANT_POSIX_MEMALIGN || HAVE_MEMALIGN || HAVE_VALLOC
	free (mem);
#else
	mem_assert (memsize <= sys_page_size);
    g_trash_stack_push (&compat_valloc_trash, mem);
#endif
}
{% endhighlight %}