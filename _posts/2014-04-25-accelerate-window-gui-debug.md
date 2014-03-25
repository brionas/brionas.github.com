---
layout: post
title: 加速Windows GUI debug版本的编译
description: 加速Windows GUI debug版本的编译
category: compile
tags: [编译]
refer_author: Zero
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

加速Windows GUI debug版本的编译
=================================================


  

**加速Windows GUI debug版本的编译**

**1. 问题描述**   
我们重构我们的GUI程序时，增加了很多小的工程库，VC2008编译GUI最顶层DLL库libpkgA的速度让人几乎无法忍受。
  
以下是从我们的buildbot系统里截取出来的LOG：

> 28\>…   
> 28\>Embedding manifest…   
> 28\>Build Time 188:17

编译时间**188分钟**。

GUI app依赖于这个DLL库，同时也依赖其它一些库，编译它的LOG如下：

> 79\>…   
> 79\>Embedding manifest…   
> 79\>Build Time 157:32

又需要**157分钟**。

如果算上编译其它子库的时间(\~37mins)，那么编译GUI app所需要的时间为
382mins (6.4hours)。   
我想，clean build DEBUG GUI app需要6.4个小时的时间几乎是无法忍受的。

我们工程项目的库依赖如下：   
[![lib-deps](/assets/image/2014-03/2014-04-25-accelerate-window-gui-debug_files/091329_x1Me_1422693.png)](/assets/image/2014-03/2014-04-25-accelerate-window-gui-debug_files/091329_x1Me_1422693.png)
  
最顶层的DLL库错综复杂的依赖下面所有的静态库。

**如何加速这个编译过程？**

​1) Google? 当你输入"VC compile link
improve"这类关键字时，得到可能会有参考价值的结果可能有：

-   http://blogs.msdn.com/b/vcblog/archive/2010/04/01/vc-tip-get-detailed-build-throughput-diagnostics-using-msbuild-compiler-and-linker.aspx
-   http://blogs.msdn.com/b/vcblog/archive/2009/09/10/linker-throughput.aspx
-   http://stackoverflow.com/questions/143808/how-to-improve-link-performance-for-a-large-c-application-in-vs2005

​2)
到SO提问，我想我们还没搞清楚为什么编译慢，那个环节慢，没头没脑去提问，估计也得不到很好的结果。

面对如此情况，我们只能硬着头皮上，尝试着先把问题分析下，认识清楚我们所面对的问题。
  
**请注意，我们VC版本是2008。**

**2. 问题分析**   
既然编译APP和DLL情况类似，以下就拿编译GUI app时存在的问题来分析：   
1) 找到以前编译GUI app成功的buildlog.htm文件，进行分析   
下面是以前编译GUI app时在项目的临时目录中产生的buildlog.htm的记录：

> Creating command line
> """e:\\GUIApp\\Build-vc90\\guiapp\\Debug\\BAT00001811168124.bat"""   
> Creating temporary file
> "e:\\GUIApp\\Build-vc90\\guiapp\\Debug\\RSP00001911168124.rsp" with
> contents   
> \[   
> /Od /I "D:\\wxWidgets-2.8.9\\src" /I "D:\\wxWidgets-2.8.9\\include" /I
> "D:\\wxWidgets-2.8.9\\contrib\\include" /I
> "E:\\GUIApp\\third\_party\\wxWidgets-2.8.9\\lib\\vc\_lib\\mswd" /I
> "E:\\GUIApp\\third\_party\\propgrid\\include" (more options)   
> \]   
> Creating command line "cl.exe
> @"e:\\GUIApp\\Build-vc90\\guiapp\\Debug\\RSP00001911168124.rsp"
> /nologo /errorReport:prompt"

可以看到VC会在编译(compile，调用cl.exe)之前先推导出该库所需要的编译选项，保存到一个临时文件中，之后以这个临时文件作为输入启动cl.exe编译项目的代码。

