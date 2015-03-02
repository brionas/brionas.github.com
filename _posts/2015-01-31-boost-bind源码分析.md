---
layout: post
title: "boost::bind源码分析"
description: "本文简要分析了boost中的bind源码"
category: c++ boost
tags: [boost, bind]
refer_author: Jin-Hua Ouyang
refer_blog_addr: http://confluence.briontech.com/pages/viewpage.action?pageId=19598598
refer_post_addr: http://confluence.briontech.com/pages/viewpage.action?pageId=19598598
---

##1  Introduction

相信大家对于boost::bind的用法已经非常熟悉，它相当于bind1st, bind2nd的一个超级加强版。

考察下面一个有趣的代码，它能通过编译吗？有运行错误吗？
{% highlight cpp %} 
void func(int){
}
 
void use_func(boost::function<void (int, int) > f)
{
     f(1,2);
}
void test_bind2()
{
    use_func(boost::bind(&func, _1));
}
{% endhighlight %} 

##2 boost::bind 要解决的问题
boost::bind的本质就是预先把函数指针和参数（包括占位符参数和实际参数）存储起来，等到调用的时候再进行真正的函数调用。

说白了，它就是一个functor。在这句话里涉及到几个关键词，函数指针，参数的存储，调用。

很自然的，我们可以想象它必然是这个样子的：

{% highlight cpp %} 
namespace boost
{
   template<...> // 一大堆的模板，可能的样子是 template< template R, class F, class Arg1, class Arg2...>
   class bind_t : public ... // 一些父类
   {
   public:
      RETURN_TYPE operator() (Arg1 ag, ...) // 几个参数
   private:
      void* func_ptr;
      Args1 arg;
      Args2 ...;
   };
}
{% endhighlight %}

上面只是我们的猜测，真正的boost::bind可能跟我们想的不一样。但是它一定要包含：

1) 函数指针存储                     -->  对应我们的Functor ;

2) 参数存储  （占位符，和实际参数)  --> 对应我们的Args ...

3) 调用                             --> operator()

##3 实现
由于boost::bind是一堆大神写的，肯定代码没有上面的那么丑。让我们一步一步来完善它。
 
###1）参数存储
我们把参数用一个类型来存储起来，让代码更干净一点。不妨把这个参数类命名为：storage。

由于boost::bind最多支持9个参数的情况，因此对应的，我们可以定义九个类。
 
{% highlight cpp %} 
template<class A1> struct storage1
{
explicit storage1 (A1 a1) : a1_(a1) {}
A1 a1_;
};
 
template<class A1, class A2> struct storage2 : public storage1<A1>
{
storage2 (A1 a1, A2 a2) : stroage1<A1>(a1), a2_(a2) {}
A2 a2_;
};
...// 省略storage3... storage9
{% endhighlight %}
storage之间用继承可以省掉很多代码。而且会带来其他的方便。
 
###2） 占位符
但是，占位符怎么办呢？我们都知道在boost::bind里，用_1, _2 ...等几个占位符来表示第几个参数等待后续传入。

这些占位符(_1, _2 ...)到底是什么呢？ 在boost/bind/placeholders.hpp里面有这样的定义。
{% highlight cpp %}
boost::arg<1> _1;
boost::arg<2> _2;
//...
{% endhighlight %}
而boost::arg是一个简单的模板类型:
{% highlight cpp %}
template <int I> struct arg
{
    arg() {}
    ...
};
{% endhighlight %}
这样，根据模板参数 <int I> 的不同，就有了9个不同类型的变量。
 
###3） 加入占位符之后的存储
现在，有了占位符。本质上，它也是一个变量，对应着一个模板类型。

为了区分占位符和普通数据类型，需要对storage做特殊处理。由于storage是一个模板类，因此，特殊处理的方案就是偏特化:

{% highlight cpp %}
template <int I> struct storage1< boost::arg<I> >
{
...// ???
}
{% endhighlight %}

