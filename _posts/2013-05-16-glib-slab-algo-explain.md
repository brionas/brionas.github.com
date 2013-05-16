---
layout: post
title: "glib��slab�㷨ʵ��ѧϰ"
description: slab�������Ϊ�˽���ڲ��ڴ���Ƭ�����⣬��linux�ں�����buddy systemһ��������ں��ڴ��������Ҫ����slab��linux�ں��е�ʵ�ֵ�ǰ��Щ���ѣ����ǲ�����Щ�����Ķ��Ĵ������˽�slab�㷨���������̡�GLIB��ʵ�ַǳ�clear��������Ϊslab�㷨��ʵ��ѧϰ�����š�
category: "memory_management"
tags: [slab, glib, Դ���Ķ�]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/categories.html
refer_post_addr: http://zeroli.github.io/memory_management/2013/03/24/glib-slab-algo-explain/
---
{% include JB/setup %}

slab�������Ϊ�˽���ڲ��ڴ���Ƭ�����⣬��linux�ں�����buddy systemһ��������ں��ڴ��������Ҫ����slab��linux�ں��е�ʵ�ֵ�ǰ��Щ���ѣ����ǲ�����Щ�����Ķ��Ĵ������˽�slab�㷨���������̡�GLIB��ʵ�ַǳ�clear��������Ϊslab�㷨��ʵ��ѧϰ�����š�  
slab��GLIB�е�ʵ������ļ���gslice.h/c�����������ʵ���ļ��У��и����ӵģ�֧�ֶ��̸߳����magazine caching�㷨��

##0. slab�㷨����������
����slab�㷨�������������Բ鿴[wiki](http://en.wikipedia.org/wiki/Slab_allocation%22%3Ehttp://en.wikipedia.org/wiki/Slab_allocation)


###slab�ṹ�嶨��
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
![](/assets/image/1345812003_5463.png)

�ӽṹ�����֪����ÿ��slabά����ǰ��slab��link prev�ͺ����slab��link prev����֪����������������һ��˫��������(slab ring)������֮�⣬��ά����һ��chunkָ�룬����ָ��free chunk�ĵ�������

###allocator�ṹ�嶨��
{% highlight cpp %}
...
/* slab allocator */
GMutex slab_mutex;
SlabInfo **slab_stack; /* array of MAX_SLAB_INDEX (allocator) */
guint color_accu;
} Allocator;
{% endhighlight %}
�����ݽṹ��֪��ȫ�ֱ���allocator��ά����һ��ָ�����飬��ԱΪSlabInfo*ָ�룬ָ��һ������slab˫������Ҳ��ǰ����ܵ�slab ring��ÿ����Ա��slabinfo*ָ���slab ring�������chunks��size����������ͬ�ģ������һ��slab ring�����chunkȫ������8 (bytes)���ڶ���slab ring��ȫ������8\*2 (bytes)���Դ����ƣ�һֱ������chunk size����ʼ��ʱ�����еĳ�Ա�����ʼ��Ϊnull������ǰslab systemû��free list��

###slab�ڴ���������Ĺ���
{% highlight cpp %}
static gpointer
slab_allocator_alloc_chunk (gsize chunk_size)
{
ChunkLink *chunk;
guint ix = SLAB_INDEX (allocator, chunk_size); // �ҵ�chunk_size����Ӧ��slab slot
/* ensure non-empty slab */
// ���û��free list����Ԥ�ȷ���һҳ������slot��һҳ���ܻ��кܶ�chunks��
if (!allocator->slab_stack[ix] || !allocator->slab_stack[ix]->chunks)
allocator_add_slab (allocator, ix, chunk_size);
/* allocate chunk */
// �����һ��chunk��client�������chunks�ҽӵ���ǰslab��
chunk = allocator->slab_stack[ix]->chunks;
allocator->slab_stack[ix]->chunks = chunk->next;
allocator->slab_stack[ix]->n_allocated++;
/* rotate empty slabs */
// �����ǰ���е�chunks���꣬��תslab ring����λ����һ��slab
if (!allocator->slab_stack[ix]->chunks)
allocator->slab_stack[ix] = allocator->slab_stack[ix]->next;
return chunk;
}
{% endhighlight %}
��client�״�����һ�����ĵ��ڴ�ʱ��slab system���ȻὫ�����size up align to 8�ı�������Ϊchunk size��Ȼ���ٷ���һҳ�ڴ档ȥ��slab info �ṹ���color(padding)��֮����һҳ��ʣ�µĹ���n��chunk size���ó�1��chunk���ظ�client,ʣ�µ�(n-1)��chunk size�ҽӵ���ǰ��slab�ڵ��ϡ�

