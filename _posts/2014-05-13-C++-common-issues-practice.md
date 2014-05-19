---
layout: post
title: C++问题汇总
description: C++问题汇总
category: C++
tags: C++
refer_author: bozhang
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

### C++ Common
1. 在C++里面，0, false, NULL 是等价的。 所以int\* p = 0; int\* p = NULL;
int p = false之间没有任何区别， 编译器不会抱怨。但是在一些特殊的情况下，
这种特性会为我们带来意想不到的错误。

        void func(const std::string& str, bool flag = true)
        {
          ....
        }
             
            void func_1()
            {
                // 这里编译没有任何问题，运行的时候直接Crash。
                // 因为编译器会把false当成一个空的地址， 然后尝试用这个地址来构造一个string, 导致出错
                func(false); // 相当于调用func(NULL, 1);
            }
    
2. 不要为虚函数设置默认参数。这是因为默认参数值是在编译期决定的，而虚函数的调用是在运行时动态决定的，这样就会出现问题。看下面例子
    
        class A
        {
        public:
            virtual void vfun(int i = 10)
            {
                cout<<"in A::fun i = "<<i<<endl;
            }
        } ;
         
        class B : public A
        {
        private:
            virtual void vfun(int i = 20)
            {
                cout<<"in B::fun i = "<<i<<endl;
            }
        };
         
        int main()
        {
            B *pb= new B;
            A *pa = pb;
            pb->vfun();// the output is "in B::fun i = 20"
            pa->vfun();// the output is "in B::fun i = 10"
            return 0;
        }

    对于这类问题,一般使用模板方法模式来解决。这样我们就不用担心参数的默认值了。

        class A
        {
        public:
            void vfun(int i = 10)
            {
                do_vfun(i);
            }
        private:
            virtual void do_vfun(int i)
            {
                cout << "in A::fun i = " << i << endl;
            }
         
        } ;
         
        class B : public A
        {
        private:
            virtual void do_vfun(int i)
            {
                cout << "in B::fun i = " << i << endl;
            }
        };

3. 匿名空间中定义的函数不要与全局的函数发生冲突。看下面例子

        void Foo() // 1
        {
        }
         
        namespace
        {
            void Foo() // 2
            {
            }
        }
         
        int main()
        {
            Foo();   // Ambiguous.
            ::Foo(); // Calls the Foo in the global namespace (Foo #1).
        }
    
    这时候如果我们想调用匿名空间里面的Foo函数应该怎么办呢？ 可以这样做
    
        void Foo() // 1
        {
        }
         
        namespace
        {
        namespace inner
        {
            void Foo() // 2
            {
            }
        }
        }
         
        int main()
        {
            Foo();        // OK now, Calls the Foo in the global namespace (Foo #1).
            ::Foo();      // Calls the Foo in the global namespace (Foo #1).
            inner::Foo(); // Call the Foo in the anonymous namespace (Foo #2)
        }
    
    也可以这样做：
    
        void Foo() // 1
        {
        }
         
        namespace
        {
            void Foo() // 2
            {
            }
        }
         
        namespace {       // re-open same anonymous namespace
            int do_main()
            {
                Foo();    // Calls local, anonymous namespace (Foo #2).
                ::Foo();  // Calls the Foo in the global namespace (Foo #1).
                return 0; // return not optional
            }
        }
         
        int main() 
        {
            return do_main();
        }

4. 代码理解， 你能看明白下面这段代码吗？
    
        template <int N>
        int (*return_array(const someclass& sc))[N]
        {
            return &sc.array;
        }
                 
        template <int N>
        int (&return_array(const someclass& sc))[N]
        {
            return sc.array;
        }        
            
    
    把他转换一下就容易理解了
    
        typedef int int_array[N];
         
        int_array* return_array(const someclass& sc)
        {
           return &sc.array;
        }
         
        int_array &return_array(const someclass& sc)
        {
            return sc.array;
        }

5. 引用做函数参数是不能给默认值， 因为不能给引用赋常量值。share\_ptr做参数时，不能像普通指针那样用NULL做默认值。比如：

        int GetStat(double xc0, double& xc1 = -1000, double& yc1 = -1000);// wrong, reference can't be assigned to one constant var
        int GetTask(int ref, shared_ptr<SomeClass> p = NULL); // wrong, should use shared_ptr<SomeClass> p = shared_ptr<SomeClass>();

