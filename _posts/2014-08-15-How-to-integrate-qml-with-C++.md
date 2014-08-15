---
layout: post
title: QML与C++的交互
description: QML与C++的交互
category: C++
tags: QML
refer_author: Bo-Tao
refer_blog_addr:
refer_post_addr:
---
介绍
===
QML 是一种基于Javascript的描述性脚本语言。它的功能跟xrc文件类似，主要用来描述应用程序主界面。
所不同的是，它可以很方便的跟C++进行交互。qml 跟C++ 的交互方式主要有以下几种：

- 可以直接在C++应用程序中加载qml文件，拿到界面各元素的指针，修改界面属性。这种方式跟xrc文件类似
- 可以将C++对象expose 到qml中，然后在qml文件中访问该对象的属性或调用对象的方法
- 可以自定义C++类，把该类注册给qml类型系统，然后可以像其他内置类型那样在qml中使用

如果充分利用qml的这些特性将qml与C++结合在一起使用可以给我们带来很多帮助，尤其对于提高代码质量改善现有设计。
具体表现在： 

- 界面与逻辑完美分离：用QML来定义界面，用C++来实现界面的响应逻辑。
- 通常的做法是，当用户在界面上操作的时候，我们从qml文件里面调用C++的响应函数。
- 应用MVC方便：用QML来描绘界面(View)，用C++代码来实现Model 和 Controller。
- 自定义控件容易：可以用C++来自定义自己的QML类型， 然后将它应用于我们的应用程序中。
- 可以在程序运行时访问或修改QML对象的属性进而改变界面的显示

在C++代码中访问或修改QML对象
===

加载qml文件应用程序中
---

我们通常都是使用qml定义自己的界面， 然后在C++应用程序中加载qml文件创建用户界面。
我们可以用QQmlComponent或者QQuickView来加载一个QML文件，加载成功之后就可以将qml文件对应的界面显示出来
下面的例子演示了怎样通过qml文件创建一个用户界面程序：
{% highlight cpp %}
int main(int argc, char **argv)
{
    Application app(argc, argv);
    QQuickView view(QUrl::fromlocalFile("main.qml"));// load the qml file
    view.show();
    return app.exec();
}
{% endhighlight %}
对应的界面效果如下：

<img width="480px" src="/assets/image/2014-08/qmlcxx.png" type="image/png" />

访问QML中的子对象
---

加载QML文件之后,它会返回一个C++对象的指针，我们可以通过这个指针来修改QML的各种属性值。
具体的说就是可以通过QObject::setProperty函数来修改QML中控件的各种属性进而改变界面的显示效果。
下面的例子演示了怎样在C++代码中修改QML中的各种属性值来改变界面：
{% highlight cpp %}
// Cpp code
QQuickView view;
view.setSource(QUrl::fromLocalFile("main.qml"));
 
QObject *object = view.rootObject();
object->setProperty("hight", 500);
object->setProperty("width", 500);
 
QObject *rect = object->findChild<QObject*>("rect");
if (rect) btn->setProperty("color", "blue");
view.show();
 
// main.qml
Item {
     width: 200; height:200
     Rectangle {
         color: "green"
         width: 100; height: 100
         objectName: "rect"
         Rectangle {
             color: "blue"
             x: 100; y: 100; width: 100; height: 100
         }
     }
}
{% endhighlight %}
实际上我们可以根据QML中每个控件的objectName(QML中每个控件都可以有一个唯一的标识类似XRC中的name)
通过函数findChild来拿到QML中各子控件的对象指针进而访问或修改控件，但是我们不建议这样做。因为使用QML的初衷
是将用户接口UI的实现与C++逻辑分离开来，如果将界面跟逻辑混杂在一起，失去了使用QML的意义了。所以C++的实现
应该尽可能少的知道QML用户界面的实现以及QML对象树的结构(QML中的各个控件是以树状结构组织起来的，每个控件
都可以包含子控件，每个子控件又可以有各自的子控件)。

在QML中访问C++中的对象并调用C++中的函数
===
当加载QML到C++应用程序中的时候，如果能够直接将C++中的数据嵌入到QML中将会非常的有用。
比如，我们可以在QML文件中通过被插入的C++对象来调用C++中的方法。或者用一个C++的对象
作为QML界面的数据模型。实际上，任何继承自QObject的对象都可以在运行时expose到QML中
供QML 代码调用，包括对象各种属性，方法，信号以及槽函数等。C++中的数据和函数不需要做
任何的修改就可以被QML代码直接访问。Qt 中提供了一系列的方法把C++中的数据注入到QML中使用。

用户想要在QML中访问C++ 中对象，方法如下：

- 被访问的对象的必须继承自QObject
- 在类的定义中添加Q_PROPERTY宏，它的作用是将类的某成员变量expose给QML使用。
- 它同时定义了变量的读函数，写函数以及变量值改变时发出的信号
- 在类的定义中添加Q_INVOKABLE宏，它的作用是将类的某成员函数expose给QML使用。槽函数不用添加Q_INVOKABLE宏，默认就可以在QML中调用 
- 在C++代码中，通过函数QQmlContext::setContextProperty()将对象注入QML

