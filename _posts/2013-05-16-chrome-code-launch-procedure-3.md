---
layout: post
title: "[Chrome源码阅读]Chrome启动代码流程3"
description: 从URL bar中进行URL的导航过程
category: "chrome"
tags: [chrome源码阅读, C++]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/categories.html
refer_post_addr: http://zeroli.github.io/chrome/2013/03/25/chrome-code-launch-procedure3/
---
{% include JB/setup %}

**Chrome启动代码流程3：(v2.0版，Windows平台)**

##从URL bar中进行URL的导航过程

接着上一篇，不管是接受用户输入的一个有效的URL地址，还是启动Chrome时自动load一个URL地址，the navigation controller都会进入到NavigationController::LoadURL函数。LoadURL最终调用LoadEntry函数。
{% highlight cpp %}
void NavigationController ::LoadEntry( NavigationEntry* entry ) {
  // Handle non-navigational URLs that popup dialogs and such, these should not
  // actually navigate.
  if ( HandleNonNavigationAboutURL(entry ->url()))
    return;   // 这里是否有内存泄漏，因为entry是new出来的，之后并没有被删除?

  // When navigating to a new page, we don't know for sure if we will actually
  // end up leaving the current page.  The new page load could for example
  // result in a download or a 'no content' response (e.g., a mailto: URL).
  DiscardNonCommittedEntriesInternal();
  pending_entry_ = entry;
  NotificationService:: current()->Notify (
      NotificationType::NAV_ENTRY_PENDING ,
      Source<NavigationController >(this),
      NotificationService::NoDetails ());
  NavigateToPendingEntry( false);
}
{% endhighlight %}

NavigateToPendingEntry干的事情就是获取TabContents，然后调用它的NavigateToPendingEntry函数。注意，TabContents的NavigateToPendingEntry函数是虚函数，WebContents重新实现了它。
{% highlight cpp %}
bool WebContents ::NavigateToPendingEntry( bool reload ) {
  NavigationEntry* entry = controller ()->GetPendingEntry();
  // 根据NavigationEntry来更新对应的render view host对象。
  RenderViewHost* dest_render_view_host = render_manager_.Navigate (*entry);
  if (! dest_render_view_host)
    return false;  // Unable to create the desired render view host.

  // Used for page load time metrics.
  current_load_start_ = TimeTicks::Now ();

  // Navigate in the desired RenderViewHost.
  dest_render_view_host-> NavigateToEntry(*entry , reload);

  if ( entry->page_id () == -1) {
    // HACK!!  This code suppresses javascript: URLs from being added to
    // session history, which is what we want to do for javascript: URLs that
    // do not generate content.  What we really need is a message from the
    // renderer telling us that a new page was not created.  The same message
    // could be used for mailto: URLs and the like.
    if ( entry->url ().SchemeIs( chrome::kJavaScriptScheme ))
      return false ;
  }

  // Clear any provisional password saves - this stops password infobars
  // showing up on pages the user navigates to while the right page is
  // loading.
  GetPasswordManager()-> ClearProvisionalSave();

  if ( reload && !profile ()->IsOffTheRecord()) {
    HistoryService* history =
        profile()->GetHistoryService (Profile:: IMPLICIT_ACCESS);
    if ( history)
      history->SetFavIconOutOfDateForPage (entry-> url());
  }

  return true;
}
{% endhighlight %}
render_manager_.Navigate函数调用将会创建一个新的render view来与这个render view host对应。
首先更新render view host，然后用render view host来创建render view对象，这个对象处于另一个进程中render process.
WebContent继承了RenderViewHostManager::Delegate类，同是也继承了RenderViewHost::Delegate类。
所以在RenderViewHostManager::Navigate中调用delegate_::CreateRenderViewForRenderManager就是调用WebContents的函数。