6. 重说delete 与 delete \[\] 的区别  
   看一个简单的例子

        class A {
        public:
            int m_data[10];
        };
             
        A* pA = new A[20];
        delete pA; // we know here should use delete [] pA

    我们知道这里应该用delete \[\] pA;，但是delete
    pA;真的会导致内存泄漏吗？其实, 在这里不管是用delete
    还是用delete\[\]，那20个A对象的空间其实都会被free掉,
    只是如果用delete的话，它只会调用第一个A的析构函数，
    其他19个A对象的析构函数并没有被调用，
    因而可能导致内存泄露，特别是在A的析构函数需要额外释放资源时。

7. 在Java中有final关键字，它的作用是使某一个类不能被继承。  
    比如：
    
        public final class A {
        }
        // 编译错误！A是final类型，不可被继承！
        //!public class B extends A{
        //!}
    
    其实, 在C++11 里面，已经有了final关键字了。但是不用final关键字也可以实现这一功能。方法有很多  
     方法一：
    
        class A0 {
            friend class A;
        private:
            A0() { }
            ~A0() { }
        };
         
        class A : virtual private A0
        {
        };
         
        class B : public A   {   };
    
    方法二：
    
        class FinalClassBase
        {
        protected:
            FinalClassBase(const int) {}
            ~FinalClassBase() {}
        };
         
        #define final_class(c) class c : private virtual FinalClassBase
         
        final_class(Base)
        {};
         
        class Derived : public Base {}; // This won't compile.
    
8. 关于虚继承，不管继承体系有多长，被虚继承的类只会有一份，只会被构造一次，而且是最先被构造。  
     看下面的例子：
    
        #include <iostream>
         using namespace std;
         
         struct VBase
         {
              VBase(int val)
              {
                    cout << "base:" << "value is " << val << endl;
              }
         };
         struct Base1: public virtual VBase
         {
             Base1(int val)
              :VBase(val)
             {
                  cout << "base1" << endl;
             }
        };
        struct Base2: public virtual VBase
        {
            Base2(int val)
              :Base1(val),VBase(0)
            {
                 cout << "base2" << endl;
            }
        };
         
        struct Drived : public Base1, public Base2
        {
            Drived(int val):Base1(val), Base2(val), VBase(0) {}
        }
        int main()
        {
           Drived drive(99);
           return 0;
        }
        // the output is:
        // base:value is 0
        // base1
        // base2
    
    分析： 在这里，VBase 被Base1 和Base2虚继承,
    然后Base1和Base2又被Drived继承。在构造Drived的时候，会最先调用VBase的构造函数然后再调用Base1和Base2的构造函
    数，在调用Base1和Base2构造函数的时候，不会再重复调用VBase的构造函数了。
    
9. 关于前置声明，
    在工程里面，我们推荐使用前置声明，只有当我们需要知道这个类型的大小或者需要知道它的函数签名的时候，我们才需要获得它的定义。这里总结一下什么时候可以用前置声明。  
     假设有Class A 和 Class B,
    
    1. A继承至B, 需要包含B的定义  
    1. A有一个类型为B的成员变量， 需要包含B的定义  
    1. A有一个类型为B的指针成员变量, 可用前置声明  
    1. A有一个类型为B的引用成员变量, 可用前置声明。  
    1. A有stl 容器，容器内元素类型不管是B还是B\*， 比如std::vector, 可用前置声明 
    1. A有一个成员函数，它的返回值和参数类型是B， 可用前置声明  
    1. A在头文件中调用了B中的函数， 需要包含B的定义。  
    1. A 中有两个函数， 一个函数调用另一个函数
    
            B& DoFunc(B& b);
            B& Func(B& b) { return DoFunc(b);} // 可用前置声明，但是随便把其中一个B&换成C就得包含B定义
    
    1. 在不同命名空间的前置声明方式
    
            //a.h
            namespace SpaceB
            {
                class B;
            }
            namespace SpaceA
            {
                using SpaceB::B;
                Class A { ... };
            }
    
10. C++中的参数依赖查找规则：查找当前命名空间中是否有匹配的函数声明.
    如果函数中有类类型的参数,
    那么这个类参数所在的命名空间也会被考虑到函数查找的范围内.
    而且这两部分命名空间并没有先后顺序.  
     例如,example1 可编译通过，example2 不能编译通过
    
        // example 1
        namespace NS
        {
            class T { };
            void f(T);
        }
        NS::T parm;
        int main()
        {
            f(parm); // can pass compile
        }
     
        // example2
        namespace NS
        {
            class T { };
            void f(T);
        }
        void f( NS::T );
        int main()
        {
            NS::T parm;
            f(parm); // can not pass compile
        }
    
