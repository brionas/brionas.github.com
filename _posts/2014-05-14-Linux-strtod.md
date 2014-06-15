---
layout: post
title: Linux下strtod出错与C++国际化问题 
description: Linux下strtod出错与C++国际化问题
category: C++
tags: strtod
refer_author: pwang
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}



### 问题

最近，公司的GUI程序（基于wxWidgets）在Linux下出来一个bug，说起来有点不可思议。

类似下面的一块代码

	const char* str="0.06";
	char* end=NULL;
	double d = strtod(str, &end);
	printf("str=%s, end=%s, %f\n", str, end, d);

输出结果是：

	str=0.06, end=.06, 0,000000  



难道strtod会出错了吗？不可能吧?!

为了确认这个问题，写一个小程序在客户机子上测试。

`test 1`

	#include <stdio.h>
	#include <stdlib.h>
	
	int main(int argc, char* argv[])
	{
	 const char* str="0.06";
	 char* end=NULL;
	 double d = strtod(str, &end);
	 printf("str=%s, end=%s, %f\n", str, end, d);
	 return 0;
	}


  
 编译并运行这个测试程序，发现结果完全正常。说明GUI程序本身存在某种问题。

后来客户怀疑可能与locale设置相关，就更改了locale设置。客户的机子默认是德语，改成环境变量LC\_ALL=C后，GUI程序能够正常运行了！

strtod在德语环境下会出错？！！

###   重现bug

简单修改测试程序 main.cpp，得到 

`test 2`

	#include <stdio.h>
	#include <stdlib.h>
	#include <locale.h>
	
	int main(int argc, char* argv[])
	{
	    printf("LC_ALL:%s\n"
	        "LC_CTYPE:%s\n"
	        "LC_MESSAGES:%s\n"
	        ,setlocale(LC_ALL, NULL)
	        ,setlocale(LC_CTYPE, NULL)
	        ,setlocale(LC_MESSAGES, NULL)
	        );
	
	    const char* str="0.06";
	    char* end=NULL;
	    double d = strtod(str, &end);
	    printf("str=%s, end=%s, %f\n", str, end, d);
	    return 0;
	}

为了模拟客户的环境，设置当前的LANG为德语：
 运行程序，输出结果是：

	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0,06


这说明test 1 和test 2在进入main函数的时候，程序自身的locale是C。  

难道因为wx程序里面的locale变了，所以出错了吗？

但是我们自身的程序并没有改动locale啊。

在wxApp::OnInit的重载函数里面打印当前的locale看看


	bool App::OnInit () 
	{   
	    wxLogDebug("Current locale : %s", setlocale(LC_ALL, NULL));
	}


结果发现wx程序的默认locale是**de\_DE.UTF-8**, 也就是系统的locale设置。  
   

glibc下只能通过setlocale来修改程序的locale设置，所以wx程序肯定在初始化的过程中调用或间接调用了setlocale函数。

直接gdb下设置setlocale函数断点， 终于找到了出错的位置app.cpp:420

![](/assets/image/2014-05/2014-05-14-linux-strtod/20140324215521078.jpg)  