**allocator_add_slab����**
{% highlight cpp %}
static void
allocator_add_slab (Allocator *allocator,
guint ix,
gsize chunk_size)
{
ChunkLink *chunk;
SlabInfo *sinfo;
gsize addr, padding, n_chunks, color = 0;
// �������ǰϵͳһҳ���ڴ��С
gsize page_size = allocator_aligned_page_size (allocator, SLAB_BPAGE_SIZE (allocator, chunk_size));
/* allocate 1 page for the chunks and the slab */
gpointer aligned_memory = allocator_memalign (page_size, page_size - NATIVE_MALLOC_PADDING);
guint8 *mem = aligned_memory; // ���뵽page �������ڴ��ַ
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
// ������֪��ÿ�η���һҳʱ��slab_info�������Ǹ��ڴ�ҳ��ĩβ
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
// ����һ��free chunk���ڴ��ַ��Ϊ����mem֮��color (padding)�ĵ�ַ
chunk = (ChunkLink*) (mem + color);
sinfo->chunks = chunk;
// �������ڴ�ռ䣬�õ���������������
for (i = 0; i < n_chunks - 1; i++)
{
chunk->next = (ChunkLink*) ((guint8*) chunk + chunk_size);
chunk = chunk->next;
}
chunk->next = NULL; /* last chunk */
/* add slab to slab ring */
allocator_slab_stack_push (allocator, ix, sinfo);
}
}
{% endhighlight %}

**һЩ����˼��MACRO���壺**
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
���Ҫ�����8���ֽڵ��ڴ棬csz=8������ÿ�μ���chunks��������ڴ�����ܹ�����8��chunks������SLAB_INFO_SIZE�Ĵ�С��
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
`gsize val = 1 << g_bit_storage (n_bytes - 1);`������n_bytes��ʾ������up align��2��n���ݵı��������up align֮�������min_page_size(4k=2^12)Ҫ����ô�������ص�ֵ����val��Ҳ����˵allocator_aligned_page_size����һ������4k page size������˵ֻҪchunk size���ڵ���512��8 * 512 + 16 + 8 > 4096 (order >= 64)��
Ȼ�������µ�MACRO�����֪��
{% highlight cpp %}
#define MAX_SLAB_CHUNK_SIZE(al) (((al)->max_page_size - SLAB_INFO_SIZE) /8 )
#define MAX_SLAB_INDEX(al)      (SLAB_INDEX (al, MAX_SLAB_CHUNK_SIZE (al)) + 1)
{% endhighlight %}
allocator�е�slabinfo slot�����ǲ��ᳬ��64��(ֻ��63������chunk size=504)��Ҳ����˵`allocator_aligned_page_size`���ص���Զ����һ��page��size(4k)������һ��page�����ٻ���8��chunks��

������η�����һҳ�ڴ��أ�
{% highlight cpp %}
static gpointer
allocator_memalign (gsize alignment,
                    gsize memsize)
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
��������ߵ����һ��else��䣬��ô���slabϵͳ���е㸴���ˡ�compat_valloc_trash��һ��ȫ�ֵľ�ָ̬��`static GTrashStack *compat_valloc_trash = NULL; `����������Ȼ���ϵͳһ�����������16 \* page_size(4k)��С�������ڴ�ռ䣬Ȼ����Щ�ڴ���GTrashStack������������Ҫ֪��GTrashStack�ṹ֪��һ��next��ָ����һ�����򣬱����������ڴ棩�������ĵ�Ԫ����һ��ҳ���������`g_trash_stack_push`�ɵ����顣  
֮�����g\_trash\_stack\_pop����һ��ҳ���ڴ�����ȥ��

