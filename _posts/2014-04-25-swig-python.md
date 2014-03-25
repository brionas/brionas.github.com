---
layout: post
title: SWIG Python简介
description: SWIG Python简介
category: python
tags: [python]
refer_author: Echo
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

SWIG Python简介
======================================================================================================================================================================================


### 1. SWIG 是什么

SWIG是一种让脚本语言调用C/C++接口的工具，支持Perl, Python,
Ruby等很多脚本。并且对C++的继承、多态、模板，STL等都有较好的支持。使用SWIG，可以实现：

​1) 粘合C/C++库与脚本，快速开发

​2) 交互式Debug

### 2. 第一个例子

假设有如下C/C++程序：

	// example.h
	extern double g_var;
	
	int add(int a, int b);
	
	class Foo
	{
	public:
	    Foo();
	
	    int GetNum() const;
	    void SetNum(int num);
	
	private:
	    int m_num;
	};
	
	// example.cpp
	#include "example.h"
	
	double g_var = 17.001;
	
	int add(int a, int b)
	{
	    return a + b;
	}
	
	Foo::Foo()
	 : m_num(-1)
	{
	}
	
	int Foo::GetNum() const
	{
	    return m_num;
	}
	
	void Foo::SetNum(int num)
	{
	    m_num = num;
	}

为了使用SWIG导出该C++接口，我们需要为定义接口文件：

	// example.i
	%module example // 模块名称
	%{
	#include "example.h"   // %{ }%包围的代码不会被解析，直接插入到所生成的example_wrap.cxx中
	%}
	
	extern double g_var;
	
	int add(int a, int b);
	
	class Foo
	{
	public:
	    Foo();
	
	    int GetNum() const;
	    void SetNum(int num);
	
	private:
	    int m_num;
	};

可以看到，接口文件只需要对C++头文件做少许改动即可（但是如果需要处理指针，传值参数等情况时需要较大改动）。

执行命令swig -c++ -python example.i，SWIG生成了
两个文件：example.py和example\_wrap.cxx。example\_wrap.cxx是一个使用Python
CAPI实现的模块，主要用于转接example.cpp中的函数，example.py是一个为了让我们可以用OO方式使用该模块的中间层，稍后我们会详细分析SWIG的封装方式。

为了简单，使用distutils将生成的wrapper文件和原example.cpp文件一起编译，定义文件setup.py：

	from distutils.core import setup, Extension
	setup(name='example', ext_modules=[Extension("_example", ["example.cpp", "example_wrap.cxx"],
	    extra_compile_args=['-g'])])

执行命令：

python setup.py build

将生成的build/…/\_example.so移到当前目录，进入Python：

	>>> import example
	>>> example.cvar.g_var # 全局变量放在cvar中
	17.001
	>>> example.add(1, 2)
	3
	>>> f = example.Foo()
	>>> f.GetNum()
	-1
	>>> f.SetNum(11)
	>>> f.GetNum()
	11

可以看到，无论是变量、函数还是类定义，SWIG都能很容易地实现导出。

### 3. 封装浅析

SWIG是如何实现导出的呢？首先来看**全局变量**的导出方式，SWIG将所有的全局变量导出在module\_name.cvar里。在example\_wrap.cxx中CAPI初始化函数中：

	int g_var_set(PyObject *_val)
	{
	    double val;
	    int res = SWIG_AsVal_double(_val, &val); // 从PyObject解析int
	    g_var = static_cast< double >(val);
	    return 0;
	}
	
	PyObject *g_var_get(void) {
	    PyObject *pyobj = 0;
	    pyobj = SWIG_From_double(static_cast< double >(g_var)); // 使用int建立PyObject
	    return pyobj;
	}
	
	#define SWIG_INIT init_example
	extern "C" void SWIG_INIT(void)
	{
	    m = Py_InitModule((char *) SWIG_name, SwigMethods); // 注册导出函数
	    d = PyModule_GetDict(m);
	
	    PyDict_SetItemString(d,(char*)"cvar", SWIG_globals()); // _example.__dict__['cvar'] = xxx
	    SWIG_addvarlink(SWIG_globals(),(char*)"g_var",g_var_get, g_var_set); // 添加变量g_var
	}

可以看到，SWIG使用getter和setter函数封装了变量g\_var。SWIG\_globals()返回一个单例PyObject，描述结构体为swig\_varlink\_type，并自定义了\_\_getattr\_\_和\_\_setattr\_\_方法。SWIG\_addvarlink在c\_var中添加了g\_var，cvar的\_\_getattr\_\_和\_\_setattr\_\_方法负责分发访问。

