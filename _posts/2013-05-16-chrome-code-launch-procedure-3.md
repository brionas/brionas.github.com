---
layout: post
title: "[ChromeԴ���Ķ�]Chrome������������3"
description: ��URL bar�н���URL�ĵ�������
category: "chrome"
tags: [chromeԴ���Ķ�, C++]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/categories.html
refer_post_addr: http://zeroli.github.io/chrome/2013/03/25/chrome-code-launch-procedure3/
---
{% include JB/setup %}

**Chrome������������3��(v2.0�棬Windowsƽ̨)**

##��URL bar�н���URL�ĵ�������

������һƪ�������ǽ����û������һ����Ч��URL��ַ����������Chromeʱ�Զ�loadһ��URL��ַ��the navigation controller������뵽NavigationController::LoadURL������LoadURL���յ���LoadEntry������
{% highlight cpp %}
void NavigationController ::LoadEntry( NavigationEntry* entry ) {
  // Handle non-navigational URLs that popup dialogs and such, these should not
  // actually navigate.
  if ( HandleNonNavigationAboutURL(entry ->url()))
    return;   // �����Ƿ����ڴ�й©����Ϊentry��new�����ģ�֮��û�б�ɾ��?

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

NavigateToPendingEntry�ɵ�������ǻ�ȡTabContents��Ȼ���������NavigateToPendingEntry������ע�⣬TabContents��NavigateToPendingEntry�������麯����WebContents����ʵ��������
{% highlight cpp %}
bool WebContents ::NavigateToPendingEntry( bool reload ) {
  NavigationEntry* entry = controller ()->GetPendingEntry();
  // ����NavigationEntry�����¶�Ӧ��render view host����
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
render_manager_.Navigate�������ý��ᴴ��һ���µ�render view�������render view host��Ӧ��
���ȸ���render view host��Ȼ����render view host������render view���������������һ��������render process.
WebContent�̳���RenderViewHostManager::Delegate�࣬ͬ��Ҳ�̳���RenderViewHost::Delegate�ࡣ
������RenderViewHostManager::Navigate�е���delegate_::CreateRenderViewForRenderManager���ǵ���WebContents�ĺ�����

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
// NavigateToEntry����ʵ����һ��ViewMsg_Nagivate_Params�ṹ�壬Ȼ�󴫸�DoNavigate������
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
// ֱ�ӷ��͸�RenderProcess����
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
###1. ʲôʱ�򴴽�RenderViewHost��RenderProcessHost����ô�����ģ�
һ��WebContentsά��һ��RenderViewHostManager render\_manager\_ ;
Ҳ����˵�����κ�һ��WebContents���ͻ�ʵ����һ��RenderViewHostManager��RenderViewHostManagerά����RenderViewHostDelegate, RenderViewHostManager::Delegate, RenderViewHost��ǰ2����TabContents����ʼ��������RenderViewHostManager::init����������һ���µ�RenderViewHost������Դ�ʱ��SiteInstance������˵ÿ��SiteInstance�����Ӧ��һ��RenderViewHostʵ����

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
��ʵ����һ��RenderViewHostʱ������RenderWidgetHost��Ҫһ��RenderProcessHost��ͨ������SiteInstance::GetProcess���룺
{% highlight cpp %}
RenderViewHost::RenderViewHost(SiteInstance* instance,
                               RenderViewHostDelegate* delegate,
                               int routing_id,
                               base::WaitableEvent* modal_dialog_event)
    : RenderWidgetHost(instance->GetProcess(), routing_id),
      instance_(instance),
{% endhighlight %}
SiteInstance::GetProcess��������ȡ���ߴ���һ��RenderProcessHost��������SiteInstance�����뵱ǰ����������ģʽ�����������ã����Ǵ�����
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
������һ��RenderProcessHostʱ����û��ͬʱ����һ��RenderProcess�����Ӧ��

###2. ʲôʱ�򴴽�Render���̣�RenderProcess���󣬺�RenderView����
delegate_::CreateRenderViewForRenderManager����˵��WebContents��������������ΪWebContents��һ��RenderViewHostManager::delegate�������RenderViewHost::CreateRenderView����������һ��Render���̣�����������ڵĻ���
�������ý��̼�ͨ�ţ�������RenderView����
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
 // �����duplicate��ǰ���̵�ʲô���������
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
  // ������ϢҪ�󴴽��µ�Renderview
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
�����process ()->Init����һЩ���顣process()��������һ��RenderProcessHostʵ����Ʃ��BrowserRenderProcessHost������Init�����������ж�channel�Ƿ������ˣ�����������ˣ�����RenderProcess�Ѿ�����������ֻҪshareһ�¾Ϳ����ˣ�����ֱ�ӷ���true���������Ҫʵ��ʵ�Ĵ���һ��RenderProcess��  
RenderProcess����sanbox�д����ģ���˼����˵chrome�������render����ֻ����rendering������network��filesystem��ȫ������browser����������Chrome��design�ĵ�������д����  
> In addition to restricting the renderer's access to the filesystem and network, we can also place limitations on its access to the user's display and related objects. We run each render process on a separate Windows "Desktop" which is not visible to the user. This prevents a compromised renderer from opening new windows or capturing keystrokes.

��sanbox�д���render process�Ĺ����൱���ӣ�λ��BrowserRenderProcessHost::Init()�����С����������sandbox�д������̣���ֻ��Ҫ�򵥵ĵ���win32API CreateProcess��Ȼ���¼���Ľ���Handle�Ϳ����ˡ�

