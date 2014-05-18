---
layout: post
title: nginx内存管理-内存池
description: nginx内存管理-内存池
category: 源码分析 C++
tags: nginx 内存池
refer_author: rqiu
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

nginx内存管理-内存池
===================



**Nginx**是一款轻量级的Web
服务器／反向代理服务器及电子邮件（IMAP/POP3）代理服务器,
拥有占用内存少，并发性能好，稳定性好等特点，目前应用范围非常广。本文将基于nginx-1.4.7对其内存池进行分析。

#### 数据结构

nginx内存池的相关操作主要在文件./src/core/ngx\_palloc.h/c中，以下为其主要数据结构。

	// 内存池的内存管理模块，即内存池的头部，通过其对整个内存池进行管理
	struct ngx_pool_t { 
	    ngx_pool_data_t       d;         // 内存池的数据块  
	    size_t                max;       // 数据块中可申请的小块内存的最大值  
	    ngx_pool_t           *current;   // 指向当前可分配内存的内存数据块  
	    ngx_chain_t          *chain;     // 该指针挂接一个ngx_chain_t结构，便于进行内存池的抽象数据管理 
	    ngx_pool_large_t     *large;     // 指向大块内存链表的头结点，nginx中，大块内存由malloc直接申请
	    ngx_pool_cleanup_t   *cleanup;   // 指向一个特殊资源管理链表，在内存池销毁时对相关资源进行清理。
	    ngx_log_t            *log;       // 日志  
	};
	 
	 
	 // 内存池的数据块，内存池分配内存在由这些数据块构成的数据块链表中进行 
	 
	typedef struct {
	    u_char               *last;    // 当前数据块可分配内存的起始位置  
	    u_char               *end;     // 当前数据块可分配内存结束位置  
	    ngx_pool_t           *next;    // 链接到下一个数据块
	    ngx_uint_t            failed;  // 记录当前数据块内存分配不能满足需求的失败次数  
	} ngx_pool_data_t;  
	 
	 
	 
	// 内存池的大块内存节点， 通过next构成一个大块内存的链表
	struct ngx_pool_large_t {  
	    ngx_pool_large_t     *next;  // 指向下一个大块内存节点
	    void                 *alloc; // 指向为当前节点分配的内存
	};
	 
	 
	// 特殊资源管理节点， 通过next构成一个链表
	struct ngx_pool_cleanup_s {
	    ngx_pool_cleanup_pt   handler; //资源清除函数
	    void                 *data; //指向需要清除的数据
	    ngx_pool_cleanup_t   *next ; // 指向下一个需要清除的数据
	};

#### 内存结构

#### ![](/assets/image/2014-05/2014-05-14-nginx-memory-pool/nginx-memroy-structure.png)

#### 基本操作
####  创建内存池
    
	ngx_pool_t* ngx_create_pool (size_t size, ngx_log_t *log)
	{
	    ngx_pool_t  * p;
	 
	 
	    // 申请一块大小为size的内存
	    p = ngx_memalign( NGX_POOL_ALIGNMENT, size, log);
	    if ( p == NULL) {
	        return NULL;
	    }
	    //初始化数据结构成员
	    p ->d. last = (u_char *) p + sizeof(ngx_pool_t );
	    p ->d. end = (u_char *) p + size;
	    p ->d. next = NULL;
	    p ->d. failed = 0;
	 
	 
	    size = size - sizeof (ngx_pool_t) ;
	    // 可分配小块内存的最大值不能超过NGX_MAX_ALLOC_FROM_POOL 
	    p ->max = (size < NGX_MAX_ALLOC_FROM_POOL ) ? size : NGX_MAX_ALLOC_FROM_POOL;
	 
	 
	    p ->current = p;
	    p ->chain = NULL ;
	    p ->large = NULL ;
	    p ->cleanup = NULL ;
	    p ->log = log;
	 
	 
	    return p;
	}

     创建一个初始节点大小为size的内存池，需要注意几点：

