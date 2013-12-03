---
layout: post
title: Review Board 1.7.12配置（CentOS）
description: ReviewBoard 1.7.12在CentOS上的配置
category: 工具
tags: [ReviewBoard linux CentOS]
refer_author: Xuan
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}




<ul>
<li><p>Review Board简介</p></li>
<li><p>服务器配置</p></li>
<li><p>安装ReviewBot</p></li>
<li><p>客户端配置</p>
<ol><li>添加环境变量</li>
    <li>创建文件.reviewboardrc，以perforce为例</li>
    <li>将reviewboard拷贝到目录</li>
    <li>安装gundiff (Windows)</li>
    <li>使用命令提交代码</li>
    <li>基于memcache的脚本优化</li>
</ol>
</li>
</ul>

## Review Board简介
ReviewBoard是一款基于python django开发的开源代码评审工具。在代码提交到代码库之前加入代码审查，可以有效提高代码质量。

![image](/assets/image/review-board-setup/dashboard.png)

Review Board 1.7和以前版本相比有很大的改进，最大的一个优点是支持ReviewBot。ReviewBot里面自动集成cpplint, cppcheck等静态检查工具， 可以对新提交diff进行静态检查：

![image](/assets/image/review-board-setup/reviewbot.png)

![image](/assets/image/review-board-setup/cppcheck.png)