> Creating temporary file
> "e:\\GUIApp\\Build-vc90\\guiapp\\Debug\\RSP0000131924013376.rsp" with
> contents   
> \[   
> /OUT:"....\\Build-vc90\\bin\\Debug\\guiapp.exe" /INCREMENTAL:NO
> /LIBPATH:"E:\\GUIApp\\third\_party\\guiapp-Libs\\vc90\\Debug"
> /LIBPATH:"....\\Build-vc90\\bin\\Debug"
> /LIBPATH:"E:\\GUIApp\\third\_party\\guiapp-Libs\\vc90\\DLL\_Debug\\wxlib28"
> /LIBPATH:"E:\\GUIApp\\third\_party\\BugTrap\\Win32\\Bin\\"
> /LIBPATH:"E:\\GUIApp\\third\_party\\win32\\protobuf\\2.3.0\\dll32"
> /MANIFEST
> /MANIFESTFILE:"....\\Build-vc90\\guiapp\\Debug\\guiapp.exe.intermediate.manifest"
> /MANIFESTUAC:"level='asInvoker' uiAccess='false'" /DEBUG
> /PDB:"e:\\GUIApp\\Build-vc90\\bin\\Debug\\guiapp.pdb"
> /SUBSYSTEM:WINDOWS /DYNAMICBASE /NXCOMPAT /MACHINE:X86 libexpatd.lib
> libpkgPlatform.res odbc32.lib odbccp32.lib comctl32.lib rpcrt4.lib
> wsock32.lib libprotobuf.lib winmm.lib wxmsw28d.lib wxmsw28d\_stc.lib
> wxexpatd.lib wxPlotd.lib wxpngd.lib wxzlibd.lib wxjpegd.lib
> wxtiffd.lib xerces-c\_2d.lib lua51d.lib libeay32d.lib libexpatd.lib
> ssleay32d.lib lokisd.lib BugTrap.lib log4cplusD-1\_0\_4.lib
> wxcode\_msw28d\_propgrid.lib wxCode\_msw28d\_treectrl.lib xrc.lib
> chartdir50.lib zlibwapi.lib cairo.lib kernel32.lib user32.lib
> gdi32.lib winspool.lib comdlg32.lib advapi32.lib shell32.lib ole32.lib
> oleaut32.lib uuid.lib kernel32.lib user32.lib gdi32.lib winspool.lib
> comdlg32.lib advapi32.lib shell32.lib ole32.lib oleaut32.lib uuid.lib
> odbc32.lib odbccp32.lib "....\\build-vc90\\bin\\debug\\libpkgA.lib"
> "....\\build-vc90\\bin\\debug\\libD.lib"
> "....\\build-vc90\\bin\\debug\\libC.lib"
> "....\\build-vc90\\bin\\debug\\libD.lib"
> "....\\build-vc90\\bin\\debug\\libE.lib" "....\\buil   
> "....\\Build-vc90\\guiapp\\Debug\\GUIApp.obj"   
> "....\\Build-vc90\\guiapp\\Debug\\A.obj"   
> "....\\Build-vc90\\guiapp\\Debug\\B.obj"   
> "....\\Build-vc90\\guiapp\\Debug\\main.obj"   
> \]   
> Creating command line "link.exe
> @"e:\\GUIApp\\Build-vc90\\guiapp\\Debug\\RSP0000131924013376.rsp"
> /NOLOGO /ERRORREPORT:PROMPT"

可以看到VC会在链接(link，调用link.exe)之前先推导出该库所需要的编译选项，保存到一个临时文件中，之后以这个临时文件作为输入启动link.exe进行链接。

需要注意的是，推导依赖库并创建编译和链接选项的2个临时文件，是在编译(compile)之前做的。这会带来一个问题，就是如果库的依赖非常复杂，VC的自动推导过程将会变得非常慢。
  
2) 开启任务管理器观察VC++(devenv.exe)进程的活动状态   
[![process](/assets/image/2014-03/2014-04-25-accelerate-window-gui-debug_files/img_proxy.png)](/assets/image/2014-03/2014-04-25-accelerate-window-gui-debug_files/img_proxy.png)
  
