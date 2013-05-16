---
layout: post
title: "[Chrome源码阅读]Chrome启动代码流程1"
description: Chrome启动代码流程：(v2.0版，Windows平台)
category: "chrome"
tags: [chrome, 源码分析]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/categories.html
refer_post_addr: http://zeroli.github.io/chrome/2013/03/25/chrome-code-launch-procedure/
---
{% include JB/setup %}

**Chrome启动代码流程：(v2.0版，Windows平台)**

应用程序启动过程：
1) WinMain函数为入口点，定义在文件\chrome\app\chrome_exe_main.cc文件中（位于chrome_exe工程项目中）  
2) WinMain从注册表中找到当前版本的子目录，然后装载chrome.dll文件。如果没找到，则直接从当前exe目录查找dll文件，并装载。  
3) 直接从chrome.dll中找到函数ChromeMain，然后调用它。ChromeMain函数定义在\chrome\app\chrome_dll_main.cc文件中，位于chrome_dll工程项目中。
{% highlight cpp %}
HINSTANCE dll_handle = ::LoadLibraryEx (dll_name, NULL,
                                         LOAD_WITH_ALTERED_SEARCH_PATH);

  // Initialize the crash reporter.
  InitCrashReporter( client_util::GetDLLPath (dll_name, dll_path));

  bool exit_now = true ;
  if ( ShowRestartDialogIfCrashed(&exit_now )) {
    // We have restarted because of a previous crash. The user might
    // decide that he does not want to continue.
    if ( exit_now)
      return ResultCodes ::NORMAL_EXIT;
  }

  if ( NULL != dll_handle ) {
    client_util:: DLL_MAIN entry = reinterpret_cast< client_util::DLL_MAIN >(
        :: GetProcAddress(dll_handle , "ChromeMain"));
    if ( NULL != entry )
      return (entry )(instance, & sandbox_info, command_line );
  }

{% endhighlight %}

4) ChromeMain函数做一些通用组件的初始化工作，之后则根据命令行中的参数选项，要么调用RenderMain，要么调用BrowserMain函数，或者调用WorkMain或者PluginMain函数：  
{% highlight cpp %}
int rv = -1;
  if ( process_type == switches ::kRendererProcess) {
    rv = RendererMain(main_params );
  } else if (process_type == switches:: kPluginProcess) {
#if defined (OS_WIN)
    rv = PluginMain(main_params );
#endif
  } else if (process_type == switches:: kWorkerProcess) {
#if defined (OS_WIN)
    rv = WorkerMain(main_params );
#endif
  } else if (process_type .empty()) {
#if defined (OS_LINUX)
    // gtk_init() can change |argc| and |argv|, but nobody else uses them.
    gtk_init(&argc, const_cast<char***>(&argv));
#endif

    ScopedOleInitializer ole_initializer;
    rv = BrowserMain(main_params );
  } else {
    NOTREACHED() << "Unknown process type";
  }
{% endhighlight %}

5) 这里以BrowserMain函数入口为例（通常我们会进入到这里，表示启动一个Browser application)。BrowserMain函数会做一些Browser相关的初始化工作，然后根据命令行参数，来决定是以什么方式来启动Browser Window。  
a) 根据命令行参数来初始化BrowserProcess对象：  
{% highlight cpp %}
  scoped_ptr< BrowserProcess> browser_process ;
  if ( parsed_command_line.HasSwitch (switches:: kImport)) {
    // We use different BrowserProcess when importing so no GoogleURLTracker is
    // instantiated (as it makes a URLRequest and we don't have an IO thread,
    // see bug #1292702).
    browser_process. reset(new FirstRunBrowserProcess(parsed_command_line ));
    is_first_run = false;
  } else {
    browser_process. reset(new BrowserProcessImpl( parsed_command_line));
  }
{% endhighlight %}
在BrowserProcessImpl的构造函数中，会将一个全局的browserProcess\*指针进行初始化：  
{% highlight cpp %}
  g_browser_process = this;  // line 112 of browser_process_impl.cc
{% endhighlight %}
BrowserProcessImpl继承于BrowserProcess和NonThreadSafe。  
FirstRunBrowserProcess继承于BrowserProcessImpl.