查看gtk中gtk\_set\_locale的[manual](http://developer.gimp.org/api/2.0/gtk/gtk-General.html#gtk-set-locale)

gtk\_set\_locale ()

	gchar*   gtk_set_locale(void);

	 Initializes internationalization support for GTK+. gtk_init() automatically does 
	 this, so there is typically no point in calling this function.
	
	 If you are calling this function because you changed the locale after GTK+ is was
	 initialized, then calling this function may help a bit. (Note, however, that 
	 changing the locale after GTK+ is initialized may produce inconsistent results and
	 is not really supported.)
	
	 In detail - sets the current locale according to the program environment. This is the 
	 same as calling the C library function setlocale (LC_ALL, "") but also takes care of 
	 the locale specific setup of the windowing system used by GDK.

	 Returns: a string corresponding to the locale set, typically in the form 
	 lang_COUNTRY, where lang is an ISO-639 language code, and COUNTRY is an 
	 ISO-3166 country code. On Unix, this form matches the result of the setlocale(); 
	 it is also used on other machines, such as Windows, where the C library returns 
	 a different result. The string is owned by GTK+ and should not be modified or 
	 freed.



gtk为什么要这么做？

这当然也与Linux下程序的国际化一致。比如shell命令ls出错后会打印出错信息，这些出错信息可以根据当前的语言设置而变化。  

gtk提供一些内置的菜单、按钮之类的控件，比如OK, Cancel
按钮。在不同的语言设置下，这些按钮可以按照当前的默认语言来显示。

  

下面写一个简单的gtk测试程序

`test 3`

	//gtk_main.cpp
	#include <stdio.h>
	#include <stdlib.h>
	#include <locale.h>
	#include <gtk/gtk.h>
	void use_strtod()
	{
	    printf("LC_ALL:%s\n"
	            "LC_CTYPE:%s\n"
	            "LC_MESSAGES:%s\n"
	            ,setlocale(LC_ALL, NULL)
	            ,setlocale(LC_CTYPE, NULL)
	            ,setlocale(LC_MESSAGES, NULL)
	          );
	    const char* str="0.06";
	    char* end=NULL;
	    double d = strtod(str, &end);
	    printf("str=%s, end=%s, %f\n", str, end, d);
	}
	int main(int argc, char* argv[])
	{
	    use_strtod();
	    gtk_set_locale();
	    printf("after gtk_set_locale\n");
	    use_strtod();
	    return 0;
	}

编译后生成**gtk\_test**。测试结果如下：

*$ export LANG=de\_DE.UTF-8*

*$ g++ -o gtk\_test \`pkg-config --cflags --libs gtk+-2.0\`
gtk\_main.cpp*

*$ ./gtk\_test*

*LC\_ALL:C*

*LC\_CTYPE:C*

*LC\_MESSAGES:C*

*str=0.06, end=, 0.060000*

*after gtk\_set\_locale*

*LC\_ALL:de\_DE.UTF-8*

*LC\_CTYPE:de\_DE.UTF-8*

*LC\_MESSAGES:de\_DE.UTF-8*

*str=0.06, end=.06, 0,000000*

**终于在小程序里面重现了bug!**

那么， 为什么德语下会出问题？ 

下面写一个德语locale下的打印测试

`test 4`

	// ge_print.cpp
	#include <iostream>
	#include <locale>
	#include <iomanip>
	using namespace std;
	int main()
	{
	     double a =1234567.123;
	     cout << setprecision(10);
	     cout<< a << endl;
	     locale l("de_DE.UTF-8");
	     cout.imbue(l);
	     cout<< a << endl;
	     return 0;
	}


  

测试结果：

*$ g++ -o ge\_print ge\_print.cpp*

*$ ./ge\_print*

*1234567.123*

*1.234.567,123 //德语格式输出*

  

原因终于出现了：**德语输出数字默认以点号“.”作为千分位的分隔符，而逗号“,"作为小数点！**

  

引申一下，LC开头那么多环境变量和LANG到底是什么关系呢？  

查看glibc的manual [Categories of Activities that Locales
Affect](http://www.gnu.org/software/libc/manual/html_node/Locale-Categories.html#Locale-Categories)


	LC_COLLATE
	    This category applies to collation of strings (functions strcoll and strxfrm); see Collation Functions. 
	LC_CTYPE
	    This category applies to classification and conversion of characters, and to multibyte and wide characters; see Character Handling, and Character Set Handling. 
	LC_MONETARY
	    This category applies to formatting monetary values; see General Numeric. 
	LC_NUMERIC
	    This category applies to formatting numeric values that are not monetary; see General Numeric. 
	LC_TIME
	    This category applies to formatting date and time values; see Formatting Calendar Time. 
	LC_MESSAGES
	    This category applies to selecting the language used in the user interface for message translation (see The Uniforum approach; see Message catalogs a la X/Open) and contains regular expressions for affirmative and negative responses. 
	LC_ALL
	    This is not an environment variable; it is only a macro that you can use with setlocale to set a single locale for all purposes. Setting this environment variable overwrites all selections by the other LC_*variables or LANG. 
	LANG
	    If this environment variable is defined, its value specifies the locale to use for all purposes except as overridden by the variables above.


在Linux下，和国际化相关的locale环境变量有三类：LC\_ALL，LC\_\*（如LC\_CTYPE等），LANG。

根据man 7
locale里的定义，这三者的优先级为LC\_ALL\>LC\_\*\>LANG，即LC\_ALL定义的内容会覆盖LC\_\*和LANG的，LC\_\*会覆盖LANG的。

strtod应该跟**LC\_NUMERIC**有关系。
修改LC\_NUMERIC的设置为C，然后运行test
3，发现strtod出错的问题不见了，确实如此~


###### 解决办法

###### 1)  重置LC\_ALL 为C或POSIX 
	$ bash -c "export LC_ALL=C; locale; ./gtk_test"
	LANG=de_DE.UTF-8
	LC_CTYPE="C"
	LC_NUMERIC="C"
	LC_TIME="C"
	LC_COLLATE="C"
	LC_MONETARY="C"
	LC_MESSAGES="C"
	LC_PAPER="C"
	LC_NAME="C"
	LC_ADDRESS="C"
	LC_TELEPHONE="C"
	LC_MEASUREMENT="C"
	LC_IDENTIFICATION="C"
	LC_ALL=C
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000
	after gtk_set_locale
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000
	
	$ bash -c "export LC_ALL=POSIX; locale; ./gtk_test"
	LANG=de_DE.UTF-8
	LC_CTYPE="POSIX"
	LC_NUMERIC="POSIX"
	LC_TIME="POSIX"
	LC_COLLATE="POSIX"
	LC_MONETARY="POSIX"
	LC_MESSAGES="POSIX"
	LC_PAPER="POSIX"
	LC_NAME="POSIX"
	LC_ADDRESS="POSIX"
	LC_TELEPHONE="POSIX"
	LC_MEASUREMENT="POSIX"
	LC_IDENTIFICATION="POSIX"
	LC_ALL=POSIX
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000
	after gtk_set_locale
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000

###### 2) strtod依赖于LC\_NUMERIC环境变量，那么只改动LC\_NUMERIC也可以

	$ export LC_NUMERIC="C"
	$ echo $LC_NUMERIC
	C
	$ ./gtk_test
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000
	after gtk_set_locale
	LC_ALL:LC_CTYPE=de_DE.UTF-8;LC_NUMERIC=C;LC_TIME=de_DE.UTF-8;LC_COLLATE=de_DE.UTF-8;LC_MONETARY=de_DE.UTF-8;LC_MESSAGES=de_DE.UTF-8;LC_PAPER=de_DE.UTF-8;LC_NAME=de_DE.UTF-8;LC_ADDRESS=de_DE.UTF-8;LC_TELEPHONE=de_DE.UTF-8;LC_MEASUREMENT=de_DE.UTF-8;LC_IDENTIFICATION=de_DE.UTF-8
	LC_CTYPE:de_DE.UTF-8
	LC_MESSAGES:de_DE.UTF-8
	str=0.06, end=, 0.060000

###### 3) 更改LANG环境变量 
	
	$ echo $LANG
	de_DE.UTF-8
	$ bash -c "unset LANG; ./gtk_test"
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000
	after gtk_set_locale
	LC_ALL:C
	LC_CTYPE:C
	LC_MESSAGES:C
	str=0.06, end=, 0.060000
	$ echo $LANG
	de_DE.UTF-8
  