![](/assets/image/1345024986_9162.PNG)
{% highlight cpp %}
bool WebContents :: CreateRenderViewForRenderManager(
    RenderViewHost* render_view_host ) {
  RenderWidgetHostView* rwh_view = view_ ->CreateViewForWidget ( render_view_host);
  if (! render_view_host ->CreateRenderView ())
    return false ;

  // Now that the RenderView has been created, we need to tell it its size.
  rwh_view-> SetSize (view_ -> GetContainerSize());

  UpdateMaxPageIDIfNecessary( render_view_host ->site_instance (),
                             render_view_host );
  return true ;
}
{% endhighlight %}
// NavigateToEntry将会实例化一个ViewMsg_Nagivate_Params结构体，然后传给DoNavigate函数。
{% highlight cpp %}
void RenderViewHost ::NavigateToEntry( const NavigationEntry & entry,
                                     bool is_reload ) {
  ViewMsg_Navigate_Params params;
  MakeNavigateParams( entry, is_reload , ?ms);

  RendererSecurityPolicy:: GetInstance()->GrantRequestURL (
      process()->host_id (), params. url);

  DoNavigate( new ViewMsg_Navigate (routing_id(), params));
}
{% endhighlight %}
// 直接发送给RenderProcess进程
{% highlight cpp %}
void RenderViewHost ::DoNavigate( ViewMsg_Navigate* nav_message ) {
  // Only send the message if we aren't suspended at the start of a cross-site
  // request.
  if ( navigations_suspended_) {
    // Shouldn't be possible to have a second navigation while suspended, since
    // navigations will only be suspended during a cross-site request.  If a
    // second navigation occurs, WebContents will cancel this pending RVH
    // create a new pending RVH.
    DCHECK(! suspended_nav_message_.get ());
    suspended_nav_message_. reset(nav_message );
  } else {
    Send( nav_message);
  }
}
{% endhighlight %}

##FAQ:
###1. 什么时候创建RenderViewHost和RenderProcessHost，怎么创建的？
一个WebContents维护一个RenderViewHostManager render\_manager\_ ;
也就是说创建任何一个WebContents，就会实例化一个RenderViewHostManager，RenderViewHostManager维护着RenderViewHostDelegate, RenderViewHostManager::Delegate, RenderViewHost，前2者用TabContents来初始化。调用RenderViewHostManager::init函数来创建一个新的RenderViewHost对象，针对此时的SiteInstance。就是说每个SiteInstance都会对应着一个RenderViewHost实例。

{% highlight cpp %}
void RenderViewHostManager :: Init( Profile * profile ,
                                 SiteInstance * site_instance ,
                                 int routing_id ,
                                 base ::WaitableEvent * modal_dialog_event) {
  // Create a RenderViewHost, once we have an instance.  It is important to
  // immediately give this SiteInstance to a RenderViewHost so that it is
  // ref counted.
  if (! site_instance )
    site_instance = SiteInstance ::CreateSiteInstance ( profile);
  render_view_host_ = CreateRenderViewHost (
      site_instance , routing_id , modal_dialog_event);
}
RenderViewHost * RenderViewHostManager :: CreateRenderViewHost(
    SiteInstance* instance ,
    int routing_id ,
    base:: WaitableEvent * modal_dialog_event ) {
  if ( render_view_factory_ ) {
    return render_view_factory_ ->CreateRenderViewHost (
        instance , render_view_delegate_ , routing_id, modal_dialog_event );
  } else {
    return new RenderViewHost ( instance, render_view_delegate_ , routing_id ,
                              modal_dialog_event );
  }
}
{% endhighlight %}
在实例化一个RenderViewHost时，父类RenderWidgetHost需要一个RenderProcessHost，通过调用SiteInstance::GetProcess传入：
{% highlight cpp %}
RenderViewHost::RenderViewHost(SiteInstance* instance,
                               RenderViewHostDelegate* delegate,
                               int routing_id,
                               base::WaitableEvent* modal_dialog_event)
    : RenderWidgetHost(instance->GetProcess(), routing_id),
      instance_(instance),
{% endhighlight %}
SiteInstance::GetProcess，用来获取或者创建一个RenderProcessHost。它根据SiteInstance类型与当前的启动进程模式来决定是重用，还是创建。
{% highlight cpp %}
RenderProcessHost * SiteInstance :: GetProcess() {
  RenderProcessHost* process = NULL ;
  if ( process_host_id_ != -1)
    process = RenderProcessHost ::FromID ( process_host_id_);

  // Create a new process if ours went away or was reused.
  if (! process ) {
    // See if we should reuse an old process
    if ( RenderProcessHost ::ShouldTryToUseExistingProcessHost ())
      process = RenderProcessHost :: GetExistingProcessHost(
          browsing_instance_ ->profile ());

    // Otherwise (or if that fails), create a new one.
    if (! process ) {
      if (render_process_host_factory_ ) {
        process = render_process_host_factory_ -> CreateRenderProcessHost(
            browsing_instance_ ->profile ());
      } else {
        process = new BrowserRenderProcessHost (browsing_instance_ -> profile());
      }
    }

    // Update our host ID, so all pages in this SiteInstance will use
    // the correct process.
    process_host_id_ = process ->host_id ();

    // Make sure the process starts at the right max_page_id
    process-> UpdateMaxPageID (max_page_id_ );
  }
  DCHECK( process );

  return process ;
}
{% endhighlight %}
当创建一个RenderProcessHost时，并没有同时创建一个RenderProcess与其对应。