官方文档对于Reviewboad的[下载](http://www.reviewboard.org/downloads/)和[安装](http://www.reviewboard.org/docs/manual/1.7/admin/installation/linux/)有详细的介绍。

在项目使用过程中， 发现一些较容易出错的地方，以及自定制和提高性能的tips，总结如下：

## 服务器配置
1. 如果提交review评论时没发送邮件，在$reviewboard_addr/admin/log/server/页面检查server日志(其中$reviewboard_addr为reviewboard的网址)
2. 如果没有日志信息，在$reviewboard_addr/admin/settings/logging/页面打开"enablelogging"选项
3. 如果遇到以下错误，在$reviewboard_addr/admin/settings/email/页面将用户名和密码置空
<pre><code>Error sending e-mail notification with subject 'Re: Review Request: PLT-14896 change autogen code for parsing unknown nodes' on behalf of '"liang zhao"' to '"Yun Yuan","liang zhao" ,"Wenzhe Liu": SMTP AUTH extension not supported by server.
Traceback (most recent call last):
  File "c:\python25\lib\site-packages\ReviewBoard-1.5-py2.5.egg\reviewboard\notifications\email.py", line 177, in send_review_mail
    message.send()
  File "c:\python25\lib\site-packages\django-1.2.1-py2.5.egg\django\core\mail\message.py", line 175, in send
    return self.get_connection(fail_silently).send_messages([self])
  File "c:\python25\lib\site-packages\django-1.2.1-py2.5.egg\django\core\mail\backends\smtp.py", line 78, in send_messages
    new_conn_created = self.open()
  File "c:\python25\lib\site-packages\django-1.2.1-py2.5.egg\django\core\mail\backends\smtp.py", line 47, in open
    self.connection.login(self.username, self.password)
  File "C:\Python25\lib\smtplib.py", line 554, in login
    raise SMTPException("SMTP AUTH extension not supported by server.")
SMTPException: SMTP AUTH extension not supported by server.
</code></pre>
4.  如果启动web service时遇到错误:
<pre><code>Your site's extensions media directory isn't properly set up. This directory is where Review Board will store various extension media files.
Your extensions media directory is currently at: /space/work/tflex_builder/reviewboard/htdocs/media/ext
Permission problems
The directory must be writable by the web server. On Linux/Unix/Mac, you can fix this by typing:
    $ sudo chown -R root "($reviewboard)/htdocs/media/"
On Windows, right-click the data directory and change the ownership to root.
</code></pre>
可以使用以下的命令将lighttpd/aphache(而不是root)改为文件所有者
<pre><code>sudo chown -R lighttpd "($reviewboard)/htdocs/media/ext"
</code></pre>
5. reviewboard 使用memcahced缓存加速数据库访问。默认的CACHESIZE比较小， 可以设置文件/etc/sysconfig/memcached文件，将CACHESIZE改大：
<pre><code>PORT="11211" 
USER="memcached" 
MAXCONN="1024" 
CACHESIZE="4096" 
OPTIONS=""
</code></pre>

## 安装ReviewBot
1. 了解和安装ReviewBot，参考：[https://github.com/reviewboard/ReviewBot](https://github.com/reviewboard/ReviewBot)
2. 安装ReviewBot，要先安装一个消息代理。（推荐RabbitMQ，所有设置都使用默认参数）
3. 安装ReviewBot对ReviewBoard的扩展包 
<pre><code>git clone git://github.com/smacleod/ReviewBot.git
cd ReviewBot/extension 
python setup.py install
</code></pre>
安装完成后重启，现在就能使用'Review-Bot-Extension'了，可以在ReviewBoard的管理员页面进行配置 。
4. 安装ReviewBot worker
<pre><code>git clone git://github.com/smacleod/ReviewBot.git 
cd ReviewBot/bot 
python setup.py install
</code></pre>
5. 激活ReviewBot worker,参考ReviewBot,执行命令：
<pre><code>reviewbot worker -b &lt;broker_url&gt;
</code></pre>
RabbitMQ已经创建了一个默认的代理url:'amqp://guest:guest@localhost:5672//'(参考)
<pre><code>cd /reviewboard/log
sudo reviewbot worker -b amqp://guest:guest@localhost:5672// >>reviewbot.log 2>&1
</code></pre>
6. 如果worker不能监听5672端口，这可能是由Apache Qpid引起的(参考)，可以运行命令：
<pre><code>sudo /sbin/service qpidd stop
</code></pre>
7. 使用[disown](http://linux.about.com/library/cmd/blcmdl1_disown.htm)命令分离特定用户的worker进程。因此程序能够保持在后台运行
8. 配置ReviewBot
  打开ReviewBoard -- Admin -- Review-Bot-Extension -- Configure.其中"maximum comments"一定要设置，否则checks不能正确运行
9.使用自定义的cpplint
  由于需要一些自定义的cpplint规范，因此在原始cpplint的基础上做了些修改做为自定义的cpplint。
  自定义的cpplint的输出格式必须与原始的cpplint相同。所以我们自定义的cpplint需要做个小修改：
<pre><code>sys.stderr.write('%s(%s):  %s  \[%s\] \[%d\]\n' % (
           filename, linenum, message, category, confidence))
</code></pre>
  变为：
<pre><code>sys.stderr.write('%s:%s:  %s \[%s\|%s\] \[%d\|%d\]\n' % (
           filename, linenum, message, category, confidence))
</code></pre>
  如果要安装自定义的cpplint，可以用自定义的cpplint代替cpplint python安装包中的cpplint.py，然后用'python install.py'安装该包
  此外，cpplint也可以通过将cpplint.py重命名为cpplint，让它变为可执行文件并放在bin目录中(没测试过)
10. 激活cpplint和Cppcheck
  在ReviewBoard -- Admin -- Review-Bot-Extension -- Database -- Review bot tools

## 客户端配置
###1. 添加环境变量
    将python目录加到PATH环境变量里，e.g: C:\Python27;C:\Python27\Scripts

###2. 创建文件.reviewboardrc， 以perforce为例

<pre><code>REVIEWBOARD_URL="http://&lt;your review board server address&gt;/"
REPOSITORY=“&lt;your perforce server ip&gt;:&lt;port&gt;”
</code></pre>

###3. 将reviewboard拷贝到目录
Win7:

<pre><code>C:\Users\$your_username\AppData\Roaming/*$your_username为你的用户名*/
</code></pre>

XP:

<pre><code>C:\Documents and Settings\$your_username\Application Data/*$your_username为你的用户名*/
</code></pre>

Linux

<pre><code>/home/$your_username/*$your_username为你的用户名*/
</code></pre>


###4. 安装gundiff (Windows)
1.下载安装gundiff

2.在环境变量中添加gundiff的执行路径(e.g. "C:/Program files/GnuWin32/bin")

3.在cmd中执行diff检测是否已正确完成安装

![image](/assets/image/review-board-setup/reviewboard.png)


###5. 使用命令提交代码
1.下载postreview_run.py（首次使用该脚本时必须使用命令行提交代码）

2.在cmd中执行命令行提交代码：

Windows:

<pre><code>python postreview_run.py $changelist_id -d
</code></pre>

Linux:

<pre><code>python2.7 postreview_run.py $changelist_id -d
</code></pre>

3.输入用户名密码登录（如果没注册，请先注册）

![image](/assets/image/review-board-setup/reviewboardaa.png)


###6. 基于memcache的脚本优化
**问题：**

之前使用reviewboard，发现一个头疼的性能问题：reviewer第一次review code时打开网页看diff非常慢。通过查看Log找到了原因： reviewboard会从项目代码服务器下载代码，再结合客户端上传的diff生成网页，项目中的代码服务器在美国，所以生成diff网页的速度就很慢了。

**解决方案：**

由于reviewboard 1.7使用了memcache，会先从memcache读取数据，如果页面已经被缓存则直接从memcache读数据，可以减少diff页面的打开时间，因此，修改了postreview.py，在reviewee提交post-review请求时，就打开diff页面，触发服务器从代码服务器下载代码，并更新memcache缓存。 这样当reviewer稍后审查代码的时候，服务器就从memcache里面取出代码内容而不需要去代码服务器下载了， 速度快了很多。

**脚本改动：**

<pre><code>webbrowser.open_new(review_url)
</code></pre>
改为：

<pre><code>webbrowser.open_new(diff_url) 
</code></pre>