---
layout: post
title: c++ 异常处理（上）
description: c++ 异常处理（上）
category: C++
tags: C++ Exception-handling
refer_author: miliao
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}


c++ 异常处理（上）
==============


异常(exception)是c++中新增的一个特性，它提供了一种新的方式来结构化地处理错误，使得程序可以很方便地把异常处理与出错的程序分离，而且在使用上，它语法相当地简洁，以至于会让人错觉觉得它底层的实现也应该很简单，但事实上并不是这样。恰恰因为它语法上的简单没有规定过多细节，从而留给了编译器足够的空间来自己发挥，因此在不同操作系统，不同编译器下，它的实现是有很大不同的。[这篇](http://www.microsoft.com/msj/0197/exception/exception.aspx)文章介绍了windows和visual
c++是怎样基于SEH来实现c++上的异常处理的，讲得很详细，虽然已经写了很久，但原理性的东西到现在也没过时，有兴趣可以去细读一下。

至于linux下gcc是怎样做的，网上充斥着各种文档，很多但也比较杂，我这儿就简单把我这几天看到的，想到的，理解了的，不理解的，做个简单的总结，欢迎指正。

### **异常抛出后，发生了什么事情？**

根据c++的标准，异常抛出后如果在当前函数内没有被捕捉(catch)，它就要沿着函数的调用链继续往上抛，直到走完整个调用链，或者在某个函数
中找到相应的catch。如果走完调用链都没有找到相应的catch，那么std::terminate()就会被调用，这个函数默认是把程序
abort，而如果最后找到了相应的catch，就会进入该catch代码块，执行相应的操作。

程序中的catch那部分代码有一个专门的名字叫作：**Landing pad**。

从抛异常开始到执行landing pad里的代码这中间的整个过程叫作stack
unwind，这个过程包含了两个阶段：

1）从抛异常的函数开始，一帧一帧地在每个调用函数里查找landing pad。

2）如果没有找到landing pad则把程序abort，否则，则记下landing
pad的位置，再重新回到抛异常的函数那里开始，一帧一帧地清理调用链上各个函数内部的局部变量，直到landing
pad所在的函数为止。

简而言之，正常情况下，stack
unwind所要做的事情就是从抛出异常的函数开始，找到catch所在的函数，然后从头开始清理调用链上的已经创建了的局部变量。

    void func1()
    {
      cs a; // stack unwind时被析构。
      throw 3;
    }

    void func2()
    {
      cs b;
      func1();
    }

    void func3()
    {
      cs c;
      try 
      {
        func2();
      }
      catch (int)
      {
        //进入这里之前， func1, func2已经被unwind.
      }
    }


可以看出，unwind的过程可以简单看成是函数调用的逆过程，这个过程在实现上由一个专门的stack
unwind库来进行，在intel平台上，它属于Itanium
ABI接口中的一部分，且与具体的语言无关，由系统提供实现，任何上层语言都可以在这个接口的基础上实现各自的异常处理，GCC就基于这个接口来实现c++的异常处理。

### **Itanium C++ ABI**

Itanium
ABI定义了一系列函数及相应的数据结构来建立整个异常处理的流程及框架，主要的函数包括以下这些：


    _Unwind_RaiseException,
    _Unwind_Resume,
    _Unwind_DeleteException,
    _Unwind_GetGR,
    _Unwind_SetGR,
    _Unwind_GetIP,
    _Unwind_SetIP,
    _Unwind_GetRegionStart,
    _Unwind_GetLanguageSpecificData,
    _Unwind_ForcedUnwind


其中的\_Unwind\_RaiseException函数就是用于进行stack
unwind的，它在用户执行throw时被调用，然后从当前函数开始，对调用链上每个函数幀都调用一个叫作personality
routine的函数（\_\_gxx\_personality\_v0)，该函数由上层的语言定义及提供实现，\_Unwind\_RaiseException
会在内部把当前函数栈的调用现场重建，然后传给personality
routine，personality routine则主要负责做两件事情：

1）检查当前函数是否含有相应catch可以处理上面抛出的异常。

