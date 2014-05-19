---
layout: post
title: 说说尾递归
description: 说说尾递归
category: C++
tags: C++ tail-recursion 尾递归
refer_author: miliao
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

说说尾递归
=================================

微博上看到有人在讨论尾递归，想起以前曾看过@老赵写的一篇相关的博客，介绍的比较详细了，相信很多人都看过，我也在下面留了言，但挑了个刺，表示文章在关键点上一带而过了，老赵自然是懂的，但看的人如果不深入思考，未必真正的明白。

下面我说说我的理解。

**什么是尾递归**
----------------

什么是尾递归呢?(tail recursion),
顾名思议，就是一种“不一样的”递归，说到它的不一样，就得先说说一般的递归。

对于一般的递归，比如下面的求阶乘，教科书上会告诉我们，如果这个函数调用的深度太深，很容易会有爆栈的危险。


    // 先不考虑溢出问题
    int func(int n)
    {
        if (n <= 1) return 1;

        return (n * func(n-1));
    }


原因很多人的都知道，让我们先回顾一下函数调用的大概过程：

1）调用开始前，调用方（或函数本身）会往栈上压相关的数据，参数，返回地址，局部变量等。

2）执行函数。

3）清理栈上相关的数据，返回。

因此，在函数A执行的时候，如果在第二步中，它又调用了另一个函数B，B又调用C....
栈就会不断地增长不断地装入数据，当这个调用链很深的时候，栈很容易就满
了。

这就是一般递归函数所容易面临的大问题。而尾递归在某些语言的实现上，能避免上述所说的问题，注意是某些语言上，尾递归本身并不能消除函数调用栈过长的问题。  

那什么是尾递归呢？

在上面写的一般递归函数func()中，我们可以看到，func(n)是依赖于func(n-1)的，func(n)只有在得到func(n-1)的结果之后，才能计算它自己的返回值，因此理论上，在func(n-1)返回之前，func(n)，不能结束返回。

因此func(n)就必须保留它在栈上的数据，直到func(n-1)先返回。

而尾递归的实现则可以在编译器的帮助下，消除这个限制：

    // 先不考虑溢出

    int tail_func(int n, int res)
    {
         if (n <= 1) return res;

         return tail_func(n - 1, n * res);
    }

    // 像下面这样调用

    tail_func(10000000000, 1);


从上可以看到尾递归把返回结果放到了调用的参数里。

这个细小的变化导致，tail\_func(n,
res)不必像以前一样，非要等到拿到了tail\_func(n-1,
n\*res)的返回值，才能计算它自己的返回结果 -- 它完全就等于tail\_func(n-1,
n\*res)的返回值。

因此理论上：tail\_func(n)在调用tail\_func(n-1)前，完全就可以先销毁自己放在栈上的东西。
这就是为什么尾递归如果在得到编译器的帮助下，是完全可以避免爆栈的原因：每一个函数在调用下一个函数之前，都能做到先把当前自己占用的栈给先释放了，尾递归的调用链上可以做到只有一个函数在使用栈。

因此可以无限地调用！

**尾递归的调用栈优化特性**

相信读者都注意到了，我一直在强调，尾递归的实现依赖于编译器的帮助(或者说语言的规定)。

为什么这样说呢？先看下面的程序：

    #include <stdio.h>
    
    int tail_func(int n, int res)
    {
         if (n <= 1) return res;
    
         return tail_func(n - 1, n * res);
    }
    
    
    int main()
    {
        int dummy[1024*1024]; // 尽可能占用栈。
        
        tail_func(2048*2048, 1);
        
        return 1;
    }

上面这个程序在开了编译优化和没开编译优化的情况下编出来的结果是不一样的。

如果不优化，直接 gcc -o tr func\_tail.c
编译然后运行的话，程序会爆栈崩溃。

但如果开优化的话，gcc -o tr -O2 func\_tail.c

最后就能正常运行。

这里面的原因就在于，尾递归的写法只是具备了使当前函数在调用下一个函数前把当前占有的栈销毁，但是会不会真的这样做，是要具体看编译器是否最终这样做。

