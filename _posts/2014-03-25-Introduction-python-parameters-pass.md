---
layout: post
title: python参数传递介绍
description: python参数传递介绍
category: python
tags: [python]
refer_author: 
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

python参数传递介绍
===========================================================================================================================================================================================================================================================================================


python函数传递参数大家应该都有了解，网上的资料比较多且杂，
这里统一整理分成以下三部分介绍：

1.  **参数传递方式**  
2.  **参数类型**  
3.  **默认参数**  

#### 1. 参数传递方式 

在C++中，函数传递方式有值传递和引用传递两种方式:

-   值传递，函数调用时，系统会生成一份相关参数值的拷贝，函数体内关于该参数的操作都作用于这份拷贝，函数退出后函数调用者的相关变量值不会改变。  

-   引用传递：与值传递会生成一份参数值的拷贝不同，引用传递只会保存一份关于参数的引用，在函数体内给予该参数的修改等操作，都会影响到函数体外相关变量的值，这样虽然可能由于函数调用会使变量值发生意向不到的改变，但由于不需要对整个参数所占的内存做全局拷贝，引用传递在效率上会有很大的提升。

在python中，严格意义我们不能说值传递还是引用传递，python中一切都是对象，我们应该说传不可变对象和传可变对象。

###### **传不可变对象**

当函数参数为不可变对象（整数，字符串，元组）时，函数体内的参数在被改变之前，会一直持有该对象的引用，但当参数发生改变时，由于该对象为不可变对象，必须生成一份新的拷贝作为函数的本地变量，函数对该本地变量的修改不会影响函数调用者的变量值，这一点类似于上面所说的值传递，见下面代码：  
     

	def func_pass_value(x):
	    print "before: x=", x, " id=", id(x)
	    x = 2
	    print "after: x=", x, " id=", id(x)
	>>> x = 1
	>>> func_pass_value(x)
	before: x= 1  id= 5363912
	after: x= 2  id= 5363888
	>>> x, id(x)
	(1, 5363912)

###### **传可变对象**

当函数参数为可变对象（列表，字典）时，除非发生赋值操作，函数体类的参数会一直持有该对象的引用，函数对该参数的修改也会影响到函数调用者的变量值，这一点类似于上面介绍的引用传递。但在函数体内发生赋值操作时，也会生成一份新的拷贝作为函数的本地变量，函数对该本地变量的修改不会影响到函数调用者的变量值，见下面代码：

	def func_pass_ref1(list):
	    print list
	    list = [4, 5]
	    print list
	
	>>> def func_pass_ref2(list):
	    print list
	    list += [4, 5]
	    print list
	
	>>> al = [1, 2, 3]
	>>> func_pass_ref1(al)
	[1, 2, 3]
	[4, 5]
	>>> al
	[1, 2, 3]
	>>> func_pass_ref2(al)
	[1, 2, 3]
	[1, 2, 3, 4, 5]
	>>> al
	[1, 2, 3, 4, 5]

#### 2. 函数参数类型

python函数传递参数类型比较多，按照是否确定参数数目可分为定长参数变长参数，按照是否引入关键字分为普通参数和关键字参，我们从以下五个方面进行介绍：

-   定长普通参数  
-   定长关键字参数  
-   变长普通参数  
-   变长关键字参数  
-   复合参数  

###### **定长普通参数**

定长普通参数为最常见的参数传递方式，参数传递时要求实参和虚参的个数相同，顺序也必须相同，否则会出现错误

	def fun1(a, b):
	  print a+b
	
	if __name__ == "__main__":
	  a = 2
	  b = 3
	  fun(a, b)

如果要在设置默认参数，默认参数应该放在参数链表的尾部,
否则在函数调用时会给默认参数指定实参，从而使默认参数的设置失去意义。

	def fun1(a, b，c=10):
	  print a+b+c
	
	if __name__ == "__main__":
	  a = 2
	  b = 3
	  fun(a, b)
	  fun(a, b, 4)

定长普通参数对参数位置要求严格，要想摆脱位置对参数的束缚，python引入了关键字参数。

###### **定长关键字参数**

函数采用定长关键字参数进行参数传递时，必须设定指定参数的名字，函数调用时将以键值对的方式进行传递，若要给某个参数设定缺省值，在函数定义时直接指定缺省键值对即可，且不受位置的束缚，在函数调用时则不需要设定指定参数的键值对。

	def fun2(Name='Tom', Age=20, Sex="Male"):
	  print 'Name:', Name
	  print 'Age:', Age
	  print 'Sex:', Sex
	
	if _name_ == "_main_":
	  fun2(Name="Alice", Sex="Female")
	  fun2(Name="Bob", Age=22)

有些情况我们在定义一个函数时，并不能能够确定接受的参数的个数，只是知道需要对所接受的参数依次进行处理，这种情况下需要使用变长参数进行参数传递，python在定义变长参数时引入了\*,
 定义变长普通参数时在参数名前面放在\*，如\*args，定义变长关键字参数时在参数前面放两个\*号，如\*\*kargs。下面分别进行介绍：