然后看一下SWIG如何对**函数导出**的：

	PyObject *_wrap_add(PyObject *SWIGUNUSEDPARM(self), PyObject *args) {
	    PyObject *resultobj = 0;
	    int arg1 ;
	    int arg2 ;
	    int result;
	    int val1 ;
	    int val2 ;
	    PyObject * obj0 = 0 ;
	    PyObject * obj1 = 0 ;
	
	    if (!PyArg_ParseTuple(args,(char *)"OO:add",&obj0,&obj1)) SWIG_fail;
	    SWIG_AsVal_int(obj0, &val1);
	    arg1 = static_cast< int >(val1);
	    SWIG_AsVal_int(obj1, &val2);
	    arg2 = static_cast< int >(val2);
	    result = (int)add(arg1,arg2);
	    resultobj = SWIG_From_int(static_cast< int >(result));
	    return resultobj;
	}
	
	static PyMethodDef SwigMethods[] = {
	    { (char *)"add", _wrap_add, METH_VARARGS, NULL},
	    ...
	    { NULL, NULL, 0, NULL }
	};
	
	extern "C" void SWIG_init(void) {
	    PyObject *m = Py_InitModule((char *) SWIG_name, SwigMethods); // 注册导出函数
	    ...
	}

Python CAPI本身就支持函数导出，SWIG的实现也很清晰，不在赘述。

然后是**类Foo的导出**，先来看一下Python中Foo对象的一些信息：

	>>> f
	<example.Foo; proxy of <Swig Object of type 'Foo *' at 0x5c8770> >
	>>> type(f)
	<class 'example.Foo'>
	>>> f.this
	<Swig Object of type 'Foo *' at 0x5c8770>
	>>> type(f.this)
	<type 'PySwigObject'>

SWIG使用PySwigObject来保存对象C++指针以及指针类型（swig\_type\_info）：

	typedef struct {
	    PyObject_HEAD
	    void *ptr;
	    swig_type_info *ty;
	    int own;
	    PyObject *next;
	} PySwigObject;

我们对Foo对象的操作可以总结为几种：构造、删除、调用函数。SWIG将这几种操作封装成普通函数导出：

	static PyMethodDef SwigMethods[] = {
	    { (char *)"new_Foo", _wrap_new_Foo, METH_VARARGS, NULL},
	    { (char *)"Foo_GetNum", _wrap_Foo_GetNum, METH_VARARGS, NULL},
	    { (char *)"Foo_SetNum", _wrap_Foo_SetNum, METH_VARARGS, NULL},
	    { (char *)"delete_Foo", _wrap_delete_Foo, METH_VARARGS, NULL},
	    { NULL, NULL, 0, NULL }
	};

调用封装函数\_wrap\_\*时（\_wrap\_new\_Foo除外），将携带指针及类型信息的PySwigObject作为参数传递（以\_wrap\_Foo\_GetNum为例）：

	PyObject *_wrap_Foo_GetNum(PyObject *SWIGUNUSEDPARM(self), PyObject *args) {
	    PyObject *resultobj = 0;
	    Foo *arg1 = (Foo *) 0 ;
	    int result;
	    void *argp1 = 0 ;
	    PyObject * obj0 = 0 ;
	
	    PyArg_ParseTuple(args,(char *)"O:Foo_GetNum",&obj0); // obj0即Python中Foo对象的self
	    SWIG_ConvertPtr(obj0, &argp1,SWIGTYPE_p_Foo, 0 |  0 ); // 从PySwigObject(self.this)中拿到指针
	    arg1 = reinterpret_cast< Foo * >(argp1);
	    result = (int)((Foo const *)arg1)->GetNum();
	    resultobj = SWIG_From_int(static_cast< int >(result));
	    return resultobj;
	}

为了方便调用（或者说实现OO式调用），SWIG使用Shadow
Class进一步对其进行了封装，example.py中定义了一个Foo类，并在其成员函数中调用\_example中导出的函数：

	class Foo(_object):
	    def __init__(self, *args):
	        this = _example.new_Foo(*args) // 新建Foo，保存在self.this中
	        try: self.this.append(this)
	        except: self.this = this
	    def GetNum(*args): return _example.Foo_GetNum(*args)
	    def SetNum(*args): return _example.Foo_SetNum(*args)
	    __swig_destroy__ = _example.delete_Foo // _example.so会调用
	    __del__ = lambda self : None;

对于很多复杂类型，例如数组，引用等，SWIG都使用指针来表示，方法与类封装类似。如果希望使用其他方式，SWIG提供了typemap来实现自定义类型传递控制，具体参见SWIG手册。

总结：

本文只是介绍了SWIG
Python对C+的基本使用方法，还有很多内容没有涉及，例如操作符，继承、多态、模板等。由于C+本身复杂的特性，要支持这些，使用SWIG时还需要注意一些繁琐的东西。

另外如果想要了解SWIG实现交互式调试的方法，可以参考*[使用Swig
Python动态绑定C++对象](http://www.cnblogs.com/xiao-liang/p/3569277.html)*
