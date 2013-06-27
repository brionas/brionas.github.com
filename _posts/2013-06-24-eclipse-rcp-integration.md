---
layout: post
title: "在Eclipse RCP中集成HTML/CSS/Javascript"
description: Integrate HTML/CSS/Javascript in Eclipse RCP
category: "eclipse"
tags: [eclipse]
refer_author: Yongqiang
refer_blog_addr: http://www.bloodlee.com/
refer_post_addr: http://www.bloodlee.com/wordpress/?p=624
---
{% include JB/setup %}

 
 新项目对UI的要求很高，产品负责人常常要求实现比较Fancy的功能。基于Eclipse RCP平台，用SWT/JFace实现起来比较麻烦，而且组里人手也不够，逼得我们常常要找替代方案。

Eclipse RCP虽然在Look & Feel上有所提高，但它的强项且不是前端，而是在于代码组织（模块，服务），功能扩展等方面。但如果能在RCP环境中集成Ｗeb前端的技术(HTML/CSS/Javascript)，那实现这些Fancy的功能就容易的多。顺着这个思路我做了些实践，做了一个小页面，可以选择本地图片，再做PS处理

![](http://www.bloodlee.com/wordpress/wp-content/uploads/2013/06/webkit_sample.png)


##基本方案 – 使用Browser控件

SWT支持Browser控件，到3.7M2之后开始支持各平台下WebKit。各平台对WebKit要求不同，要参考SWT FAQ。我的体会是Mac（Safari），Linux平台（GTK+ WebKit）很好，Windows比较差。此控件也支持MOZILLA（比较旧的版本），但效果很差，很多CSS库无法运行。（本文多在Linux下WebKit的实践。）

首先要嵌入Browser，
{% highlight cpp %}
Browser b = new Browser(parent, SWT.WEBKIT);
b.setUrl("http://www.google.com");
{% endhighlight %}

之后，Browser就会显示Google主页的内容。

##本地的HTML/CSS/JS文件

当然我们不是要做真正的浏览器，目标是使用本地的文件。实现思路也很清楚，就是要访问RCP Bundle里的文件。RCP也是支持的，可以用如下代码。

假设有如下目录结构：
![](http://www.bloodlee.com/wordpress/wp-content/uploads/2013/06/web_folder.png)

{% highlight cpp %}
// 访问web/hello.html
URL url = Activator.getDefault().getBundle().getResource("web/hello.html");
b.setUrl(FileLocator.toFileURL(url).toString()); // try..catch省略
{% endhighlight %}

但这里要注意一个问题，我们的HTML常常会用到JS或CSS等外部文件。如果这些文件全都打包在Jar文件里，HTML文件内容可以被正常访问，但CSS等就无法被引用了。

解决的方法也很简单，就是不打包这部分文件，可以使用Feature加Fragment的方法。


![](http://www.bloodlee.com/wordpress/wp-content/uploads/2013/06/code_structure.png)
webkit_test是Host Plugin, webkit_test_webfiles是它的Fragment，webkit_test_feature是包括这两个Plugin的Feature。在Feature的设置中，打开“Unpack这个Fragment在安装时”。这样就可以得到一个Folder而不是Jar。

##如何和Java代码互通呢？

这是个大问题。用了HTML，那我们要不要写个Server或者某个嵌入式的Server呢？如果需要，方案就会比较复杂，毕竟RCP还是专注于桌面端的（虽然有RAP，那是后话）。但当我发现BrowserFunction后，这个问题就简单了。

假如，HTML想通过一个函数得到RGB中红、绿、蓝的值，这些值却是在SWT的Text中输入的。函数分别是r(), g(), b().

{% highlight cpp %}
var circle = two.makeCircle(-70, 0, 50);
var rect = two.makeRectangle(70, 0, 100, 100);
circle.fill = '#FF8000';
rect.fill = 'rgb(' + r() + ', ' + g() + ', ' + b() + ')'; // 'rgba(0, 200, 255, 0.75)';
 
var group = two.makeGroup(circle, rect);
group.translation.set(two.width / 2, two.height / 2);
group.scale = 0;
group.noStroke();
{% endhighlight %}

在RCP的Java代码中，
{% highlight cpp %}
new BrowserFunction(b, "r") {
 
  @Override
  public Object function(Object[] arguments) {
    return text_r.getText();
  }
 
};
 
new BrowserFunction(b, "g") {
 
  @Override
  public Object function(Object[] arguments) {
    return text_g.getText();
  }
 
};
 
new BrowserFunction(b, "b") {
 
  @Override
  public Object function(Object[] arguments) {
    return text_b.getText();
  }
 
};
{% endhighlight %}

BrowserFunction构造的第一个入参是Browser Instance，第二个是函数名。这可以大大扩展JS的能力。

那SWT可以控制Brower吗？比如调用JS里的Function。答案也是可以的。首先Browser提供了很多的Listener，让SWT捕获各种事件，此外可以直接用Browse.excute()来执行JS Function。

比如JS中定义了draw()函数，Java代码就可以用如下代码来调用 。

{% highlight cpp %}
b.execute("draw()");
{% endhighlight %}

##总结

在Sample中，我集成了Two.js，jquery, jquery-ui和AlloyImage.js, 实现了一些图片选择处理的功能，Webkit都能工作良好，让我惊艳，也让我坚信这个方向探索的潜力。但也遇到了不少问题，如Resize时页面的刷新问题，如何调试HTML页面，跨平台问题等，还需要更多时间的探索。Sample的代码在GitHub上，在Ubuntu 13.04， Mac OS X 10.8上都可以工作良好。