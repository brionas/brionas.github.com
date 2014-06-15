---
layout: post
title: 跳表与无阻塞
description: 跳表与无阻塞
category: Data Structure
tags: skiplist
refer_author: Bean
refer_blog_addr:
refer_post_addr:
---

##跳表
跳表（亦称跳跃列表）是一个很有意思的数据结构，[Pugh](http://www.cs.umd.edu/~pugh/)通过它向人们展示了随机化数据结构的存在性，还证明了跳表跟二叉查找树是渐进逼近，即各项操作的复杂度是等价的。

为什么说跳表是很有意思呢？你可以说它是一个链表（单链表或者双链表，取决于实现，但必须是有序的），也可以说它是多个链表的并联！神奇吧，这正是跳表特别的地方。

先给张图看下

<img width="800px" src="/assets/image/2014-06/skiplist-nonblocking/sl.svg" type="image/xml+svg" />


上图中，以灰色填充的是跳表的root(或者称为head，其实没所谓)，包含了若干个前驱（Forward pointer）指针（在不引起歧义的情况下，以下简称为前驱），其中的数字表示层次（层高？这个图就需要上下颠倒过来了）。白色背景的是元素节点，节点按元素（这里是整数）的大小顺序依次衔接。我们从两个角度来看跳表的结构：

- 节点按顺序连接着，连接箭头在第一层表示，而每个节点都包含若干个前驱，每个前驱指向次序较后的节点。
- 当从查找的角度去看跳表，我们会从最高的链表开始查找，如果找不到需要的节点或者碰到次序较后的节点，那么往低一层的链表查，直到找到或者到达底层。譬如说要查找的是-150，首先从第6层（最高层）出发。由于第6层前驱为空，我们来到第5层，发现-83是第一个前驱，然而比-150大，于是往第4层走，这时的前驱是-174，比目标小，于是往前走，发现又是-83。这样我们在-174这里往第3层走。不错，这时前驱就是-150，我们找到了！

现在看下，什么是随机化？跳表的随机化在于节点前驱数目的分配。

为了维持O(log n)的查找效率，每次给节点分配的前驱数目会受到概率p的约束，即若节点在第k层出现，那么在第k+1层出现的概率为p。通常p取为 0.5。
如果p偏小，则最终跳表偏矮，节点显得稠密，查找效率偏低（在均摊意义上）；反之，则每个节点的前驱增加，使用更多的空间。

##无锁难题

解决并发环境中的资源竞争问题的实现有多种类别

- 互斥锁，锁机制被作为基本的手段，用来控制资源的访问顺序，然而，随着多核时代的带来，锁的一些问题也被成为大家议论的焦点
- 无锁算法， 依靠硬件提供的原子操作，将操作以事务的形式进行

然而，[ABA问题](http://en.wikipedia.org/wiki/ABA_problem) 把无锁算法难倒了！


由于跳表是并联链表，为简单起见，我们只分析单链表的插入和删除操作，在没有锁机制下，会遇到哪些问题？

### 链表插入

对于给定的链表，假设访问者有线程甲、乙、丙、丁等。

再假设当前的内存视图如下所示：

<img width="800px" src="/assets/image/2014-06/skiplist-nonblocking/a.svg" type="image/xml+svg" />

现在线程乙要插入E，并确定了插入位置就在A和B之间，即Insert(A，E)，于是执行E.Next := B

<img width="800px" src="/assets/image/2014-06/skiplist-nonblocking/b.svg" type="image/xml+svg" />

然后将E附在A后面，A.Next := E

<img width="800px" src="/assets/image/2014-06/skiplist-nonblocking/c.svg" type="image/xml+svg" />

现在我们分析下这个插入操作存在什么问题：

1. 线程乙在执行E.Next := B时，甲刚刚修改了B.Next，把C删除（未必是回收），当然乙不在乎！

    <img width="600px" style="align:left;" src="/assets/image/2014-06/skiplist-nonblocking/1.svg" type="image/xml+svg" />

1. 由于E只有乙可见，E以及E.Next（注意，不是指它所指的内存）不会被别的线程改写，但如果修改了B呢？例如回收B的内存，或者改变了*值*！这样的话，E.Next := B的结果是不可意料的！


1. 如果在乙执行这个操作时，甲先移除了B，那么A.Next = C， 如果BC间没有其它节点插入。

    <img width="600px" style="align:left;" src="/assets/image/2014-06/skiplist-nonblocking/2.svg" type="image/xml+svg" />

1. 如果丙也在A和B之间插入一个节点E'。如果丙先插入成功，如图所示。 如果乙继续他的插入操作，必有A.Next = E，这样丙插入的节点就丢失了，呈现了所谓的Lost Update问题。反之，情况相仿。

    <img width="600px" style="align:left;" src="/assets/image/2014-06/skiplist-nonblocking/3.svg" type="image/xml+svg" />

1. 在执行A.Next := E 前，如果A被删除了，甚至是回收了。你该知道执行A.Next := E意味着什么！

    <img width="600px" style="align:left;" src="/assets/image/2014-06/skiplist-nonblocking/4.svg" type="image/xml+svg" />

**CAS**是现代处理器中能够对一处内存进行原子比较更新的指令。在排除ABA问题存在情况下，即在假定A和B节点在事务进行期间不被删除的情况下，多个在AB间竞争插入的问题可以通过CAS来解决，当然前提是A.Next要CAS能支持，幸好指针类型是没问题的！

{% highlight Ada%}
loop
   B := A.Next;
   E.Next := B;
   exit when CAS(A.Next, B, E);
   --  if A.Next = B then
   --     A.Next := E;
   --     return true;
   --  else
   --     return false;
   --  end if;
end loop;
{% endhighlight %}
### 链表删除

链表插入操作过程中遇到的ABA问题，是由于删除操作引起的。尽管如此，我们还是要分析下，删除操作遇到的难题！

现在甲要删除E，于是执行事务：

{% highlight Ada%}
E := A.Next;
B := E.Next;
if A.Next = E and E.Next = B then
   A.Next := B;
end if;   
{% endhighlight %}

这样一个事务包含了三个独立的操作(两个比较，一个赋值) ，每一个操作前后都存在并发冲突。

1. 判断A.Next = E时，可能有新的节点插进来，使得A.Next = Z。
1. 判断E.Next = B时，可能有新的节点插进来，使得E.Next = W。
1. 在执行A.Next := B时，可能线程丁刚刚删除了B。
1. 在执行A.Next := B时，可能线程丙刚刚删除了A。

在假定第34种情况不发生的情况下（例如只有一个回收线程，而且是延迟回收的），有没有类似CAS那样的指令来避免前2种呢？理论上可采用**DCAS** (Double Compare And Swap的缩写，不是Double Words Compare And Swap)，**DCAS**能够原子地比较两处内存的值，并设置第一处内存。很可惜，目前的硬件没有实用的**DCAS**！原则上说，如果一个事务包含n个内存不变的断言就需要**n-CAS**!

{% highlight Ada%}
B := E.Next;
function DCAS (A.Next, E, E.Next, B, B) return boolean is
begin
   if A.Next = E and E.Next = B then
      A.Next := B;
      return true;
   end if;
   return false;
end DCAS;
{% endhighlight %}

有些文章生成DCAS能够绕过ABA问题来安全地删除节点，那是天真的想法！可以断言，在不使用辅助数据结构的条件下，正确的删除操作是不能保证的！
至于无阻塞辅助数据结构，推荐参考

- [NBAda](https://gitorious.org/nbada/nbada)
- [libcds](http://libcds.sourceforge.net/)

##实践中的跳表

###非阻塞

目前而言，学术界多数关于跳表的论文，它们的题目大多是non-blocking，即非阻塞！事实上，non-blocking包含了三个层次：

1. Wait-free (无延缓)
1. Lock-free （无锁）
1. Obstruction-free （无阻塞）

关于这些定义的细节，请参考[wikipedia](http://en.wikipedia.org/wiki/Non-blocking_synchronization)。

无阻塞意味着，线程间是无感知的并发，相互间 不依赖，不排斥。

但是目前的论文大多满足第三类Obstruction-free，而非Lock-free。他们的实现都围绕着ABA问题，要么不提供直接的物理删除(对应的，称为逻辑删除，即标记)，要么使用辅助数据结构（本质上可以归类为GC，如Hazard Pointer），要么在外层作读写同步。

###跳表的插入和逻辑删除
对于链表而言，查找一般是插入和删除必须进行的操作。对于跳表也不例外，但是又有特别之处：在查找过程中，通过记录跳转位置，可以使得插入和删除都是log(n)的！

对于只有逻辑删除的情形下，删除只是对节点做标记；而对于写者间的冲突，不影响读者，而**CAS**保证没有**Lost Update**！向跳表各不相同的节点后面插入新的节点的操作之间是不需要同步的，亦即，可以并发地进行；但要注意，竞争同一位置，会引起开销问题，通常竞争失败者可通过让自己被抢占来避免激烈的竞争。这样一来，某些线程可能陷入非预想的等待（重试）时间，因此不是wait-free；但是并发线程间并不互斥，竞争是隐蔽的，一个线程可能一直等待下去，甚至而死，但不会影响其他线程，因此是lock-free的。

如果引入物理删除，那么GC也是必要的，Hazard Pointer或者RCU都是读者友好的，可以考虑使用。但只能达到Obstruction-free！

逻辑删除的实现可参考LevelDB，而带GC的请参考[stasis](http://code.google.com/p/stasis)，它包含了一个具有Hazard Pointer支持的跳表。

### Demo

作为练习，我用Ada写了一个[Demo](https://github.com/lyarbean/skip-list)，同样只支持逻辑删除操作，目前只提供x86_64的支持，还在修改中。其中的Insert函数篇幅较大，但覆盖了上面提到的各种并发冲突的情况。