-   size的大小必须介于sizeof(ngx\_pool\_t)和NGX\_MAX\_ALLOC\_FROM\_POOL(4K)之间，小于
    sizeof(ngx\_pool\_t)的话初始节点没有足够的空间来存放内存池的管理信息，可能造成程序Crash,
    超过NGX\_MAX\_ALLOC\_FROM\_POOL会造成内存浪费
-   实际可用的内存小于size，因为初始节点会占用部分内存存放内存池的管理信息，且该节点的max字段会选择size-sizeof(ngx\_pool\_t)和NGX\_MAX\_ALLOC\_FROM\_POOL的较小值。
-   创建内存池时，大块内存链表和特殊资源清理链表为空。

####  申请内存 

     nginx内存申请有好几个函数

	void * ngx_palloc (ngx_pool_t *pool, size_t size )
	
	从内存池中分配一块大小为size的内存，分配内存的起始地址进行内存对齐。
	
	void * ngx_pnalloc(ngx_pool_t *pool , size_t size)
	
	同ngx_palloc 的区别是分配内存的起始地址不进行内存对齐
	
	void * ngx_pcalloc (ngx_pool_t *pool, size_t size )
	
	调用ngx_palloc分配内存，并进行清零操作。
	
	void * ngx_pmemalign (ngx_pool_t *pool, size_t size , size_t alignment)
	
	调用ngx_palloc分配内存，不论大小都挂在大块内存链表上。


由于以上几种函数都是ngx\_palloc的变种，我们主要介绍一下ngx\_palloc , nginx内存分配分为小块内存分配和大块内存分配，ngx\_palloc会根据内存大小来选择进行何种分配方式。

	void *
	ngx_palloc (ngx_pool_t *pool, size_t size )
	{
	    u_char      *m ;
	    ngx_pool_t  * p;
	 
	 
	    // 依据size的大小决定进行大块内存分配还是小块内存分配
	    if ( size <= pool-> max) {
	        // 小块内存分配
	        
	        // 定位到目前可分配内存的数据块
	        p = pool ->current;
	        
	        // 选择有足够可分配内存大小的数据块进行内存分配
	        do {
	            // 按NGX_ALIGNMENT进行内存对齐
	            m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT) ;
	            
	            
	            if (( size_t) (p-> d.end - m) >= size) {
	                p ->d. last = m + size ;
	 
	 
	                return m;
	            }
	            p = p-> d.next ;
	        } while (p );
	        
	        // 没有足够可分配内存大小的数据块，新申请数据块进行内存分配
	        return ngx_palloc_block (pool, size) ;
	    }
	    // 大块内存分配
	    return ngx_palloc_large (pool, size) ;
	}
	 
	static void *
	ngx_palloc_block (ngx_pool_t *pool, size_t size )
	{
	    // 申请新的数据块并分配size大小的内存
	    u_char      *m ;
	    size_t       psize;
	    ngx_pool_t  * p, *new, *current ;
	 
	    psize = ( size_t) (pool-> d.end - ( u_char *) pool);
	 
	 
	    m = ngx_memalign( NGX_POOL_ALIGNMENT, psize , pool->log );
	    if ( m == NULL) {
	        return NULL;
	    }
	 
	    new = ( ngx_pool_t * ) m;
	 
	    new ->d. end = m + psize;
	    new ->d. next = NULL;
	    new ->d. failed = 0;
	 
	 
	    m += sizeof (ngx_pool_data_t) ;
	    m = ngx_align_ptr(m, NGX_ALIGNMENT);
	    new ->d. last = m + size ;
	 
	 
	    // 定位到数据块链表尾节点，并构建current链表，便于快速定位到可分配内存的数据块
	    current = pool ->current;
	    for ( p = current ; p-> d.next ; p = p->d .next) {
	        if ( p->d .failed++ > 4) {
	            current = p-> d.next ;
	        }
	    }
	    
	    // 把新申请的数据块挂到数据块链表的尾节点
	    p ->d. next = new ;
	    pool->current = current ? current : new ;
	 
	    return m;
	}
	 
	static void *
	ngx_palloc_large (ngx_pool_t *pool, size_t size )
	{   
	    // 大块内存分配
	    void              *p ;
	    ngx_uint_t         n;
	    ngx_pool_large_t  * large;
	    
	    // 从系统中申请大小为size的大块内存
	    p = ngx_alloc (size, pool-> log);
	    if ( p == NULL) {
	        return NULL;
	    }
	 
	    n = 0 ;
	    // 遍历大块内存链表头三个节点，把新申请的内存挂到内存为空的节点上
	    for ( large = pool->large ; large; large = large->next) {
	        if ( large->alloc == NULL) {
	            large ->alloc = p;
	            return p;
	        }
	        
	        if ( n++ > 3 ) {
	            break;
	        }
	    }
	    // 从内存池中新申请一块内存作为一个新的内存节点，把上面新申请的内存挂到该节点上，并把该节点挂到大块内存链表的头部，形成新的大块内存链表。
	    large = ngx_palloc (pool, sizeof(ngx_pool_large_t ));
	    if ( large == NULL) {
	        ngx_free (p) ;
	        return NULL;
	    }
	 
	    large ->alloc = p;
	    large ->next = pool-> large;
	    pool->large = large;
	 
	    return p;
	}

