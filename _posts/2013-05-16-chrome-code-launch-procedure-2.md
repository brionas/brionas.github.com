---
layout: post
title: "[Chrome源码阅读]Chrome启动代码流程2"
description: TAB URL 启动和navigation初始化
category: "chrome"
tags: [chrome, 源码分析]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/
refer_post_addr: http://zeroli.github.io/chrome/2013/03/25/chrome-code-launch-procedure2/
---
{% include JB/setup %}

**Chrome启动代码流程2：(v2.0版，Windows平台)**

TAB URL 启动和navigation初始化

接着上一篇，关于AddTabWithURL函数。这个函数是在启动Chrome Browser应用程序时调用的，会自动打开一些tabs，用来导航欲加载的URLs。譬如上次运行Chrome时打开并记录的URLs。
这个函数定义在Browser类中：
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
实际干活的是CreateTabeContentsForURL函数，这个函数根据URL来确定URL的类型，然后实例化具体的TabContents，有可能是WebContents(这个最常见），或者HmtlDialogContents，又或者是DOMUIContents。
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
在实例化具体的TabContents的过程中，会通过RenderViewHostManager对象来创建一个RenderViewHost，这个RenderViewHost要么创建一个新的renderer process，要么重用一个存在的renderer process，取决于SiteInstance\*这个参数:   
1) 每个TAB一个进程;  
2) 每个域名一个进程;  
3) 每个网页一个进程;  
4) 每个网页以及从这个网页说打开的其他连接一个进程。  
当然如果SiteInstance\*为空，则直接创建一个新的render process。即使是重用一个render process，在那个render process里之后仍然会创建一个新的render view 对象来对应这个新的render view host。

SetupController函数将会Setup一个新的NavigationController实例(\src\chrome\browser\tab_contents\navigation_controller.h)，用来之后的LoadURL调用。这时TabContents就和NavigationController建立了关系。得注意的是一个NavigationController可以绑定多个Tabs。
{% highlight cpp %}
void NavigationController :: LoadURL( const GURL & url, const GURL & referrer,
                                   PageTransition ::Type transition) {
	// The user initiated a load, we don't need to reload anymore.
	needs_reload_ = false ;
	
	NavigationEntry* entry = CreateNavigationEntry ( url, referrer , transition );
	
	LoadEntry( entry );
}
{% endhighlight %}
创建一个NavigationEntry，然后load。

通过TabContents的controller()函数可以拿到NavigationController实例，这样函数调用将TabContents添加到TabStripModel中进行管理。
{% highlight cpp %}
tabstrip_model_. AddTabContents(contents , -1, transition, foreground);
{% endhighlight %}
TabContents和NavigationController组合成TabContentsData，被TabStripModel管理。TabStripModel里有一个vector变量维护一个TabContentsData列表。

![](/assets/image/1345622242_9385.png)
