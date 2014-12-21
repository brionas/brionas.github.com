---
layout: post
title: "Django学习 - Model"
description : Django 里面的model主要是数据访问的一个抽象层，隔离了具体的数据库操作，提供了高层访问接口，看看这么用吧。
category : Django
tags : [Django, Python]
refer_author: Tank
refer_blog_addr: http://tanklee.github.io
refer_post_addr: http://tanklee.github.io/2014/07/08/django-model/
---

##一、安装数据库适配器
我这里用的MySQL-python-1.2.3.win32-py2.7.exe
这是一个python库，安装后位于，$PYTHON_HOME/lib/site-packages/MySQLdb
这样我们就可以在Python代码里面通过MySQLdb访问MySQL了

    C:\Users\tanli>python
    Python 2.7.3 (default, Apr 10 2012, 23:31:26) [MSC v.1500 32 bit (Intel)] on win
    32
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import MySQLdb
    >>> print MySQLdb.__version__
    1.2.3
    >>> MySQLdb.connect(user='root', db='test', passwd='', host='localhost')
    <_mysql.connection open to 'localhost' at 25c62e8>


##二、设置项目数据库
在项目的配置文件settings.py中指定数据库引擎

    # Database
    # https://docs.djangoproject.com/en/1.6/ref/settings/#databases

    DATABASES = {
        'default': {
            #'ENGINE': 'django.db.backends.sqlite3',
            #'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),

            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'djangodb',
            'USER': 'root',
            'PASSWORD': '',
            'HOST': '127.0.0.1',
            'PORT': '3306',
        }
    }

打开django project python shell测试数据库连接

    F:\toycode\django-project\mysite>python manage.py shell
    Python 2.7.3 (default, Apr 10 2012, 23:31:26) [MSC v.1500 32 bit (Intel)] on win
    32
    Type "help", "copyright", "credits" or "license" for more information.
    (InteractiveConsole)
    >>> from django.db import connection
    >>> cursor = connection.cursor()
    >>>

如果没有显示什么错误信息，那么你的数据库配置是正确的。

##三、为项目创建App
现在已经确认数据库连接正常工作了，让我们来创建一个 Django app,一个包含模型，视图和Django代码
，并且形式为独立Python包的完整Django应用， 这里我们新建一个books应用。

    E:\toycode\django-project\mysite\mysite>manage.py startapp books
    E:\toycode\django-project\mysite\mysite>

    mysite
         mysite
         manage.py
         books
              __init.py__
               admin.py
              models.py
              test.py
              views.py

##四、创建Model类
这里用书籍-作者-出版商为例说明,书籍有书名和出版日期。 它有一个或多个作者
（和作者是多对多的关联关系）， 只有一个出版商（和出版商是一对多的关联关系，也被称作外键）.编辑models.py

    from django.db import models

    # Create your models here.

    class Publisher(models.Model):
        name = models.CharField(max_length=30)
        address = models.CharField(max_length=50)
        city = models.CharField(max_length=60)
        state_province = models.CharField(max_length=30)
        country = models.CharField(max_length=50)
        website = models.URLField()

    class Author(models.Model):
        first_name = models.CharField(max_length=30)
        last_name = models.CharField(max_length=40)
        email = models.EmailField()

    class Book(models.Model):
        title = models.CharField(max_length=100)
        authors = models.ManyToManyField(Author)
        publisher = models.ForeignKey(Publisher)
        publication_date = models.DateField()

每个模型相当于单个数据库表，每个属性也是这个表中的一个字段。 属性名就是字段名，
它的类型（例如 CharField ）相当于数据库的字段类型 （例如 varchar ）。
每个数据库表对应一个类”这条规则的例外情况是多对多关系。 在我们的范例模型中， 
Book 有一个 多对多字段 叫做 authors 。 该字段表明一本书籍有一个或多个作者，
但 Book 数据库表却并没有 authors 字段。 相反，Django创建了一个额外的表（多对多连接表）
来处理书籍和作者之间的映射关系。最后需要注意的是，我们并没有显式地为这些模型定义任何主键。 除非你单独指明，否则Django会自动为每个模型生成一个自增长的整数主键字段id,每个Django模型都要求有单独的主键.


##五、 Model部署到数据库

修改项目配置文件,激活books APP

    INSTALLED_APPS = (
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
        'mysite.books',
    )

现在我们可以创建数据库表了。 首先，用下面的命令验证项目模型的有效性：

    F:\toycode\django-project\mysite>python manage.py validate
    0 errors found

validate 命令检查你的模型的语法和逻辑是否正确。 如果一切正常，你会看到 0 errors found 消息。
如果出错，请检查你输入的模型代码。如果出错，请检查你输入的模型代码。 错误输出会给出非常有用
的错误信息来帮助你修正你的模型。