###slab�ڴ��ͷŵĲ���
slab�ڴ��ͷŵĲ�����slab_allocator_free_chunk����ɣ�chunk_size��up allign��8�ı������ڴ��С��
{% highlight cpp %}
static void
slab_allocator_free_chunk (gsize    chunk_size,
                           gpointer mem)
{
  ChunkLink *chunk;
  gboolean was_empty;
  guint ix = SLAB_INDEX (allocator, chunk_size); // �ҵ�chunk size������slab slot���
  // ����chunk size��������page�Ĵ�С(��slab�����ȥ��chunk����������page)
  gsize page_size = allocator_aligned_page_size (allocator, SLAB_BPAGE_SIZE (allocator, chunk_size));
  // �����ͷ�chunk�ĵ�ַ����page���ڴ��׵�ַ
  gsize addr = ((gsize) mem / page_size) * page_size;
  /* mask page address */
  guint8 *page = (guint8*) addr;
  // slab info�ṹ����һ��page�����24�ֽ�
  SlabInfo *sinfo = (SlabInfo*) (page + page_size - SLAB_INFO_SIZE);
  /* assert valid chunk count */
  // ���slabӦ����chunks�������ȥ����Ȼ���slabӦ�ñ����ո�ϵͳ���Ժ�֪��Ϊʲô��
  mem_assert (sinfo->n_allocated > 0);
  /* add chunk to free list */
  was_empty = sinfo->chunks == NULL;
  // �����chunk���뵽free chunk listͷ
  chunk = (ChunkLink*) mem;
  chunk->next = sinfo->chunks;
  sinfo->chunks = chunk;
  sinfo->n_allocated--;
  /* keep slab ring partially sorted, empty slabs at end */
  // ���֮ǰ���е�chunks�������ȥ�ˣ���ʱ����free chunk����Ӧ����slab slot֪��
  if (was_empty)
    {
      /* unlink slab */
      SlabInfo *next = sinfo->next, *prev = sinfo->prev;
      next->prev = prev;
      prev->next = next;
      if (allocator->slab_stack[ix] == sinfo)
        allocator->slab_stack[ix] = next == sinfo ? NULL : next;
      /* insert slab at head */
      // ��slab slotֱ��ָ�������free chunk��slab���Է����´η���������chunkʱ��ֱ�ӻ�ȡ��
      allocator_slab_stack_push (allocator, ix, sinfo);
    }
  /* eagerly free complete unused slabs */
  // ���²�������ΪʲôҪ��mem_assert (sinfo->n_allocated > 0);ԭ��
  // �����slab���е�chunk��freeʱ���Ϳ��Խ����slab����ҳ�淵�ظ�ϵͳ������TrashStack�ڴ�ء�
  if (!sinfo->n_allocated)
    {
      /* unlink slab */
      SlabInfo *next = sinfo->next, *prev = sinfo->prev;
      next->prev = prev;
      prev->next = next;
      if (allocator->slab_stack[ix] == sinfo)
        allocator->slab_stack[ix] = next == sinfo ? NULL : next;
      /* free slab */
      allocator_memfree (page_size, page);  // ���ظ�ϵͳ�����ڴ��
    }
}
{% endhighlight %}
������chunk��free��slab���ظ�ϵͳ�����ڴ�أ�
{% highlight cpp %}
static void
allocator_memfree (gsize    memsize,
                   gpointer mem)
{
#if     HAVE_COMPLIANT_POSIX_MEMALIGN || HAVE_MEMALIGN || HAVE_VALLOC
  free (mem);
#else
  mem_assert (memsize <= sys_page_size);
  g_trash_stack_push (&compat_valloc_trash, mem);
#endif
}
{% endhighlight %}