这样每个storage都对应这一个偏特化版本。问题是省略的地方(...)应该填入什么样的代码？

回到偏特化的目的上，偏特化的目的是要让之后函数调用的时候，能准确的根据参数类型(是占位符，还是实参)，选择函数调用的参数。

既然这样，我们需要加入某种机制，使得两种类型，普通类型和特化类型分别返回不同的结果。

C++ trait是一种选择，比如我们可以在普通版本和偏特化版本中加入一个arg_type变量。分别指向不同的类型(T 和arg<I>)。

但boost::bind却不是这样做的。 其成员变量A1 a_。就可以作为一种标志位。

因此这个偏特化版本的完整定义是：
 
{% highlight cpp %} 
template <int I> struct storage1< boost::arg<I> >
{
    explicit storage1( boost::arg<I> ) {}
    static boost::arg<I> a1_() { return boost::arg<I>(); }
    ...// 省略的部分代码与文档的逻辑无关
}
{% endhighlight %}

这样我们就可以定义某种结构来取参数了。

比如在调用的时候，我们可以把传入的参数构造成某种类型，然后在参数调用的过程中，根据返回类型，决定使用a1_还是传入的参数。

注意到，如果该类型是个占位符，boost::bind将a_ 从变量变成了静态函数！达到了节省空间的目的。
 
现在我们有了一个参数的storage，不妨把它命名为storage1。而boost::bind支持九个参数，storage从一到九的过程是怎样的呢？

继承和组合都能达到我们的目的。boost::bind采用的是继承，storage2继承storage1, storage3继承storage2... 以此类推。如下图示：

![](/assets/image/2015-01/boost-bind-3.png)

采用继承的好处是不仅可以省下许多代码，而且对空间的节省也有一定的好处。 
 
boost::bind将构造boost::bind时，和调用时传入参数"构造的类型"统一起来。

在构造时为存储参数和占位符构造一个storage；在取参数的时候需要传入真正的参数用以取代占位符，把这些参数再构造另一个storage。

当调用触发时，根据前一个storage(构造时)中参数的类型做不同处理：如果是占位符，则从后者中取参数；否则，从前者中取。

这样做，也是行得通的。但是boost::bind采取的方案是一种更优雅的办法。
 
###4） boost::bind 采取的方案
既然有为占位符而生的boost::arg<int>， 为什么不另外创建一个为普通类型而生的模板类呢？

这个类是value 类。

{% highlight cpp %}
template<class T> class value
{
public:
    value(T const & t): t_(t) {}
    T & get() { return t_; }
    T const & get() const { return t_; }
    bool operator==(value const & rhs) const
    {
        return t_ == rhs.t_;
    }
private:
    T t_;
};
{% endhighlight %}

这样storage模板的类型可能是value， 也可能是boost::arg。

取参数的时候就比较简单了。

假设s_cont 表示构造boost::bind时创建的storage类，而s_call表示调用创建的类。为这个storage加上operator[] 操作符，以storage 1为例:

{% highlight cpp %}
template <class A1>
struct storage
{
A1 operator[] (boost::arg<1>) const { return a1_; }
template<class T> T & operator[] (_bi::value<T> & v) const { return v.get(); }
...
};
{% endhighlight %}

在参数调用的时候，我们只需要

{% highlight cpp %}
s_call[s_cont::a1]
{% endhighlight %}

就能拿到对应的类型。
 
###5)   参数获取
由于取参数的操作与存储无关，可以将这两个职责分离开来。定义listN类，分别接口继承自stroageN，并将operator[]操作符置于其中。

当然，listN不仅仅只有这一个职责，它还有另一个职责，这也是为什么开头部分的代码在调用的时候( f(1, 2) )虽然传入的参数个数不匹配，却能正确的调用的原因。

现在我们可以改写boost::bind，它应该有类似如下的定义:

{% highlight cpp %}
template <class R, class F, class L>
class bind
{
public:
   bind_t(F f, L const & l): f_(f), l_(l) {}
   ...
private:
   F f_;
   L l_;
};
{% endhighlight %}

