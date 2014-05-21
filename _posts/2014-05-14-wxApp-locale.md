---
layout: post
title: Windows下wxApp的locale设置 
description: Windows下wxApp的locale设置  
category: wxWidgets C++
tags: wxWidgets
refer_author: pwang
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}



在写完文章`Linux下strtod出错与C++国际化问题`有一个新的疑问：strtod和wxApp在Windows下表现如何呢。

查询MSDN得知，VC同样提供了setlocale函数。

  

 [setlocale 函数说明](http://msdn.microsoft.com/zh-cn/library/x99tb11d.aspx)

	char *setlocale(
	   int category,
	   const char *locale 
	);
	wchar_t *_wsetlocale(
	   int category,
	   const wchar_t *locale 
	);


category参数：

LC\_ALL

所有类别，如下所示。

LC\_COLLATE

strcoll 、 \_stricoll, wcscoll、\_wcsicoll、 strxfrm、 \_strncoll、 \_strnicoll、\_wcsncoll、 \_wcsnicoll 和 wcsxfrm 函数。

LC\_CTYPE

字符处理函数 (除外 isdigit、isxdigit、mbstowcs和 mbtowc，不受影响)。

LC\_MONETARY

localeconv 函数返回货币格式信息。

LC\_NUMERIC

对于格式化输出实例
(例如 printf)，数据转换实例和通过 localeconv返回的非货币格式信息的十进制点字符。    除十进制点字符之外，还有 LC\_NUMERIC 设置通过 [localeconv](http://msdn.microsoft.com/zh-cn/library/8658zdx3.aspx) 返回的千位分隔符和分组控件字符串。

LC\_TIME

strftime 、 wcsftime 函数。


与[glibc的函数声明](http://www.gnu.org/software/libc/manual/html_node/Locale-Categories.html#Locale-Categories)相比，category参数接受的类型稍微少了一点儿。

  

###下面简要测试下wxApp在Windows下的行为


测试代码如下：


	bool App::OnInit () 
	{   
	    wxLogDebug("Current code page : %s", setlocale(LC_CTYPE, NULL));
	    //".ACP" means using the code page which OS uses currently 
	    setlocale(LC_CTYPE, ".ACP");
	    wxLogDebug("Current code page : %s", setlocale(LC_CTYPE, NULL));
	    ...
	}


输出结果是：

![](/assets/image/2014-05/2014-05-14-wxApp-locale/20140325212540781.jpg)

现在可知，**wxApp在Win****dows上的locale默认是C，没有使用系统的locale。这与Linux下的行为截然不同。**  

不得不说，wx库在跨平台开发中真是到处是陷阱。。。
   

参考MSDN的[区域设置名称、语言和国家/地区字符串](http://msdn.microsoft.com/zh-cn/library/hzz3tw78.aspx)说明，locale的表述格式如下：


	locale :: "locale_name"
	        | "language[_country_region[.code_page]]"
	        | ".code_page"
	        | "C"
	        | ""
	        | NULL


所以输出的字符串具体可以解释为：

	language： Chinese
	
	country region: Peoples's republic of China
	
	code page: 936

代码936表示 GBK简体中文，1252表示西欧拉丁语系。


另外，
Windows下通过setlocale来设置时，locale字符串参数也提供了一定的灵活性。

比如，下面三行代码所表达的意思一致。

	snippet_file_name="blog_20140325_4_177866" name="code"}
	setlocale( LC_ALL, "en-US" );
	setlocale( LC_ALL, "English" );
	setlocale( LC_ALL, "English_United States.1252" );


接着测试下Windows下德语是否还有问题

	#include <stdio.h>
	#include <stdlib.h>
	#include <locale.h>
	void use_strtod()
	{
	    printf("LC_ALL:%s\n"
	        "LC_CTYPE:%s\n"
	        //"LC_MESSAGES:%s\n"
	        , setlocale(LC_ALL, NULL)
	        , setlocale(LC_CTYPE, NULL)
	        //,setlocale(LC_MESSAGES, NULL)
	        );
	    char* str = "0.06";
	    char* end = NULL;
	    double d = strtod(str, &end);
	    printf("str=%s, end=%s, %f\n", str, end, d);
	}
	void change_locale(int cat, const char* str)
	{
	    if (!setlocale(cat, str))
	        printf("fail to change locale: %s \n", str);
	    else
	        printf("success to change locale: %s \n", str);
	}
	int main(int argc, char* argv[])
	{
	    use_strtod();
	    setlocale(LC_ALL, ".ACP"); //to avoid using the default "C" locale.
	    printf("System locale: %s\n", setlocale(LC_ALL, NULL));
	    //change_locale(LC_ALL, "en-US"); //fail
	    //change_locale(LC_ALL, "English"); //sucess
	    change_locale(LC_ALL, "English_United States.1252");
	    //change_locale(LC_ALL, "de-DE"); //fail
	    //change_locale(LC_ALL, "German"); //sucess
	    change_locale(LC_ALL, "German_Germany.1252");
	    use_strtod();
	    change_locale(LC_NUMERIC, "C");
	    use_strtod();
	    return 0;
	}

对这个小程序稍微解释下：

​1.
程序第8行和11行之所以注释掉，是因为Windows下没有LC\_MESSAGES这个参数。

​2. line 20： 通过判断setlocale的返回值，可以知道到底当前操作成功没有。

​3. main函数先后测试在默认locale、系统locale和德语下strtod的运行情况。

​4. line 37：仅仅修改LC\_NUMERIC，看看strtod是否能正确工作。

​5. line 30和33在某些机子上会失败，所以一定要检测setlocale的返回值。

测试结果是：

![](/assets/image/2014-05/2014-05-14-wxApp-locale/20140325212511359.jpg)  

这说明Windows
下德语locale也存在同样的问题，同样可以通过修改locale的设置来避免这个问题发生。

  

**那么在不改变locale的情况下能不能实现自定义的数字格式输出呢？** 

C++提供了相应的类来解决这种需求。C++ locale的用法可以参考
[C++manual](http://www.cplusplus.com/reference/locale/)

测试代码如下：

	#include <iomanip>
	#include <iostream>
	using namespace std;
	
	template <typename T>
	struct thousands: std::numpunct <T>
	{
	public:
	    thousands() { }
	protected:
	    T do_thousands_sep() const { return '.';}
	    std::basic_string <T> do_grouping() const { return "\3"; }
	    T do_decimal_point() const { return ','; }
	};
	
	int main()
	{
	    locale loc( locale::classic(), new thousands <char> () );
	    cin.imbue( loc );
	    cout.imbue( loc );
	
	    //cout << "LC_ALL: " << setlocale(LC_ALL,NULL) << endl;
	    cout << setprecision(10);
	    cin >> setprecision(10);
	
	    cout << "output number 123456.1 > " << 123456.1 << endl;
	
	    double d;
	    cout << "Enter a number > ";
	    cin >> d;
	    cout << "Your input number is " << d << endl;
	
	    return 0;
	}


  
 测试结果：

	output number 123456.1 > 123.456,1
	Enter a number > 654.321,123
	Your input number is 654.321,123
  
 这个测试程序在Linux同样可以正常运行。