b) 调用BrowserInit::ProcessCommandLine，这个函数调用LaunchBrowser（调用LaunchBrowserImpl函数）去启动，OpenURLsInBrowser （这个函数将会初始化一个Browser实例对象，然后在其上调用AddTabWithURL函数去实例化一些TabContents对象。
之后便调用browser->window()->Show()去显示browser window。最后便进入了UI事件循环系统。
{% highlight cpp %}
  int result_code = ResultCodes ::NORMAL_EXIT;
  if ( parameters .ui_task ) {
    if ( pool ) pool -> Recycle();
    MessageLoopForUI:: current ()->PostTask ( FROM_HERE, parameters .ui_task );
    RunUIMessageLoop( browser_process .get ());
  } else if (BrowserInit :: ProcessCommandLine( parsed_command_line ,
                                             std ::wstring (), local_state, true ,
                                             profile , &result_code )) {
    if ( pool ) pool -> Recycle();
    RunUIMessageLoop( browser_process .get ());
  }
{% endhighlight %}
BrowserInit::LaunchBrowserImpl这个函数做以下事情：
{% highlight cpp %}
  LaunchWithProfile lwp(cur_dir, command_line);
  bool launched = lwp.Launch(profile, process_startup);
{% endhighlight %}
BrowserInit::Launch() 这个函数实际上做以下事情：
{% highlight cpp %}
// Open the required browser windows and tabs.
  // First, see if we're being run as a web application (thin frame window).
  if (!OpenApplicationURL(profile)) {
    std::vector<GURL> urls_to_open = GetURLsFromCommandLine(profile_);
    RecordLaunchModeHistogram(urls_to_open.empty()?
                              LM_TO_BE_DECIDED : LM_WITH_URLS);
    // Always attempt to restore the last session. OpenStartupURLs only opens
    // the home pages if no additional URLs were passed on the command line.
    if (!OpenStartupURLs(process_startup, urls_to_open)) {
      // Add the home page and any special first run URLs.
      Browser* browser = NULL;
      if (urls_to_open.empty())
        AddStartupURLs(&urls_to_open);
      else
        browser = BrowserList::GetLastActive();
      OpenURLsInBrowser(browser, process_startup, urls_to_open);
      // NOTE: 这个函数返回的browser指针并没有被销毁，内存泄漏？
      // 其实并没有，Browser生命期交给了BrowserView来管理
    }
  } else {
    RecordLaunchModeHistogram(LM_AS_WEBAPP);
  }
{% endhighlight %}
// OpenURLsInBrowser 添加Tag with URL到browser windows，然后返回一个Browser实例化对象。
{% highlight cpp %}
   Browser * BrowserInit :: LaunchWithProfile::OpenURLsInBrowser (
    					Browser* browser ,
    					bool process_startup ,
    					const std ::vector < GURL>& urls ) {
  DCHECK(! urls .empty ());
  if (! browser || browser -> type() != Browser ::TYPE_NORMAL )
    browser = Browser ::Create ( profile_);

  for ( size_t i = 0; i < urls .size (); ++ i) {
    TabContents* tab = browser -> AddTabWithURL(
        urls [i ], GURL(), PageTransition ::START_PAGE , ( i == 0), NULL );
    if ( i == 0 && process_startup )
      AddCrashedInfoBarIfNecessary (tab );
  }
  browser-> window ()->Show ();
  // TODO(jcampan): http://crbug.com/8123 we should not need to set the initial
  //                focus explicitly.
  browser-> GetSelectedTabContents ()->SetInitialFocus ();

  return browser ;
}
{% endhighlight %}