2）清掉调用栈上的局部变量。

显然，我们可以发现personality routine所做的这两件事情和前面所说的stack
unwind所要经历的两个阶段一一对应起来了，因此也可以说，stack
unwind主要就是由personality routine来完成，它相当于一个callback。


    _Unwind_Reason_Code (*__personality_routine)
            (int version,
             _Unwind_Action actions,
             uint64 exceptionClass,
             struct _Unwind_Exception *exceptionObject,
             struct _Unwind_Context *context);


注意上面的第二个参数，它是用来告诉personality routine，当前是处于stack
unwind的哪个阶段的，其它的参数则主要用来传递与异常相关的信息及当前函数的上下文。

前面一直说stack
unwind包含两个阶段，具体到调用链上的函数来说，就是每个函数在unwind的过程中都会被personality
routine遍历两次。

下面的伪代码展示了\_Unwind\_RaiseException内部的大概实现，算是对前面的一个总结：


    _Unwind_RaiseException(exception)
    {
        bool found = false;
        while (1)
         {
             // 建立上个函数的上下文
             context = build_context();
             if (!context) break;
             found = personality_routine(exception, context, SEARCH);
             if (found or reach the end) break;
         }

        while (found)
        {
            context = build_context();
            if (!context) break;
            personality_routine(exception, context, UNWIND);
            if (reach_catch_function) break;
        }
    }

 

 ABI中的函数使用到了两个自定义的数据结构，用于传递一些内部的信息。


    struct _Unwind_Context;

    struct _Unwind_Exception {
      uint64     exception_class;
      _Unwind_Exception_Cleanup_Fn exception_cleanup;
      uint64     private_1;
      uint64     private_2;
    };



根据接口的介绍，\_Unwind\_Context是一个对调用者透明的结构，用于表示程序运行时的上下文，主要就是一些寄存器的值，函数返回地址等，它由接口实现者来定义及创建，但我没在接口中找到它的定义，只在gcc的源码里找到了一份它的[定义](http://stuff.mit.edu/afs/sipb/project/gcc-3.2/share/gcc-3.2.1/gcc/unwind-dw2.c)。


    struct _Unwind_Context
    {
      void *reg[DWARF_FRAME_REGISTERS+1];
      void *cfa;
      void *ra;
      void *lsda;
      struct dwarf_eh_bases bases;
      _Unwind_Word args_size;
    };


至于\_Unwind\_Exception，顾名思义，它在unwind库内用于表示一个异常。

### **C++ ABI.**

基于前面介绍的Itanium
ABI，编译器层面也定义了一系列的ABI来与之交互。当我们在代码中写下"throw
xxx"时，编译器会分配一个数据结构来表示该异常，该异常有一个头部，定义如下：

    struct __cxa_exception { 
      std::type_info *    exceptionType;
      void (*exceptionDestructor) (void *); 
      unexpected_handler    unexpectedHandler;
      terminate_handler    terminateHandler;
      __cxa_exception *    nextException;

      int     handlerCount;
      int     handlerSwitchValue;
      const char *     actionRecord;
      const char *     languageSpecificData;
      void *     catchTemp;
      void *     adjustedPtr;

      _Unwind_Exception    unwindHeader;
    };


注意其中最后一个变量：\_Unwind\_Exception
unwindHeader，就是前面Itanium接口里提到的接口内部用的结构体。当用户throw一个异常时，编译器会帮我们调用相应的函数分配出如下一个结构：

![](/assets/image/2014-05/2014-05-13-C-Plus-plus-Exception-Handle_files/exception_1.png)


其中\_cxa\_exception就是头部，exception\_obj则是"throw
xxx"中的xxx，这两部分在内存中是连续的。
异常对象由函数\_\_cxa\_allocate\_exception()
进行创建，最后由\_\_cxa\_free\_exception() 进行销毁。

当我们在程序里执行了抛出异常后，编译器为我们做了如下的事情：

1）调用\_\_cxa\_allocate\_exception函数，分配一个异常对象。

2）调用\_\_cxa\_throw函数，这个函数会将异常对象做一些初始化。

3）\_\_cxa\_throw() 调用Itanium ABI里的\_Unwind\_RaiseException()
从而开始unwind。

4）\_Unwind\_RaiseException()对调用链上的函数进行unwind时，调用personality
routine。

5）如果该异常如能被处理(有相应的catch)，则personality
routine会依次对调用链上的函数进行清理。

6）\_Unwind\_RaiseException() 将控制权转到相应的catch代码。

​7) unwind完成，用户代码继续执行。