###### **变长普通参数**

变长普通参数只需要在参数前面放一个\*号即可，在函数体内部把形参当作一个参数链表或元组来处理即可， 当函数使用了变长普通参数进行参数传递时，我们把不知道元素个数的链表或元组当做参数传递给它，只是在参数传递时需要对链表或元组进行unpack，即在相应参数前面加\*号。

	def fun3(*args):
	  for arg in args:
	    print arg
	
	if _name_ == "_main_":
	  foo(1,2,3,"good")
	  arglist = 1,2,3,"good"
	  foo(*arglist)

要注意的是在函数定义时变长参数（包括后面介绍的变长关键字参数）必须设置在定长参数后面。

###### **变长关键字参数**

变长关键字长参数类似于变长普通参数，只是变长关键字参数在函数定义时在参数前面加两个\*号，在函数体内部把参数当作一个参数字典来处理，并且在函数调用时可以把一个字典当作参数传递给它，同样也需要对字典进行unpack，即在字典参数前面加\*\*.

	def fun4(**kwargs):
	  for k, v in kwargs.items():
	    print k, v
	
	if _name_ == "_main_":
	  fun4(Name="Jim", Age=22， Sex="Male")
	  argDict = {Name:"Jim", Age:22, Sex:"Male"}
	  fun4(**argDict)

对于变长关键字参数，我们可以任意指定关键字的名字，函数调用的内部会去获取关键字的名字和值。

###### **复合参数**

在实际编程的过程当中，我们可能面临这样一种情况，函数定义时不能够确定会接受到什么样参数，包括参数的数目，参数的类型等，这个时候，我们就会用到这里介绍的复合参数，复合参数及前面介绍的四种参数的混搭形式。

	fun5(name, Age="20", *args, **kwargs):
	  print name
	  print "Age: ", Age
	  for arg in args:
	  print arg for k, v in kwargs.items():
	  print k, v
	
	if _name_ == "_main_":
	  arglist = 1,2,3,"good"
	  argDict = {Name:"Jim", Age:22, Sex:"Male"}
	  fun4("Jim", Age=22, *arglist, **argDict)

只是在函数定义时对参数的顺序特定的要求：

-   关键字参数必须在普通参数后面  
-   变长普通参数必须放在定长参数的后面  
-   变长关键字参数必须放在变长普通参数的后面  

函数调用时赋值的过程为：

	依次为普通参数赋值  
	为关键字参数赋值  
	将多余出的即键值对行后的零散实参打包组成一个元组传递给赋给变长普通参数*args  
	将多余的键值对形式的实参打包一个字典传递给变长关键字参数**kargsdef   

#### **3. 关于默认参数**

python采用可变对象（列表， 字典）作为默认参数时有一个很奇怪的地方：

	def func_default(data=[]):
	    data.append(1)
	    return data, id(data)
	>>> func_default() ([1], 182898096392)
	>>> func_default() ([1, 1], 182898096392)
	>>> func_default() ([1, 1, 1], 182898096392)

由上面可以看到，随着函数调用采用默认参数的次数越多，函数返回的列表越来越长，列表的地址却保持不变，这个现象对象python新手来说可能会很奇怪，关于这个问题，相关manual里边已经做了说明：

**Default parameter values are evaluated when the function definition is
executed.** This means that the expression is evaluated once, when the
function is defined, and that the same “pre-computed” value is used for
each call. This is especially important to understand when a default
parameter is a mutable object, such as a list or a dictionary: if the
function modifies the object (e.g. by appending an item to a list), the
default value is in effect modified.

即函数的默认参数只在函数定义时才计算默认参数值，以后每次使用默认参数的函数调用都只是在同一个对象上进行操作，可以通过函数对象的func\_defaults变量得到默认参数的当前值：

	def func_default(data=[]):
	    data.append(1)
	    return data
	
	>>> func_default.func_name
	'func_default'
	>>> func_default.func_defaults
	([],)
	>>> func_default()
	[1]
	>>> func_default.func_defaults
	([1],)
	>>> func_default()
	[1, 1]
	>>> func_default.func_defaults
	([1, 1],)
	>>> func_default()
	[1, 1, 1]
	>>> func_default.func_defaults
	([1, 1, 1],)

若想要达到你正常的效果，可以用一个占位符来替代改默认值：

	def func_default(data=None):
	    if data is None:
	        data = []
	    data.append(1)
	    return data
	
	>>> func_default()
	[1]
	>>> func_default()
	[1]
	>>> func_default()
	[1]

#### Reference

1.  [http://docs.python.org/2/reference/compound\_stmts.html\#function](http://docs.python.org/2/reference/compound_stmts.html#function)
2.  [http://www.python-course.eu/passing\_arguments.php](http://www.python-course.eu/passing_arguments.php)
3.  [http://effbot.org/zone/default-values.htm](http://effbot.org/zone/default-values.htm)