下面的例子演示了怎样把一个C++ 中的对象暴露给qml然后在qml通过该对象中调用C++中的方法。
在下面的qml文件中，cppobj是我们注入的C++对象。我们通过该对象访问C++中的数据调用C++中的函数
{% highlight cpp %}
// main.qml
Item {
    id: buttonrow
    width: 250; height: 50
    RowLayout {    
        TextField {
            id: txtvalue1
            text: cppobj.value1 // get the value by MyCppObject::value1
        }  
        TextField {
            id: txtvalue2
            text: cppobj.value2 // the value by MyCppObject::value2
        }  
        Button {
            id:button1
            text: cppobj.value1 + cppobj.value2
            onClicked: {
                cppobj.resetAllValues(); // call the Cpp function to reset values, then the TextField value will be update automatolly
            }
        }
    }
}
{% endhighlight %}
为了能够让上述qml代码工作， 我们需要在C++代码中实现MyCppObject类并把它暴露给QML，对应的C++代码如下：
{% highlight cpp %}
// cpp code
int main(int argc, char* argv[])
{
     QApplication app(argc, argv);
     QQmlEngine engine;
     MyCppObject cppobj("value1", "value2");
     // export the cppobj to qml and in the qml file, we can access the object by the name "cppobj"
     engine.rootContext()->setContextProperty("cppobj", &cppobj);
     QQuickView view(&engine, NULL);
     view.setSource(QUrl::fromLocalFile("main.qml"));
     view.show();
     return app.exec();
}
 
// MyCppObject implement
class MyCppObject : public QObject
{
    Q_OBJECT
    Q_PROPERTY(QString value1 READ value1 WRITE setValue1 NOTIFY value1Changed )
    Q_PROPERTY(QString value2 READ value2 WRITE setValue2 NOTIFY value2Changed )
public:
    MyCppObject(const QString& value1, const QString& value2): QObject(), m_value1(value1), m_value2(value2) {  }
    Q_INVOKABLE void resetAllValues();
     
    void setValue1(const QString& v);
     
    QString value1() const;// { return m_value1; }
    void setValue2(const QString& v);
     
    QString value2() const;// {return m_value2; }
signals:
    void value1Changed(const QString&);
    void value2Changed(const QString&);
private:
    QString m_value1;
    QString m_value2;
};
void MyCppObject::resetAllValues()
{
    m_value1 = "";
    m_value2 = "";
    emit value1Changed(m_value1);
    emit value2Changed(m_value2);
}
void MyCppObject::setValue1(const QString& v)
{
    if (m_value1 != v)
    {
        m_value1 = v;
        emit value1Changed(m_value1);
    }
}
QString MyCppObject::value1() const { return m_value1; }
void MyCppObject::setValue2(const QString& v)
{
    if (m_value2 != v)
    {
        m_value2 = v;
        emit value2Changed(m_value2);
    }
}
QString MyCppObject::value2() const {return m_value2; }
{% endhighlight %}
把C++类导入QML中使用
===
上面讲的是怎样把一个C++对象注入到qml中使用，其实我们也可以直接在qml文件中使用C++中定义的类型。
比如，用C++自定义一个类型，然后把自定义的类型注册给QML类型系统。这样自定义类型就可以像其它内置类型那样在qml文件中被使用。
通过这种方法，我们可以用C++ 来拓展现有的QML 类型系统，自定义自己的类型并把它整合到已有的QML代码里面。 

用户想要自定义自己的类型，方法如下:

- 在C++中，实现一个直接或者间接派生自QObject 类
- 在C++中，通过函数qmlRegisterType将该自定义类型注册到QML 类型系统中
- 在QML中，导入含有自定义类型的C++模块
- 在QML中，像其他内置类型那样使用自定义的类型

下面的例子，在C++中自定义一个MyTimer 类型再把MyTimer类型注册给QML类型系统。然后可以像其他内置类型那样在qml文件中使用MyTimer类。
使用的时候，我们只需要导入包含MyTimer的对应的模块名称及其版本号"import CustomComponents 1.0"。模块的名称及版本号是在C++代码中我们注册MyTimer类给QML类型系统是指定的。
{% highlight cpp %}
import QtQuick 2.0
import CustomComponents 1.0 // 这里指定MyTimer所属的模块名称及版本号
Item {
     width: 200; height:200
     Rectangle {
         id: rect1
         color: "green"
         width: 100; height: 100
         Rectangle {
             id: rect2
             color: "blue"
             x: 100; y: 100; width: 100; height: 100
             MyTimer { // 这里MyTimer是在C++中定义的一个类
                id: timer
                interval: 1000 // MyTimer类有m_interval这个属性
                onTimeout: {
                    rect1.color = "blue"
                    rect2.color = "green"
                }
            }   
         }
     }
}
{% endhighlight %}
为了使上述的qml 代码工作， 我们需要在C++中定义MyTimer类，并把MyTimer注册个QML类型系统同时指定MyTimer所属的模块及版本号。
下面的C++代码将MyTimer 类注册给QML类型系统
{% highlight cpp %}
int main(int argc, char *argv[])
{
    QApplication app(argc, argv);
    // 这里指定MyTimer所属的模块名称及版本号，必须与qml中保持一致
    qmlRegisterType<MyTimer>("CustomComponents", 1, 0, "MyTimer");
     
    QQuickView view;
    view.setSource(QUrl::fromLocalFile("main.qml"));
    view.show();
    return app.exec();
}
 
// MyTimer.h
class MyTimer : public QObject
{
    Q_OBJECT
    Q_PROPERTY(int interval READ interval WRITE setInterval NOTIFY intervalChanged)
public:
    MyTimer(QObject * p = NULL);
    void setInterval(int msec);
    int interval();
     
signals:
    void intervalChanged();
    void timeout();
    void activeChanged();
     
public slots:
    void start();
    void stop();
private:
    QTimer* m_timer;
    int m_interval;
};
{% endhighlight %}
总结
===
以上简单介绍了qml与C++的交互的方式，主要包括以下几种：

- 在C++中，加载qml文件
- 在qml中，访问C++对象并调用C++方法
- 在qml中，使用导入的C++ 类型

关于QML 与C++的交互，如果想进一步了解的话可以参照Qt官方文档。