从c++的角度看，一个完整的异常处理流程就完成了，当然，其中省略了很多的细
节，其中最让人觉得神秘的也许就是personality
routine了，它是怎么知道当前Unwind的函数是否有相应的catch语句呢？又是怎么知道该怎样去清理这个函数内的局部变量呢？具体实现这儿先
不细说，只需要大概明白，其实它也不知道，只有编译器知道，因此在编译代码的时候编译器会建立建立一些表项来保存相应的信息，使得personality
routine可以在运行时通过这些事先建立起来的信息进行相应的查询。

### **Unwind的过程**

 unwind的过程是从\_\_cxa\_throw()里开始的，请看如下源码：

    extern "C" void
    __cxxabiv1::__cxa_throw (void *obj, std::type_info *tinfo,
    void (_GLIBCXX_CDTOR_CALLABI *dest) (void *))
    {
       PROBE2 (throw, obj, tinfo);

       // Definitely a primary.
       __cxa_refcounted_exception *header = __get_refcounted_exception_header_from_obj (obj);
       header->referenceCount = 1;
       header->exc.exceptionType = tinfo;
       header->exc.exceptionDestructor = dest;
       header->exc.unexpectedHandler = std::get_unexpected ();
       header->exc.terminateHandler = std::get_terminate ();
       __GXX_INIT_PRIMARY_EXCEPTION_CLASS(header->exc.unwindHeader.exception_class);
       header->exc.unwindHeader.exception_cleanup = __gxx_exception_cleanup;

       #ifdef _GLIBCXX_SJLJ_EXCEPTIONS
       _Unwind_SjLj_RaiseException (&header->exc.unwindHeader);
       #else
       _Unwind_RaiseException (&header->exc.unwindHeader);
       #endif

       // Some sort of unwinding error. Note that terminate is a handler.
       __cxa_begin_catch (&header->exc.unwindHeader);
       std::terminate ();
    }