11. 一个将数值转换成字符的方法，供参考
        
        const char* convert(char buf[], int value)
        {
           static char digits[19] =
            { '9', '8', '7', '6', '5', '4', '3', '2', '1',
              '0', '1', '2', '3', '4', '5', '6', '7', '8', '9' };
           const char* zero = digits + 9;
         
           int i = value;
           char* p = buf;
           do {
              int lsd = i % 10;
              i /= 10;
              *p++ = zero[lsd];
           } while (i != 0);
         
         if (value < 0)
            *p++ = '-';
         *p = '\0';
          std::reverse(buf, p);
          return p;
        }

12. 小心char数组：  
    
    1. 最好将数组全部初始化为0(可使用memset)或者至少在数组的末尾加一个0,否则数组的内容不确定  
    1. 因为数组的最后一个元素必须是0， 所以给数组分配长度时应该加1， 例如
    
            wxFile file(filename, wxFile::read );
            if(!file.IsOpened())
            {
                errMsg = wxT("Failed to open file");
                return ap_buffer;
            }
            long len = file.Length();
            char* buffer = new char[len]; // wrong, length not enouth,we at least append "0' at the end
    
    1. char s\[LENGTH\]={0}效果与memset((void \*)s, 0,
    LENGTH)一样，全部初始化为0。但是要注意char
    s\[LENGTH\]={'a'};只初始化了s\[0\]='a'  
    1. char数组全部初始化为0后，数组名并不等于NULL
    
13. 怎样才能在不修改已有代码的情况下，测试或者访问一个类中的私有成员函数。一个可行的方法：
    
        class A{
            ...
        private:
            void func(); // for example we want to test this func
        };
        // fisrt we can add a head file assume UnitTest.h
        // @UnitTest.h
        #define private public
        #define class struct
        ...
         
        // @Test file
        // #include the head file "UnitTest.h" before Class A's head file
        void testFunc() {
          A a = new A()
          a.func()    // we can call private method directly
        }

 14. typeid是用来获取对象的类型信息的, 它的返回值跟编译器有关。
    
         class foo
         {
         public:
           void say_type_name()
           {
             std::cout << typeid(int).name() << std::endl
                       << typeid(char).name()<< std::endl
                       << typeid(double).name()<< std::endl
                       << typeid(string).name()<< std::endl
                       << typeid(this).name() << std::endl
           }
         };
          
         int main()
         {
           foo f;;
           f.say_type_name();
         }

        在VC9，输出:
    
            int
            char
            double
            class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> >
            foo*
    
        在GCC，输出:
        
            i
            c
            d
            Ss
            P3foo
    
        这是因为typeid的输出跟编译器有关。在GCC里面，它默认输出的是不可读的信息。我们可以用一个简单的命令把它还原成可读信息。
    
            $ c++filt i c d Ss P3foo
            int
            char
            double
            std::basic_string<char, std::char_traits<char>, std::allocator<char> >
            foo*
    
        如果你想在代码里面获取这些信息，可以这样做：
    
            #include <cxxabi.h>
            int status;
            char *realname = abi::__cxa_demangle(typeid(obj).name(), 0, 0, &status);
            std::cout << realname;
            free(realname);

    
15. 静态成员函数没有this指针，因此不能直接访问非静态的成员变量。除此之外，静态成员函数与一般的成员函数没有任何区别，它们都有相同的作用域访问权限。比如，它们都能访问私有的成员变量或函数。
    
        class test
        {
           public:
             void entry() { proc(this);}
           protected:
             static void proc(test*);
           private:
             int var;
         
        };
         
        void test::proc(test* pt)
        {
            if (pt) pt->var = 11; // It is OK here.
        }
    
16. 小心size\_t, 我们不能简单的将它与unsigned int
    等同。因为：虽然在32位机器上size\_t 和 unsigned int
    都是32位的。但是，在64位机器上，size\_t变成64位的可unsigned
    int却还是32位的,所以会有问题。
    
17. std::map operator(\<) 如何比较大小：成对成对的比较，
    从第一个元素开始, 先比较它的key,key 如果相等再比较value. value
    如果也相等,再去比较第二个元素的key 和 value.依此类推
    
18. fprintf 会比fwrite 和 fstream要慢很多倍
    