####释放内存

	ngx_int_t
	ngx_pfree(ngx_pool_t *pool , void *p )
	{
	    // 严格意义上这里应该叫大块内存释放，它只对大块内存链表的内存进行释放，其他内存会在内存池销毁时统一释放
	    ngx_pool_large_t  * l;
	    for ( l = pool->large ; l; l = l ->next) {
	        if ( p == l->alloc ) {
	            ngx_log_debug1 (NGX_LOG_DEBUG_ALLOC, pool->log, 0,
	                           "free: %p", l ->alloc) ;
	            ngx_free (l-> alloc);
	            l ->alloc = NULL ;
	            return NGX_OK;
	        }
	    }
	 
	    return NGX_DECLINED;
	}

####重置内存池
   
 重置内存池不同于销毁内存池，他只对销毁大块内存链表，并把内存池里所有已分配的内存标记为无效，而不进行实际的内存释放操作

	void
	ngx_reset_pool (ngx_pool_t *pool)
	{
	    ngx_pool_t        *p ;
	    ngx_pool_large_t  * l;
	    
	    // 释放大块内存链表的内存
	    for ( l = pool->large ; l; l = l ->next) {
	        if ( l->alloc ) {
	            ngx_free (l-> alloc);
	        }
	    }
	    
	    // 把大块内存链表置空
	    pool->large = NULL;
	 
	 
	    // 把内存池所有已分配内存标记为无效
	    for ( p = pool; p ; p = p->d .next) {
	        p ->d. last = (u_char *) p + sizeof(ngx_pool_t );
	    }
	}

####销毁内存池

    内存池销毁时会对与该内存池相关的内存或资源进行系统的释放。

	void
	ngx_destroy_pool (ngx_pool_t *pool)
	{
	    ngx_pool_t          *p , * n;
	    ngx_pool_large_t    * l;
	    ngx_pool_cleanup_t  * c;
	    
	    // 遍历特殊资源清理链表，清理特殊资源（关闭，删除文件等）。
	    for ( c = pool->cleanup ; c; c = c ->next) {
	        if ( c->handler ) {
	            ngx_log_debug1 (NGX_LOG_DEBUG_ALLOC, pool->log, 0,
	                           "run cleanup: %p", c );
	            c ->handler( c->data );
	        }
	    }
	 
	    // 释放大块内存链表内存
	    for ( l = pool->large ; l; l = l ->next) {
	 
	        ngx_log_debug1 (NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc );
	        if ( l->alloc ) {
	            ngx_free (l-> alloc);
	        }
	    }
	 
	 
	    // 释放内存池所有内存
	    for ( p = pool, n = pool ->d. next; /* void */; p = n, n = n->d .next) {
	        ngx_free (p) ;
	 
	 
	        if ( n == NULL) {
	            break;
	        }
	    }
	}