我们可以看到\_\_cxa\_throw最终调用了\_Unwind\_RaiseExceptioni()，stack
unwind就此开始，如前面所说，unwind分为两个阶段，分别进行搜索catch及清理调用栈，具体的实现可参看如下代码：


    /* Raise an exception, passing along the given exception object.  */

    _Unwind_Reason_Code
    _Unwind_RaiseException(struct _Unwind_Exception *exc)
    {
      struct _Unwind_Context this_context, cur_context;
      _Unwind_Reason_Code code;

      uw_init_context (&this_context);
      cur_context = this_context;

      /* Phase 1: Search.  Unwind the stack, calling the personality routine
         with the _UA_SEARCH_PHASE flag set.  Do not modify the stack yet.  */
      while (1)
        {
          _Unwind_FrameState fs;

          code = uw_frame_state_for (&cur_context, &fs);

          if (code == _URC_END_OF_STACK)
        /* Hit end of stack with no handler found.  */
        return _URC_END_OF_STACK;

          if (code != _URC_NO_REASON)
        /* Some error encountered.  Ususally the unwinder doesn't
           diagnose these and merely crashes.  */
        return _URC_FATAL_PHASE1_ERROR;

          /* Unwind successful.  Run the personality routine, if any.  */
          if (fs.personality)
        {
          code = (*fs.personality) (1, _UA_SEARCH_PHASE, exc->exception_class,
                        exc, &cur_context);
          if (code == _URC_HANDLER_FOUND)
            break;
          else if (code != _URC_CONTINUE_UNWIND)
            return _URC_FATAL_PHASE1_ERROR;
        }

          uw_update_context (&cur_context, &fs);
        }

      /* Indicate to _Unwind_Resume and associated subroutines that this
         is not a forced unwind.  Further, note where we found a handler.  */
      exc->private_1 = 0;
      exc->private_2 = uw_identify_context (&cur_context);

      cur_context = this_context;
      code = _Unwind_RaiseException_Phase2 (exc, &cur_context);
      if (code != _URC_INSTALL_CONTEXT)
        return code;

      uw_install_context (&this_context, &cur_context);
    }


    static _Unwind_Reason_Code
    _Unwind_RaiseException_Phase2(struct _Unwind_Exception *exc,
                      struct _Unwind_Context *context)
    {
      _Unwind_Reason_Code code;

      while (1)
        {
          _Unwind_FrameState fs;
          int match_handler;

          code = uw_frame_state_for (context, &fs);

          /* Identify when we've reached the designated handler context.  */
          match_handler = (uw_identify_context (context) == exc->private_2
                   ? _UA_HANDLER_FRAME : 0);

          if (code != _URC_NO_REASON)
        /* Some error encountered.  Usually the unwinder doesn't
           diagnose these and merely crashes.  */
          return _URC_FATAL_PHASE2_ERROR;

          /* Unwind successful.  Run the personality routine, if any.  */
          if (fs.personality)
          {
            code = (*fs.personality) (1, _UA_CLEANUP_PHASE | match_handler,
                        exc->exception_class, exc, context);
            if (code == _URC_INSTALL_CONTEXT)
              break;
            if (code != _URC_CONTINUE_UNWIND) 
              return _URC_FATAL_PHASE2_ERROR;
          }

          /* Don't let us unwind past the handler context.  */
          if (match_handler)
             abort ();

          uw_update_context (context, &fs);
        }

      return code;
    }


如上两个函数分别对应了unwind过程中的这两个阶段，注意其中的：

    uw_init_context()
    uw_frame_state_for()
    uw_update_context()

这几个函数主要是用来重建函数调用现场的，它们的实现涉及到一大堆的细节，这儿卖个关子先不细说，大概原理就是，对于调用链上的函数来说，它们的很
大一部分上下文是可以从堆栈上恢复回来的,如ebp,esp,返回地址等。编译器为了让unwinder可以从栈上获取这些信息，它在编译代码的时候，建
立了很多表项用于记录每个可以抛异常的函数的相关信息，这些信息在重建上下文时能够指导程序怎么去搜索栈上的东西。

### **做点有意思的事情**

说了一大堆，下面写个测试的程序简单回顾一下前面所说的关于异常处理的大概流程：


    #include <iostream>
    using namespace std;

    void test_func3()
    {
        throw 3;

        cout << "test func3" << endl;
    }

    void test_func2()
    {
        cout << "test func2" << endl;
        try
        {
            test_func3();
        }
        catch (int)
        {
            cout << "catch 2" << endl;
        }
    }

    void test_func1()
    {
        cout << "test func1" << endl;
        try
        {
            test_func2();
        }
        catch (...)
        {
            cout << "catch 1" << endl;
        }
    }

    int main()
    {
        test_func1();
        return 0;
    }


上面的程序运行起来后，我们可以在\_\_gxx\_personality\_v0 里下一个断点。


    Breakpoint 2, 0x00dd0a46 in __gxx_personality_v0 () from /usr/lib/libstdc++.so.6
    (gdb) bt
    #0  0x00dd0a46 in __gxx_personality_v0 () from /usr/lib/libstdc++.so.6
    #1  0x00d2af2c in _Unwind_RaiseException () from /lib/libgcc_s.so.1
    #2  0x00dd10e2 in __cxa_throw () from /usr/lib/libstdc++.so.6
    #3  0x08048979 in test_func3 () at exc.cc:6
    #4  0x080489ac in test_func2 () at exc.cc:16
    #5  0x08048a52 in test_func1 () at exc.cc:29
    #6  0x08048ad1 in main () at exc.cc:39
    (gdb)


 从这个调用栈可以看出，异常抛出后，我们的程序都做了些什么。如果你觉得好玩，你甚至可以尝试去hook掉其中某些函数，从而改变异常处理的行为，这种hack的技巧在某些时候是很有用的，比如说我现在用到的一个场景：