你不可能傻傻的坐在电脑前等着微软的VC进程恢复到正常状态，于是你去看会儿书，浏览下今天的新闻，又或者去刷刷火车票，在你做完这些之后，回到VC界面，然后发现，我靠，还是没有任何LOG输出。你毅然果然的Kill
devexe进程。然后重新打开solution文件，开启Linker选项中的"Show
Progress"选项：Display All Progress Messages
(/VERBOSE)，又启动app的build。去看部电影再回来。。。

回来之后，你可能会看到下面类似的输出:

> 28\>Linking…   
> 28\>Starting pass 1   
> 28\>Processed /DEFAULTLIB:oleacc   
> 28\>Processed /DEFAULTLIB:msvcprtd   
> 28\>Processed /DEFAULTLIB:uuid.lib   
> 28\>Processed /DEFAULTLIB:libboost\_regex-vc90-mt-gd-1\_37.lib   
> 28\>Processed /DEFAULTLIB:libboost\_signals-vc90-mt-gd-1\_37.lib   
> 28\>Processed /DEFAULTLIB:MSVCRTD   
> 28\>Processed /DEFAULTLIB:OLDNAMES   
> 28\>AppBase.obj : warning LNK4075: ignoring '/EDITANDCONTINUE' due to
> '/INCREMENTAL:NO' specification   
> 28\>Searching libraries   
> 28\> Searching ....\\Build-vc90\\bin\\Debug\\xrc.lib:   
> 28\> Found "void **cdecl ui\_xrc::InitXmlResource(void)"
> (?InitXmlResource@ui\_xrc@@YAXXZ)   
> 28\> Referenced in AppCommonObjMgr.obj   
> 28\> Loaded xrc.lib(tmp\_init.obj)   
> 28\> Found "void**cdecl xml\_setting\_dlg\_init(void)"
> (?xml\_setting\_dlg\_init@@YAXXZ)   
> 28\> Referenced in xrc.lib(tmp\_init.obj)   
> 28\> Loaded xrc.lib(xml\_setting\_dlg.obj)   
> 28\> Found "void \_\_cdecl
> verify\_setup\_tflex\_submask\_editor\_init(void)"
> (?verify\_setup\_tflex\_submask\_editor\_init@@YAXXZ)   
> 28\> Referenced in xrc.lib(tmp\_init.obj)

可以看到VC会自动从推导出的依赖库中尝试resolve所有未解决的符号。

**3. 方案提出**   
1) 将libpkgA动态库编译改成静态库编译   
之前问题中，不仅GUI app编译器会慢，libpkgA
DLL库编译也很慢，时间上不相上下。为什么编译libpkgA DLL库也这样慢呢？   
要知道libpkgA
DLL是配置成依赖于其它静态库的，我们知道静态库可以看成是obj文件的打包集合，当编译这个DLL库的时候，VC会根据项目依赖库列表自动推导所有的依赖库，而且是递归的，因为被依赖的静态库依赖其它静态库，后者的符号并没有进入到前者的LIB文件中。像libpkgA依赖的库非常庞杂，VC递归的推导将会非常慢。(奇怪的是，为什么这个过程会这么慢，它应该只会生成一个完整的库依赖列表就可以了，难道它需要用解决那些静态库的符号的方法进行判断，决定某个库最终是否放到最后的依赖列表中？)

改成静态库编译之后，仅仅只是许多obj文件的打包操作，故可以非常快。   
同时改成静态库之后，也避免了相同的符号在最终的app
exe运行加载这个DLL后中存在两份的可能性，因为app
exe也可以链接那些静态库。

​2) 显示的告诉VC，一个工程所依赖的库   
对于我们自己编写的依赖库，我们通常不会显式的写到项目的编译选项中(VC的项目linker页面input栏)，而是会将一些外部依赖的第三方库写在那里，因为那些第三方库不会频繁的更改。VC2008中可以设置库的相互依赖性(Project/Project
Dependencies)，然后加上Link Library Dependency=yes。   
一种方案是我们可以将自己编写的依赖库显式写到项目的编译选项中，另一种方案就是在代码中写如下的pragma指示编译器去依赖某个库:



	#pragma comment(lib, "xxx.lib")