19. 小心 std::string.c\_str(), 当string里面包含"\\0"的时候,
    它可能会把string 截短,
    因为c\_str()会把string转化成C风格的字符串，C风格的字符串以"\\0"作为结尾符。比如，下面的代码就可能有潜在的问题
    
        std::string data;
        &one_function_read_file_content($file_path, data);
        &another_function_do_something(data.c_str());// if file $file_path contains "0", then the string will be truncated
    
20. 避免在for循环里面分配内存
    
21. 避免用const修饰的常量作为数组的size,
    而应该用宏或者enum.因为const修饰的常量值在编译期可能是无效的,而数组size在编译期必须确定。
    
22. C++中定义模板的模板, 例子
    
        template <template<class> class _Foo, class _Trait>
        void Test(_Foo<_Trait>& foo)
        {
            int a(0);
            _Foo<Trait2<a>> foo2;
        }
    
 23. \_\_cdecl, \_\_stdcall 以及\_\_thiscall的区别

        1. \_\_cdecl： 这是C/C++函数的默认调用规范，参数从右向左依次传递并压入堆栈，由调用函数负责堆栈的清退，这种方式可以传递个数可变的参数
        1. \_\_stdcall：这是Win API函数使用的规范，参数从右向左依次传递并压入堆栈，由被调用函数负责堆栈的清退，它生成的函数代码比__cdecl小，但是当参数有可变个数的参数时会转化为\_\_cdecl
        1. \_\_thiscall：这是C++非静态成员函数的默认调用规范，不能使用个数可变的参数。当调用非静态成员函数时，this指针直接保存在ECX寄存器而不是压入函数堆栈，其他方面与\_\_stdcall相同
    
    
24. 虚函数的访问权限问题

    1. 访问权限是在编译期决定的, 而多态或者说虚函数的调用是在运行时决定的  
    1. 虚函数的访问权限是在基类中声明虚函数是决定的，跟派生类中重写的虚函数的访问权限无关。  
           例如：
        
            class B {
            public:
                virtual int f();
            };
             
            class D : public B {
            private:
                int f();
            };
             
            void f()
            {
                D d;
                B* pb = &d;
                D* pd = &d;
             
                pb->f(); // OK:B::f()is public,
                         // D::f() is invoked
                pd->f(); // error:D::f()is private
            }
        
    
 25. 浮点数的跨平台问题  
    在一些早期32位的X86机器上，处理器使用用80位拓展精度的寄存器做相关浮点运算，然后再根据是float/double截取成32位或64位。这样就有可能造成一些错误的结果。比如：

          // test machine information
          // Linux gopherbc 2.6.9-22.ELsmp #1 SMP Sat Oct 8 19:11:43 CDT 2005 i686 i686 i386 GNU/Linux
          #include <iostream>
          using namespace std;
           
          int main()
          {
              double d_end = 6;
              double d_step = 0.2;
           
              // here will convert a special 80-bit extended-precision format to 32 bit format, it lead issue
              int n = static_cast<int>(d_end / d_step);
           
              cout << "ori:" << d_end/d_step << ", i:" << n << endl;
              return 0;
          }
          // the result is 30, 29
 

26. 一个多线程下的代码优化实例：  
     优化前：
    
        void WxTaskStateObserver::SubscribeUpdateEvent()
        {
            if (m_life_tracker) return;
            wxMutexLocker locker(m_tracker_lock);
            m_life_tracker.reset(new int(1));
        }
         
        void WxTaskStateObserver::UnsubscribeUpdateEvent()
        {
            if (!m_life_tracker) return;
            wxMutexLocker locker(m_tracker_lock);
            m_life_tracker.reset();
        }
         
        /// This function may run in another thread,
        /// so it will send the data to the handler.
        void WxTaskStateObserver::Update(const StateWithTaskInfos& state_with_task_infos)
        {
            wxMutexLocker locker(m_tracker_lock);
            if (!m_life_tracker) return;
         
            using boost::bind;
            boost::shared_ptr<StateWithTaskInfos> infos_ptr(
                new StateWithTaskInfos(state_with_task_infos));
            TaskEvtListener::AddPendingSignal(
                bind(&WxTaskStateObserver::DoUpdate, this, infos_ptr), m_life_tracker);
        }
    
    优化后：
    
        void WxTaskStateObserver::SubscribeUpdateEvent()
        {
            if (m_life_tracker) return;
            LifeTracker tracker = boost::make_shared<int>(1);
            {
                wxMutexLocker locker(m_tracker_lock);
                m_life_tracker.swap(tracker);
            }
        }
         
        void WxTaskStateObserver::UnsubscribeUpdateEvent()
        {
            if (!m_life_tracker) return;
            LifeTracker tracker;
            {
                wxMutexLocker locker(m_tracker_lock);
                m_life_tracker.swap(tracker);
            }
        }
         
        /// This function may run in another thread,
        /// so it will send the data to the handler.
        void WxTaskStateObserver::Update(const StateWithTaskInfos& state_with_task_infos)
        {
            LifeTracker tracker;
            {
                wxMutexLocker locker(m_tracker_lock);
                if (!m_life_tracker) return;
                tracker = m_life_tracker;
            }
         
            using boost::bind;
            boost::shared_ptr<StateWithTaskInfos> infos_ptr(
                new StateWithTaskInfos(state_with_task_infos));
            TaskEvtListener::AddPendingSignal(
                bind(&WxTaskStateObserver::DoUpdate, this, infos_ptr), tracker);
        }
    
