---
layout: post
title: "[ChromeԴ���Ķ�]Chrome������������2"
description: ""
category: "chrome"
tags: [chrome, Դ�����]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/categories.html
refer_post_addr: http://zeroli.github.io/chrome/2013/03/25/chrome-code-launch-procedure2/
---
{% include JB/setup %}

**Chrome������������2��(v2.0�棬Windowsƽ̨)**

TAB URL ������navigation��ʼ��

������һƪ������AddTabWithURL���������������������Chrome BrowserӦ�ó���ʱ���õģ����Զ���һЩtabs���������������ص�URLs��Ʃ���ϴ�����Chromeʱ�򿪲���¼��URLs��
�������������Browser���У�
{% highlight cpp %}
TabContents* Browser ::AddTabWithURL(
    const GURL& url , const GURL& referrer , PageTransition:: Type transition ,
    bool foreground, SiteInstance * instance) 
{
  GURL url_to_load = url ;
  if ( url_to_load.is_empty ())
    url_to_load = GetHomePage();
  TabContents* contents =
      CreateTabContentsForURL(url_to_load , referrer, profile_, transition ,
                              false, instance );
  tabstrip_model_. AddTabContents(contents , -1, transition, foreground);
  // By default, content believes it is not hidden.  When adding contents
  // in the background, tell it that it's hidden.
  if (! foreground)
    contents-> WasHidden();
  return contents;
}
{% endhighlight %}
ʵ�ʸɻ����CreateTabeContentsForURL�����������������URL��ȷ��URL�����ͣ�Ȼ��ʵ���������TabContents���п�����WebContents(��������������HmtlDialogContents���ֻ�����DOMUIContents��
{% highlight cpp %}
TabContents* Browser ::CreateTabContentsForURL(
    const GURL& url , const GURL& referrer , Profile* profile,
    PageTransition:: Type transition , bool defer_load,
    SiteInstance* instance) const {
  // Create an appropriate tab contents.
  GURL real_url = url ;
  TabContentsType type = TabContents ::TypeForURL(& real_url);
  DCHECK( type != TAB_CONTENTS_UNKNOWN_TYPE );

  TabContents* contents = TabContents ::CreateWithType( type, profile , instance);
  contents-> SetupController(profile );

  if (! defer_load) {
    // Load the initial URL before adding the new tab contents to the tab strip
    // so that the tab contents has navigation state.
    contents-> controller()->LoadURL (url, referrer, transition );
  }

  return contents;
}
{% endhighlight %}
��ʵ���������TabContents�Ĺ����У���ͨ��RenderViewHostManager����������һ��RenderViewHost�����RenderViewHostҪô����һ���µ�renderer process��Ҫô����һ�����ڵ�renderer process��ȡ����SiteInstance\*�������:   
1) ÿ��TABһ������;  
2) ÿ������һ������;  
3) ÿ����ҳһ������;  
4) ÿ����ҳ�Լ��������ҳ˵�򿪵���������һ�����̡�  
��Ȼ���SiteInstance\*Ϊ�գ���ֱ�Ӵ���һ���µ�render process����ʹ������һ��render process�����Ǹ�render process��֮����Ȼ�ᴴ��һ���µ�render view ��������Ӧ����µ�render view host��

SetupController��������Setupһ���µ�NavigationControllerʵ��(\src\chrome\browser\tab_contents\navigation_controller.h)������֮���LoadURL���á���ʱTabContents�ͺ�NavigationController�����˹�ϵ����ע�����һ��NavigationController���԰󶨶��Tabs��
{% highlight cpp %}
void NavigationController :: LoadURL( const GURL & url, const GURL & referrer,
                                   PageTransition ::Type transition) {
  // The user initiated a load, we don't need to reload anymore.
  needs_reload_ = false ;

  NavigationEntry* entry = CreateNavigationEntry ( url, referrer , transition );

  LoadEntry( entry );
}
{% endhighlight %}
����һ��NavigationEntry��Ȼ��load��

ͨ��TabContents��controller()���������õ�NavigationControllerʵ���������������ý�TabContents��ӵ�TabStripModel�н��й���
{% highlight cpp %}
tabstrip_model_. AddTabContents(contents , -1, transition, foreground);
{% endhighlight %}
TabContents��NavigationController��ϳ�TabContentsData����TabStripModel����TabStripModel����һ��vector����ά��һ��TabContentsData�б�

![](/assets/image/1345622242_9385.png)