#### 特殊资源管理

     
即将大小为size的特殊资源添加到内存池进行管理，内存池会在销毁时对该资源进行清理，例如我们如果想要在内存池销毁时关闭一个文件，我们可以构造一个新的特殊资源清理节点ngx\_pool\_cleanup\_t ;挂在特殊资源清理链表的头部形成新的链表，该节点的handle指向文件关闭函数***void ngx\_pool\_delete\_file (void \*data)***，data指向文件名，这样内存池销毁时会通过该节点的handle对data进行处理。

	ngx_pool_cleanup_t *
	ngx_pool_cleanup_add (ngx_pool_t *p, size_t size )
	{
	    // 添加特殊资源管理节点
	    ngx_pool_cleanup_t  * c;
	 
	    // 从内存池中分配内存构造特殊资源管理节点
	    c = ngx_palloc (p, sizeof( ngx_pool_cleanup_t));
	    if ( c == NULL) {
	        return NULL;
	    }
	 
	    // 从内存池中分配内存用来存放特殊资源
	    if ( size) {
	        c ->data = ngx_palloc( p, size);
	        if ( c->data == NULL) {
	            return NULL;
	        }
	    } else {
	        c ->data = NULL ;
	    }
	 
	    c ->handler = NULL ;
	    // 将新的节点插入特殊资源管理链表的头部形成新的链表
	    c ->next = p-> cleanup;
	 
	    p->cleanup = c;
	    ngx_log_debug1 (NGX_LOG_DEBUG_ALLOC, p->log , 0 , "add cleanup: %p" , c) ;
	 
	    return c;
	}
 
 
 
// 通过handle对内存池进行文件清理操作
	
	void ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd);
 
 
// 关闭data指定文件

	void ngx_pool_cleanup_file(void *data);
 
 
// 删除data指定文件

	void ngx_pool_delete_file(void *data);

#### 内存特性

1.  避开了频繁的malloc、free操作，节省了内存申请释放的时间，提高了系统性能
2.  多次申请，一次释放，内存池在使用过程当中，只需要申请内存并使用，不需要考虑内存的释放，内存池销毁时会释放该内存池相关的所有内存。这个是nginx内存管理比较特殊的地方，虽然会增加系统的内存负担，但简化了内存操作，便于系统进行内存管理。
3.  通过特殊资源管理链表，采用回调函数增加了内存管理的灵活性，实际上对内存池的功能进行了很大的扩展，使内存池具有资源管理的功能，通过将一些特殊资源（文件关闭，文件删除）注册的特殊资源管理链表上，内存池销毁时会完成相应的资源清理功能。
4.  与基于slab的内存池相比，减少了内存碎片。一次申请一个内存页，减少了外部碎片，在一个数据块内会分配不同大小的内存块，同时对大小内存进行分开管理，一定程度上减少了内部碎片的产生。
5.  小块内存管理时， 在定位可分配内存的数据块时，
    当一个数据块超过4次不能满足申请时，这个数据块在以后的申请中会直接忽略，这样虽然会在加快可分配内存数据块的查找，但在一定程度上也造成内存的浪费。
6.  内存池生命周期中，由于只申请不释放，在某些极端情况（生命周期特别长，申请内存特别多）的情况下，内存池会占用相当多的资源，进而影响系统性能。

nginx内存池采用较为精简的结构实现了比较全面的内存管理功能，为提高内存管理的效率和性能，在许多地方进行了一定的折中，特别是其多次申请，一次释放和采用回调函数管理特殊资源的思想为我们以后进行资源管理提供了很好的思路。

#### 参考文献：

[http://blog.csdn.net/livelylittlefish/article/details/6586946](http://blog.csdn.net/livelylittlefish/article/details/6586946)

[http://www.cnblogs.com/didiaoxiong/p/nginx\_memory.html](http://www.cnblogs.com/didiaoxiong/p/nginx_memory.html)

[http://ialloc.org/2013/ngx-notes-memory-pool/](http://ialloc.org/2013/ngx-notes-memory-pool/)

[http://tengine.taobao.org/book/chapter\_02.html\#ngx-pool-t-100](http://tengine.taobao.org/book/chapter_02.html#ngx-pool-t-100)