27. 关于GNU linker选项: --wrap symbol  
    可以用这个选项来对symbol符号使用包装函数. 任何对symbol符号的引用会被解析成_wrap_symbol. 而任何对_real_symbol的引用会被解析成symbol.这可以用来为系统函数提供一个包装. 包装函函数应当被叫做_wrap_symbol. 如果需要调用原本的那个函数，那就应该调用_real_symbol.利用这个特性，我们可以把GCC里面默认的异常处理函数包装一下，这样的话，在异常发生的时候，就会先调用我们自己的函数，我们可以在里面做一些处理然后再调用真正的异常处理函数。

        extern "C" void __real___cxa_throw( void* thrown_exception,
                                    std::type_info* tinfo, void ( *dest )( void* ) )
                                    __attribute__(( noreturn ));
        extern "C" void __wrap___cxa_throw( void* thrown_exception,
                                    std::type_info* tinfo, void ( *dest )( void* ) )
        {
            const char* name = reinterpret_cast<const std::type_info*>(tinfo)->name();
            // ... do some other operation
            __real___cxa_throw( thrown_exception, tinfo, dest );
        }
         
        // and wrap the function gxx_personality_v0 is also OK for exception handle and maybe more effective
        extern "C" _Unwind_Reason_Code __real___gxx_personality_v0(int version,
                _Unwind_Action actions,
                _Unwind_Exception_Class excep,
                struct _Unwind_Exception * ex,
                struct _Unwind_Context * context);
         
            extern "C" _Unwind_Reason_Code __wrap___gxx_personality_v0(
                    int version,
                    _Unwind_Action actions,
                    _Unwind_Exception_Class exception_class,
                    struct _Unwind_Exception *ue_header,
                    struct _Unwind_Context *context)
        {
            return PersonalityRoutineWrapper(__real___gxx_personality_v0, version,
                    actions, exception_class, ue_header, context);
        }
    
#### Boost Libraries related
    
1. shared\_ptr 里面有 operator bool().
    如果shared\_ptr里面的指针不为空就会返回true.所以我们可以这样写代码：
    
        shared_ptr<A> a_sp = someFunctionReturningSharedPtr();
        if (a_sp)
        {
            cout << a_sp->someData << endl;
        }
        else
        {
            cout << "Shared Pointer is NULL << endl;
        }
    
2. 使用boost::bind的时候, 如果想绑定参数的话最好绑定一个引用参数。  
   boost::bind会创建一个函数对象。如果有绑定参数的话，参数会被copy一份存到函数对象里面去。如果想保存参数的引用的话，就要使用boost::ref 和boost::cref。
       例如：
    
        class  A {
            public:
               virtual  void print () {
                    cout  << "This is A" << endl;
                };
            };
         
        class B: public A {
        public:
           virtual  void print () {
                cout  << "This is B" << endl;
            };
        };
         
        void print( A & a)
        {
              a.print();
        }
         
        void  test ( A &  a)
        {
              boost::bind(print, a)(); // error, here should use boost::ref(a) or it will output "this is A" which is not expected
        }
         
        int main()
        {
              B  b;
              test (b);
         
              return 0;
        }
    
3. 在使用boost::filesystem里面的函数的时候别忘了进行异常处理,
    因为boost::filesystem里面的函数跟当前的操作系统和文件系统有关，所以很有可能会抛出异常，如果我们没做异常处理的话，程序就会crash. 所以我们应该这样使用，例如：
    
        try
        {
            std::cout << boost::filesystem::exists(file) << std::endl;
        }
        catch (boost::filesystem::filesystem_error &e)
        {
            std::cerr << e.what() << std::endl;
        }