###2. 什么时候创建Render进程，RenderProcess对象，和RenderView对象？
delegate_::CreateRenderViewForRenderManager或者说是WebContents类的这个函数（因为WebContents是一种RenderViewHostManager::delegate）会调用RenderViewHost::CreateRenderView函数，创建一个Render进程，如果还不存在的话。
进而借用进程间通信，来创建RenderView对象。
{% highlight cpp %}
bool RenderViewHost :: CreateRenderView() {
  DCHECK(! IsRenderViewLive ()) << "Creating view twice" ;

  // The process may (if we're sharing a process with another host that already
  // initialized it) or may not (we have our own process or the old process
  // crashed) have been initialized. Calling Init multiple times will be
  // ignored, so this is safe.
  if (! process ()->Init ())
    return false ;
  DCHECK( process ()->channel ());
  DCHECK( process ()->profile ());

  renderer_initialized_ = true ;

#if defined ( OS_WIN)
  HANDLE modal_dialog_event_handle ;
  HANDLE renderer_process_handle = process ()-> process(). handle ();
  if ( renderer_process_handle == NULL )
    renderer_process_handle = GetCurrentProcess ();
 // 这个是duplicate当前进程的什么句柄？？？
  BOOL result = DuplicateHandle ( GetCurrentProcess(),
      modal_dialog_event_ ->handle (),
      renderer_process_handle ,
      & modal_dialog_event_handle ,
      SYNCHRONIZE ,
      FALSE ,
      0);
  DCHECK( result ) <<
      "Couldn't duplicate the modal dialog handle for the renderer." ;
#endif

  DCHECK( view ());

  ModalDialogEvent modal_dialog_event ;
#if defined ( OS_WIN)
  modal_dialog_event. event = modal_dialog_event_handle ;
#endif
  // 发送消息要求创建新的Renderview
  Send( new ViewMsg_New (gfx :: IdFromNativeView( view ()->GetPluginNativeView ()),
                       modal_dialog_event ,
                       delegate_ ->GetWebkitPrefs (),
                       routing_id ()));

  // Set the alternate error page, which is profile specific, in the renderer.
  GURL url = delegate_ -> GetAlternateErrorPageURL();
  SetAlternateErrorPageURL( url );

  // If it's enabled, tell the renderer to set up the Javascript bindings for
  // sending messages back to the browser.
  Send( new ViewMsg_AllowBindings ( routing_id(), enabled_bindings_ ));

  // Let our delegate know that we created a RenderView.
  delegate_-> RenderViewCreated (this );

  return true ;
}
{% endhighlight %}
上面的process ()->Init做了一些事情。process()函数返回一个RenderProcessHost实例，譬如BrowserRenderProcessHost。它的Init函数会首先判断channel是否建立好了，如果建立好了，代表RenderProcess已经创建，我们只要share一下就可以了，函数直接返回true。否则就需要实打实的创建一个RenderProcess。  
RenderProcess是在sanbox中创建的，意思就是说chrome限制这个render进程只负责rendering，访问network和filesystem，全部交给browser进程来做。Chrome的design文档中这样写道：  
> In addition to restricting the renderer's access to the filesystem and network, we can also place limitations on its access to the user's display and related objects. We run each render process on a separate Windows "Desktop" which is not visible to the user. This prevents a compromised renderer from opening new windows or capturing keystrokes.

在sanbox中创建render process的过程相当复杂，位于BrowserRenderProcessHost::Init()函数中。如果不是在sandbox中创建进程，则只需要简单的调用win32API CreateProcess，然后记录它的进程Handle就可以了。