从引言开始的例子可以看出， boost::bind(&f, _1) 这个明明只有一个参数的Functor，能正确的转换成两个参数的。实际上，只要保证第一个参数（已有）类型匹配，它根本不关心后面跟了多少个参数。

因此，可以猜测，boost::bind 至少实现了九种重载的operator。

{% highlight cpp %}
  template<class A1> result_type operator()(A1 & a1) const
  {
      list1<A1 &> a(a1);
      ...//
  }
  template<class A1, class A2> result_type operator()(A1 & a1, A2 & a2)
  {
      list2<A1 &, A2 &> a(a1, a2);
      ...//
  }
....
{% endhighlight %}

省略的部分是什么样的代码？

现在，boost::bind 参数存储的部分真正的类间关系图如下：
 
![](/assets/image/2015-01/boost-bind-2.png) 
 
###6） 正确的调用
从引言部分的例子可以看到，尽管我们传入 f(10, 20) 以两个参数的方式去调用真正的函数，却没有出错。

这说明，boost::bind能根据函数真正需要参数的类型，而不是调用时传入的类型去匹配正确的调用。

而我们在使用boost::bind的时候，在构造bind的结构时，传入了正确的参数类型！

包含这些正确参数类型的参数保存在内部结构listN 中。因此，只要在listN中实现对应的operator 就能保证正确的调用:

{% highlight cpp %}
template<class A1, class A2> result_type operator()(A1 & a1, A2 & a2)
{
    list2<A1 &, A2 &> a(a1, a2);
    BOOST_BIND_RETURN l_(type<result_type>(), f_, a, 0)；
}
{% endhighlight %}
把注意力放到最后一行代码，可以看到。构造boost::bind时候传入的前期绑定的参数信息保存在l_ 中，函数指针保存在f_ 中。真正调用时候传入的参数信息保存在a 中。

有了这些信息，我们就能保证调用的正确性了。

{% highlight cpp %}
template<class F, class A> void operator()(type<void>, F & f, A & a, int)
{
    unwrapper<F>::unwrap(f, 0)(a[base_type::a1_], a[base_type::a2_]);
}
...
{% endhighlight %}
 
###7)  统一入口
当然，可以看到，我们使用boost::bind的时候，传入的模板参数并不是template< class R, class F, class L> 这三个。因此boost::bind需要定义一系列的接口（对应着九个不同类型的参数个数），并最终返回正确的bind_t类型。

下面是两个参数的一个例子：
{% highlight cpp %}
 template<class R, class B1, class B2, class A1, class A2>
    _bi::bind_t<R, BOOST_BIND_ST R (BOOST_BIND_CC *) (B1, B2), typename _bi::list_av_2<A1, A2>::type>
    BOOST_BIND(BOOST_BIND_ST R (BOOST_BIND_CC *f) (B1, B2), A1 a1, A2 a2)
{
    typedef BOOST_BIND_ST R (BOOST_BIND_CC *F) (B1, B2);
    typedef typename _bi::list_av_2<A1, A2>::type list_type;
    return _bi::bind_t<R, F, list_type> (f, list_type(a1, a2));
}
{% endhighlight %}

它根据调用时候我们使用的参数推导出，函数返回值类型R, Functor类型 R (*f) (B1, B2)，传入的参数类型 A1, A2。

##4. 总结
有了前面的这些分析，一些现象就很容易解释了。比如为什么支持 _2, _1 交换顺序等。

至于函数指针保存和调用的部分，代码比较简单；boost::bind为支持成员函数做了特殊处理，因为成员函数的调用实际上是instancePtr->foo(..)，因此需要记录instancePtr信息。这部分代码在bind_mf_cc.hpp 和 mem_fn_template.hpp。

代码比较简单易读，这里就不多做分析了。

最后附上一张内部结构的图，作为本文的结束：

![](/assets/image/2015-01/boost-bind-1.png)