如果在语言层面上，没有规定要优化这种尾调用，那编译器就可以有自己的选择来做不同的实现，在这种情况下，尾递归就不一定能解决一般递归的问题。

我们可以先看看上面的例子在开优化与没开优化的情况下，编译出来的汇编代码有什么不同：

没开优化编译出来的汇编tail\_func：


    .LFB3:
            pushq   %rbp
    .LCFI3:
            movq    %rsp, %rbp
    .LCFI4:
            subq    $16, %rsp
    .LCFI5:
            movl    %edi, -4(%rbp)
            movl    %esi, -8(%rbp)
            cmpl    $1, -4(%rbp)
            jg      .L4
            movl    -8(%rbp), %eax
            movl    %eax, -12(%rbp)
            jmp     .L3
    .L4:
            movl    -8(%rbp), %eax
            movl    %eax, %esi
            imull   -4(%rbp), %esi
            movl    -4(%rbp), %edi
            decl    %edi
            call    tail_func
            movl    %eax, -12(%rbp)
    .L3:
            movl    -12(%rbp), %eax
            leave
            ret


注意上面的第21行，call
指令就是直接进行了函数调用，它会先压栈，然后再jmp去tail\_func，而当前的栈还在用！
就是说，尾递归的作用没有发挥。

再看看开了优化得到的汇编：


    tail_func:
    .LFB13:
            cmpl    $1, %edi
            jle     .L8
            .p2align 4,,7
    .L9:
            imull   %edi, %esi
            decl    %edi
            cmpl    $1, %edi
            jg      .L9
    .L8:
            movl    %esi, %eax
            ret


注意第7，第10行，尤其是第10行！

tail\_func()里面没有函数调用！它只是把当前函数的第二个参数改了一下，直接就又跳到函数开始的地方。
这里的实现其实就是：下一个函数调用继续延用了当前函数的栈！
这就是尾递归所能带来的效果:
控制栈的增长，且减少压栈，程序运行的效率也更高！

上面所写的是c的实现，正如前面所说的，这并不是所有语言都摆支持，有些语言，比如说python，
尾递归的写法在python上就没有任何作用，该爆的时候还是会爆。

    def func(n, res):

        if (n <= 1):
            return res

        return func(n-1, n*res)

    if __name__ =='__main__':
        print func(4096, 1)

不仅仅是python，据说C\#也不支持，我在网上搜到了这个链接：[https://connect.microsoft.com/VisualStudio/feedback/details/166013/c-compiler-should-optimize-tail-calls](https://connect.microsoft.com/VisualStudio/feedback/details/166013/c-compiler-should-optimize-tail-calls)

微软的人在上面回答说，实现这个优化有些问题需要处理，并不是想像中那么容易，因此暂时没有实现，但是这个回答是在2007年的时候了，到现在岁月变迁，不知支持了没？我看老赵写的尾递归博客是在09年，用c\#作的例子，估计现在c\#是支持这个优化的了(待考).

### ** 尾调用**

前面的讨论一直都集中在尾递归上，这其实有些狭隘，尾递归的优化属于尾调用优化这个大范畴，所谓尾调用，形式它与尾递归很像，都是一个函数内最后一个动作是调用下一个函数，不同的只是调用的是谁，显然尾递归只是尾调用的一个特例。

 

    int func1(int a)
    {
       static int b = 3;
       return a + b;
    }

    int func2(int c)
    {
        static int b = 2;

        return func1(c+b);
    }

 

上面例子中，func2在调用func1之前显然也是可以完全丢掉自己占有的栈空间的，原因与尾递归一样，因此理论上也是可以进行优化的，而事实上
这种优化也一直是程序编译优化里的一个常见选项，甚至很多的语言在标准里就直接要求要对尾调用进行优化，原因很明显，尾调用在程序里是经常出现的，优化它
不仅能减少栈空间使用，通常也能给程序运行效率带来比较大的提升。
