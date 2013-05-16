---
layout: post
title: "[ChromeԴ���Ķ�]Chrome������������1"
description: Chrome�����������̣�(v2.0�棬Windowsƽ̨)
category: "chrome"
tags: [chrome, Դ�����]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/categories.html
refer_post_addr: http://zeroli.github.io/chrome/2013/03/25/chrome-code-launch-procedure/
---
{% include JB/setup %}

**Chrome�����������̣�(v2.0�棬Windowsƽ̨)**

Ӧ�ó����������̣�
1) WinMain����Ϊ��ڵ㣬�������ļ�\chrome\app\chrome_exe_main.cc�ļ��У�λ��chrome_exe������Ŀ�У�  
2) WinMain��ע������ҵ���ǰ�汾����Ŀ¼��Ȼ��װ��chrome.dll�ļ������û�ҵ�����ֱ�Ӵӵ�ǰexeĿ¼����dll�ļ�����װ�ء�  
3) ֱ�Ӵ�chrome.dll���ҵ�����ChromeMain��Ȼ���������ChromeMain����������\chrome\app\chrome_dll_main.cc�ļ��У�λ��chrome_dll������Ŀ�С�
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

4) ChromeMain������һЩͨ������ĳ�ʼ��������֮��������������еĲ���ѡ�Ҫô����RenderMain��Ҫô����BrowserMain���������ߵ���WorkMain����PluginMain������  
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

5) ������BrowserMain�������Ϊ����ͨ�����ǻ���뵽�����ʾ����һ��Browser application)��BrowserMain��������һЩBrowser��صĳ�ʼ��������Ȼ����������в���������������ʲô��ʽ������Browser Window��  
a) ���������в�������ʼ��BrowserProcess����  
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
��BrowserProcessImpl�Ĺ��캯���У��Ὣһ��ȫ�ֵ�browserProcess\*ָ����г�ʼ����  
{% highlight cpp %}
  g_browser_process = this;  // line 112 of browser_process_impl.cc
{% endhighlight %}
BrowserProcessImpl�̳���BrowserProcess��NonThreadSafe��  
FirstRunBrowserProcess�̳���BrowserProcessImpl.

b) ����BrowserInit::ProcessCommandLine�������������LaunchBrowser������LaunchBrowserImpl������ȥ������OpenURLsInBrowser ��������������ʼ��һ��Browserʵ������Ȼ�������ϵ���AddTabWithURL����ȥʵ����һЩTabContents����
֮������browser->window()->Show()ȥ��ʾbrowser window�����������UI�¼�ѭ��ϵͳ��
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
BrowserInit::LaunchBrowserImpl����������������飺
{% highlight cpp %}
  LaunchWithProfile lwp(cur_dir, command_line);
  bool launched = lwp.Launch(profile, process_startup);
{% endhighlight %}
BrowserInit::Launch() �������ʵ�������������飺
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
      // NOTE: ����������ص�browserָ�벢û�б����٣��ڴ�й©��
      // ��ʵ��û�У�Browser�����ڽ�����BrowserView������
    }
  } else {
    RecordLaunchModeHistogram(LM_AS_WEBAPP);
  }
{% endhighlight %}
// OpenURLsInBrowser ���Tag with URL��browser windows��Ȼ�󷵻�һ��Browserʵ��������
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