模型确认没问题了，运行下面的命令来查看Django生成的对应的sql语句

    F:\toycode\django-project\mysite>python manage.py sqlall books
    BEGIN;
    CREATE TABLE `books_publisher` (
        `id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY,
        `name` varchar(30) NOT NULL,
        `address` varchar(50) NOT NULL,
        `city` varchar(60) NOT NULL,
        `state_province` varchar(30) NOT NULL,
        `country` varchar(50) NOT NULL,
        `website` varchar(200) NOT NULL
    )
    ;
    CREATE TABLE `books_author` (
        `id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY,
        `first_name` varchar(30) NOT NULL,
        `last_name` varchar(40) NOT NULL,
        `email` varchar(75) NOT NULL
    )
    ;
    CREATE TABLE `books_book_authors` (
        `id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY,
        `book_id` integer NOT NULL,
        `author_id` integer NOT NULL,
        UNIQUE (`book_id`, `author_id`)
    )
    ;
    ALTER TABLE `books_book_authors` ADD CONSTRAINT `author_id_refs_id_1a0a2829` FOR
    EIGN KEY (`author_id`) REFERENCES `books_author` (`id`);
    CREATE TABLE `books_book` (
        `id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY,
        `title` varchar(100) NOT NULL,
        `publisher_id` integer NOT NULL,
        `publication_date` date NOT NULL
    )
    ;
    ALTER TABLE `books_book` ADD CONSTRAINT `publisher_id_refs_id_974c2a46` FOREIGN
    KEY (`publisher_id`) REFERENCES `books_publisher` (`id`);
    ALTER TABLE `books_book_authors` ADD CONSTRAINT `book_id_refs_id_0a3634f3` FOREI
    GN KEY (`book_id`) REFERENCES `books_book` (`id`);
    CREATE INDEX `books_book_authors_36c249d7` ON `books_book_authors` (`book_id`);
    CREATE INDEX `books_book_authors_e969df21` ON `books_book_authors` (`author_id`)
    ;
    CREATE INDEX `books_book_81b79144` ON `books_book` (`publisher_id`);

    COMMIT;

原来django 为了我们处理好具体的sql操作,现在把这些sql提交到数据库，为model建表

    F:\toycode\django-project\mysite>python manage.py syncdb
    Creating tables ...
    Creating table django_admin_log
    Creating table auth_permission
    Creating table auth_group_permissions
    Creating table auth_group
    Creating table auth_user_groups
    Creating table auth_user_user_permissions
    Creating table auth_user
    Creating table django_content_type
    Creating table django_session
    Creating table books_publisher
    Creating table books_author
    Creating table books_book_authors
    Creating table books_book

    You just installed Django's auth system, which means you don't have any superuse
    rs defined.
    Would you like to create one now? (yes/no): yes
    Username (leave blank to use 'tanli'): admin
    Email address: tank.li@asml.com
    Password:
    Password (again):
    Superuser created successfully.
    Installing custom SQL ...
    Installing indexes ...
    Installed 0 object(s) from 0 fixture(s)

syncdb 命令是同步你的模型到数据库的一个简单方法。 它会根据 INSTALLED_APPS 
里设置的app来检查数据库， 如果表不存在，它就会创建它


##六、访问数据
有了Django 的model，就可以使用这些模型的高级的Python API了，所以的一切都是python 
object上的操作，不再关心数据库中的具体存储，那是框架该做的事。

    F:\toycode\django-project\mysite>python manage.py shell
    Python 2.7.3 (default, Apr 10 2012, 23:31:26) [MSC v.1500 32 bit (Intel)] on win
    32
    Type "help", "copyright", "credits" or "license" for more information.
    (InteractiveConsole)
    >>> from books.models import Publisher
    >>> p1 = Publisher(name='HUST', address='Hubei', city='Wuhan')
    >>> p2 = Publisher(name='ECJTU', address='Jiangxi', city='Nanchang')
    >>> p1.save()
    >>> p2.save()
    >>> print p1
    HUST
    >>> publisher_list = Publisher.objects.all()
    >>> publisher_list
    [<Publisher: HUST>, <Publisher: ECJTU>]

调用对象的 save() 方法，将对象保存到数据库中。 Django 会在后台执行一条 INSERT 语句。
这里publisher_list显示的这么友好是因为，Publisher我们自定义一个__unicode__方法，
用于Publisher对象的输出。除了save, 还有filter, delete等。


##七、创建模板
创建显示出版商的页面模板publisherlist.html


    <ul>
        { % for publisher in publishers % }
            <li> { { publisher.name } } </li>
        { % endfor % }
    </ul>


修改项目配置文件指定自动加载目录

    TEMPLATE_DIRS = (
        'F:/toycode/django-project/mysite/templates',
    )

##八、写视图函数
编辑 books/views.py

    from django.shortcuts import render

    # Create your views here.

    from django.shortcuts import render_to_response
    from books.models import Publisher

    def publisherlist(request):
        list = Publisher.objects.all()
        return render_to_response('publisherlist.html', {'publishers':list})

##九、配置URL

    urlpatterns = patterns('',
        # Examples:
        # url(r'^$', 'mysite.views.home', name='home'),
        # url(r'^blog/', include('blog.urls')),

        url('^publishers/$','books.views.publisherlist'),
    )

至此就是一个数据库驱动的web应用了，localhost:8000/publishers预览