我们使用了一个第三库，这个库里有一个消息循环，它是放在一个try/catch里面的。

    void wxEntry()
    {
        try
        {
           call_user_func();
        }
        catch(...)
        {
           unhandled_exception();
        }
    }


call\_user\_func()会调用一系列的函数，其中涉及我们自己写的代码，在某些时候我们的代码抛异常了，而且我们没有捕捉住，因此
wxEntry最终会catch住这些异常，然后调用unhandled\_exception(),
这个函数默认会调用一些清理函数，然后把程序abort，而在调用清理函数的时候，由于我们的代码已经行为不正常了，在种情况下去清理通常又会引出很多其
它的奇奇怪怪的错误，最后就算得到了coredump也很难判断出我们的程序哪里出了问题。所以我们希望当我们的代码抛出异常且没有被我们自己处理而在
wxEntry()中被捕捉了的话，我们可以把抛异常的地方的调用栈给打出来。

一开始我们尝试把\_\_cxa\_throw给hook了，也就是每当有人一抛异常，我们就把当时的调用栈给打出来，这个方案可以解决问题，但是问题很明显，它影响了所有抛异常的代码的执行效率，毕竟收集调用栈相对来说是比较费时的。

其实我们并没必要对每个throw都去处理，问题的关键就在于我们能不能识别出我们所想要处理的异常。

在这个案例中，我们恰恰可以，因为所有没被处理的异常，最终都会统一上抛到wxEntry中，那么我们只要hook一下personality
routine，看看当前unwind的是不是wxEntry不就可以了吗！

    #include <execinfo.h>
    #include <dlfcn.h>
    #include <cxxabi.h>
    #include <unwind.h>#include <iostream>using namespace std;

    void test_func1();
    static personality_func gs_gcc_pf = NULL;

    static void hook_personality_func()
    {
        gs_gcc_pf = (personality_func)dlsym(RTLD_NEXT, "__gxx_personality_v0");
    }

    static int print_call_stack()
    {
       //to do.
    }

    extern "C" _Unwind_Reason_Code
    __gxx_personality_v0 (int version,
                  _Unwind_Action actions,
                  _Unwind_Exception_Class exception_class,
                  struct _Unwind_Exception *ue_header,
                  struct _Unwind_Context *context)
    {
        _Unwind_Reason_Code code = gs_gcc_pf(version, actions, exception_class, ue_header, context);

        if (_URC_HANDLER_FOUND == code)
        {
            //找到了catch所有的函数

            //当前函数内的指令的地址
            void* cur_ip = (void*)(_Unwind_GetIP(context));
          
            Dl_info info;
            if (dladdr(cur_ip, &info))
            {
                 if (info.dli_saddr == &test_func1)
                 {
                     // 当前函数是目标函数
                     print_call_stack();
                 }
            }
        }

        return code;
    }


    void test_func3()
    {
        char* p = new char[2222222222222];
        cout << "test func3" << endl;
    }

    void test_func2()
    {
        cout << "test func2" << endl;
        try
        {
            test_func3();
        }
        catch (int)
        {
            cout << "catch 2" << endl;
        }
    }

    void test_func1()
    {
        cout << "test func1" << endl;
        try
        {
            test_func2();
        }
        catch (...)
        {
            cout << "catch 1" << endl;
        }
    }


    int main()
    {
        hook_personality_func();

        test_func1();

        return 0;
    }


上面的代码中，personality routine
返回\_URC\_HANDLER\_FOUND则意味着当前函数帧里找到相应的landing
pad，然后我们就尝试判断一下，该函数是否就是我们的目标函数，如果是则马上进行相应的处理。这个做法显然比hook
\_\_cxa\_throw要好一些，毕竟只针对一个函数做了处理，当然，与原生的异常处理相比，这里还是有一定的效率损失的，就看怎么取舍了，追求方便debug是必然要付出些代价的。