​3) 建立一个dummy工程(prebuild)来触发依赖库的预编译   
显式告诉VC编译器去依赖某个外部库，而不让它自己去推导，会带来一问题就是，当我们编译工程时，VC自己不会先去编译那些显式依赖的库(它会认为它们已经准备好了)，它也不知道该怎么去编译它们，故如果某个外部库没先编译好，链接工程时就无法打开那个库的.lib文件。
  
有什么方式可以让VC先去编译那些外部依赖库呢？用prebuild命令的方式。   
VC提供prebuild命令的机制，允许先运行一个外部命令，然后再编译工程。当然也支持编译工程之后再运行一个外部命令，叫做Post-build。
  
外部命令通常是一个BAT脚本文件，因此可以写一个BAT脚本来干点什么？   
写一个BAT脚本来一个一个的编译那些外部依赖库？代码如下:

	  for %%L in (%LIBS%) do(	
	  echo Building %%L ...
	  <<<command to build one lib %%L>>>
	  )


编译一个库工程的编译命令是什么？直接google，可以用devenv.com/devenv.exe外部命令带库工程的名字办到(其实devenv.exe是VC的可执行文件)，于是写成下面这样：？


	  for %%L in (%LIBS%) do (
	  echo Building %%L ...
	  devenv.com "%solution%" /Build "%config%"  /project %%L
	  )


试想下，如果依赖的库工程很多，devenv.com外部命令就需要启动很多次。我想这样不太好。
  
想到一句名言：软件工程中问题，通常可以引入一个间接层来解决。   
如果不断执行devenv.com命令开销有点大，可以引入一个中间依赖库libprebuild，让编译工程prebuild这个libprebuild依赖库，更重要的一部是让libprebuild工程依赖所有原先编译工程依赖的外部库。
  
假如编译工程是app，以前依赖于库工程libA, libB, …..
libN，现在引入中间依赖库之后，让libprebuild工程依赖于libA, libB, …,
libN，然后让app先执行prebuild命令，编译libprebuild工程，这样一来编译app工程，会先编译libprebuild工程，编译之前VC会自动推导出它的依赖库，然后就先编译libA,
libB, …, libN。这样就好像编译app工程预先编译libA, libB, …, libN一样。   
特别需要注意的是，libprebuild库工程需要编译成静态库，否则又回到老问题上了。

**4. 总结**   
采用了上述方案之后，任何app工程中如果存在编译变慢的问题(因为VC龟速的自动的库推导/展开)，编译采用上述的方案。步骤如下：
  
1) 在该工程中，增加一个cpp文件，里面显示依赖外部库:


	#pragma comment(lib, "A.lib")
	#pragma comment(lib, "B.lib")
	...


​2) 写一个prebuild.bat脚本，里面写上类似下面的代码(我知道Window
Batch脚本的语法很怪异):


	for %%L in (%LIBS%) do (
	echo Building %%L ...
	devenv.com "%solution%" /Build "%config%"  /project %%L
	)


其中LIBS就是那个创建出来中间依赖库: set LIBS=libprebuild,
config可以是Debug或者其它。

​3) 在工程中的Prebuild Event项的Command栏中写下如下的指令：


	call $(SolutionDir)\libprebuild\prebuild.bat $(ConfigurationName)


​4) 最后也是最关键的一步: 在Project\\Project
Dependencies对话框中，取消勾选该工程所有的依赖库项。

按照上述方法编译原来的工程，你会得到< 10 mins的编译时间。

最后说下该方案存在的问题就是：它不能支持VC界面上的Build
Only菜单项，因为不管怎样，它总是需要去启动prebuild.bat脚本编译libprebuild工程。

**5. 更好的解决方案**    

上述方案中引入的中间依赖库的问题有点繁琐，其实VC2008已经提供了一个选项"Link=> Link Library Dependencies"，如果将其改成NO， 即使在"ProjectDependency"对话框中勾选上依赖库，VC2008将不会进行自动推导并进行依赖库的展开，但是可以帮助我们预编译那些依赖库。我想这就是prebuild库引入想要的解决的问题。这样一来的话，对于原始问题的解决，我们的方案就比较让人满意了。


声明：OSCHINA 博客文章版权属于作者，受法律保护。未经作者同意不得转载。
