---
layout: post
title : Django学习 - 入门篇
description : 项目中一些工具需要提供web接口，所以抽时间看了看web开发, 好久之前在学校的时候接触过ASP, PHP ，但感觉太过繁琐，要自己设计前台页面和又要处理后台逻辑然后又代码又混在一起。显然，对于我们这种非专业web开发的人员的快速的开发需求，还有需要借组强大web开发框架的。看了下框架很多很多，有基于PHP的，也有基于python的。Diaogo就是这么一个很火的Python web 框架，之前的项目组用到的开源工具Review board, OSQA都是基于Diango开发的，自己也凭着感觉简单改过些源码，看来是时候看看Diango了，那么就从环境搭建开始吧。
category : Python
tags : [Python]
refer_author: Tank
refer_blog_addr:
refer_post_addr:

---
{% include JB/setup %}

##一、环境配置

（1）安装Python:
window比较简单，linux默认已经安装，我这里的版本Python2.7.6,有了Python就可以安装Django了

（2）下载最新版本Diango.1.6.0:
https://www.djangoproject.com/download/

（3）—解压进入目录然后执行:

	$ python setup.py install

（4）检测是否安装成功:

	$ python
	>>> import django
	>>> print django.VERSION
	(1, 6, 0, 'final', 0)
	>>> print django.__file__
	E:\Python27\lib\site-packages\django\__init__.pyc

（5）设置环境变量（可选）:
Django安装成功后默认位于 $PYTHON_INSTLL_HOME/lib/site-packges/django为方便使用Django提供的一些命令行工具，加入$PYTHON_INSTLL_HOME/lib/site-packges/django/bin到$PATH（通过设置环境变量)


##二、Django版Hello world

写个Django 版本的hello world吧

（1）创建一个名字为mysite的Django项目:

	$ E:\toycode\django-project>django-admin.py startproject mysite

Diango会为你创建一个项目的目录，结构如下：

	mysite/
	     manage.py           	#对项目操作的命令
	     mysite/                      
	           __init__.py     	#这个目录是个python包
	          setting.py       	#项目设置文件
	          urls.py           #URL映射管理
	          wsgi.py           #Python应用程序或框架和Web服务器之间的一种接口

（2）利用Diango内置的web服务器快速调试网页:默认开8000端口

	E:\toycode\django-project>cd mysite
	E:\toycode\django-project\mysite>manage.py runserver
	Validating models...

	0 errors found
	November 27, 2013 - 09:54:27
	Django version 1.6, using settings 'mysite.settings'
	Starting development server at http://127.0.0.1:8000/
	Quit the server with CTRL-BREAK.
	[27/Nov/2013 09:55:04] "GET / HTTP/1.1" 200 1757

（3）预览网页:
浏览器中输入 localhost:8000, 你将看到欢迎页面:

	It worked!
	Congratulations on your first Django-powered page.

	Of course, you haven't actually done any work yet. Next, start your first app by running python manage.py startapp [appname].
	You're seeing this message because you have DEBUG = True in your Django settings file and you haven't configured any URLs. Get to work!


（4）增加自己的页面:
新建一个文件 views.py 在 mysite/mysite 目录

	from django.http import HttpResponse

	def hello(request):
	    return HttpResponse("Hello world")


（5）增加URL 映射:
修改文件 mysite/mysite/url.py

	from django.conf.urls import patterns, include, url

	from django.contrib import admin
	admin.autodiscover()
	from mysite.views import hello

	urlpatterns = patterns('',
	    # Examples:
	    # url(r'^$', 'mysite.views.home', name='home'),
	    # url(r'^blog/', include('blog.urls')),

	    url(r'^admin/', include(admin.site.urls)),
	    #url(r'^hello/$', hello),
	)

（6）在浏览器中预览我们自己的页面:

输入localhost/hello, 你将看到:
Hello World!

##三、Django工作流程
由上面的hello world,总结一下， Django工作的流程如下：

* 浏览器发出请求/hello/
* Django由配置文件ROOT_URLCONF选项确定主URL映射配置在url.py
* url.py找到匹配的/hellp/的模式
* 找到一个匹配模式，调用该模式关联的view 函数
* view函数放回一个HttpResponse
* Django把HttpResponse转化为正确的HTTP应答，这个应答就是一个web页面


##四、做个动态网页

hello, world只是个静态的页面,现在返回一个动态的页面，我们返回服务器时间

（1）views.py增加函数current_datetime

	from django.http import HttpResponse
	import datetime

	def current_datetime(request):
	    now = datetime.datetime.now()
	    html = "<html><body>It is now %s.</body></html>" % now
	    return HttpResponse(html)

（2）urls.py增加映射

	from django.conf.urls import patterns, include, url

	from django.contrib import admin
	admin.autodiscover()
	from mysite.views import hello, current_datetime

	urlpatterns = patterns('',
	    # Examples:
	    # url(r'^$', 'mysite.views.home', name='home'),
	    # url(r'^blog/', include('blog.urls')),

	    url(r'^admin/', include(admin.site.urls)),
	    url(r'^hello/$', hello),
	    url(r'^time/$', current_datetime),
	)

(3) 输入localhost/time就可以预览了

	It is now 2013-12-06 20:15:56.661000.


##五、动态url

刚才的/time虽然是个动态网页，但url确是静态，有时我们的url也是动态的，用于传递一些参数,比如product.php? id=203，这个Django做的更加优雅，下面我们在当前时间基础上加一个事件间隔，这个间隔作为参数传递。

（1）views.py增加hours_ahead函数

	def hours_ahead(request, offset):
	    try:
	        offset = int(offset)
	    except ValueError:
	        raise Http404()
	    dt = datetime.datetime.now() + datetime.timedelta(hours=offset)
	    html = "<html><body>In %s hour(s), it will be %s.</body></html>" % (offset, dt)
	    return HttpResponse(html)

（2）urls.py增加映射

	from django.conf.urls import patterns, include, url

	from django.contrib import admin
	admin.autodiscover()
	from mysite.views import hello, current_datetime, hours_ahead

	urlpatterns = patterns('',
	    # Examples:
	    # url(r'^$', 'mysite.views.home', name='home'),
	    # url(r'^blog/', include('blog.urls')),

	    url(r'^admin/', include(admin.site.urls)),
	    url(r'^hello/$', hello),
	    url(r'^time/$', current_datetime),
	    url(r'^time/plus/(\d{1,2})/$', hours_ahead),
	)


（3）预览网页/time/plus/3

这里解释一下，这`\d{1,2}`是匹配一位或两位数，级0~99,括号表示捕获符合条件的字符串，然后传递hour_ahead做为第二个参数offect，注意所有关联的view函数都是以一个HttpRequest作为第一个参数
