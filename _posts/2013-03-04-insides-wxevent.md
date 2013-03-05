---
layout: post
title : Wxwidget源码分析-事件机制
description : 通过结合源代码分析Wx事件分发、事件响应实现细节，并分析wx实现跨平台
category : Wxwidgets库
tags : [Wxwidgets库, 源码学习]
---
{% include JB/setup %}

##前言
事件机制是进行GUI编程的一个重点，带着对`wxevent`如下疑问， 分析下源代码吧

* Q1：为什么wx中有的事件能像上传递有的则不能？
* Q2：编程时候怎么使用事件处理机制，如果一个窗口类要处理事件应该做什么？
* Q3：wx中事件机制怎么实现，wx是怎样处理事件，怎么使用事件表？
* Q4:  wx程序中消息是怎么分发的？
* Q5:  MS windows中消息是怎么样转换成wxevent
* Q6:  各种的`wxEvent`来自哪， 由谁发送而来？

##wx中的事件类
wxEvent是wx所有事件类的基类，wx中所有的事件都直接或间接继续于它。它的构造函数如下


    wxEvent(int winid = 0, wxEventType commandType = wxEVT_NULL )

它是一个抽象类， 有一个纯虚函数（虚clone函数，相当于工厂方法，有子类分别生成相应的对象）
所有的可用事件类（包括自定义事件）必须手动实现这个函数，否这`wxEvent`子类对象没法复制一份
传递给`wxPostEvent`

    // this function is used to create a copy of the event polymorphically and
    // all derived classes must implement it because otherwise wxPostEvent()
    // for them wouldn't work (it needs to do a copy of the event)
    virtual wxEvent *Clone() const = 0;

`wxEvent`有个保护成员

    protected:
        // the propagation level: while it is positive, we propagate the event to
        // the parent window (if any)
        //
        // this one doesn't have to be public, we don't have to worry about
        // backwards compatibility as it is new
        int               m_propagationLevel;

`m_propagationLevel`实际上是个整形，其取值系统定义两个常量

    // the predefined constants for the number of times we propagate event
    // upwards window child-parent chain
    enum Propagation_state
    {
        // don't propagate it at all
        wxEVENT_PROPAGATE_NONE = 0,

        // propagate it until it is processed
        wxEVENT_PROPAGATE_MAX = INT_MAX
    };

根据这个值，wx系统中所有事件可以分为两类：可传递的，和不可传递的。可传递的事件继承于`wxCommandEvent`，不可传递的事件直接继承于wxEvent，不可传递事件不会传给事件源窗口的父窗口，即只对当前窗口有效
wxActivate, wxCloseEvent, wxEraseEvent, wxFocusEvent, wxKeyEvent, wxIdleEvent, wxInitDialogEvent, wxJoystickEvent, wxMenuEvent, wxMouseEvent, wxMoveEvent, wxPaintEvent, wxQueryLayoutInfoEvent, wxSizeEvent, wxScrollWinEvent, wxSysColourChangedEvent 。
那wxCommandEvent到底有啥不同呢，接着看

    wxEvent::wxEvent(int theId, wxEventType commandType )
    {
        // ......
        m_propagationLevel = wxEVENT_PROPAGATE_NONE;
    }

    wxCommandEvent::wxCommandEvent(wxEventType commandType, int theId)
                  : wxEvent(theId, commandType)
    {
         // ......
        // the command events are propagated upwards by default
        m_propagationLevel = wxEVENT_PROPAGATE_MAX;
    }

##事件处理流程
wx事件机制采用事件表机制实现，事件表的实现稍后再分析。这里只要把事件表理解为事件到事件处理函数的的映射关系即可。事件发生时，会在事件表里查找事件处理函数，
事件表查找一般流程是：  
**    当前窗口事件表--->父类事件表-->父类的父类事件表--> ，，，，-->父窗口事件表**  
一旦查找成功，匹配结束，调用事件处理函数,返回一个true，不再继续向上查找事件

    //common/event.cpp
    bool wxEvtHandler::ProcessEvent(wxEvent& event)
    {
         // ....
            // 事件哈希表查找处理（这里会搜索父类）
            if ( GetEventHashTable().HandleEvent(event, this) )
                return true;


         // ....
        // EvtHandler类组成一个栈，搜索下一个
        if ( GetNextHandler() )
        {
            if ( GetNextHandler()->ProcessEvent(event) )
                return true;
        }

        // .....
        // 继续上传递，这是个虚函数，对于wxWindow向父窗口传递 
        return TryParent(event);
    }

##程序中使用事件处理机制
对于用wx编程，要使一个类具备事件处理能力，需要  
1. 定义一个类直接或间接继承wxEvtHandle（一般wxWindow的窗口或控件子类、wxApp都继承了wxEvtHandler）
2. 在类的声明中，前加入事件表声明宏 DECLARE_EVENT_TABLE()
3. 类的实现中，加入事件表实现，以BEGIN_EVENT_TABLE(MyWin, BaseWin)开始 ，以END_EVENT_TABLE()结束，中间填入要处理的事件映射宏
    
    BEGIN_EVENT_TABLE(MyFrame, wxFrame)  
        EVT_MENU(Minimal_Quit,  MyFrame::OnQuit)  
        EVT_MENU(Minimal_About, MyFrame::OnAbout)  
    END_EVENT_TABLE()
    
一堆像模像样的宏，那么这背后wx到底做了什么呢 ？接着分析事件表的具体实现

##事件表的实现
先说说事件表机制的相关的数据结构

**事件类型(wxEventType)**  
wx中事件类型（wxEventType）和事件类(wxEvent）是两个不同概念,上面分析事件类是一序列继承于wxEvent的类,wxCommandEvent事件类可传递，其他不可传递。一个事件类有多个事件类型，比如同是wxCommandEvent事件，有下列事件类型
* wxEVT_COMMAND_BUTTON_CLICKED
* wxEVT_COMMAND_CHECKBOX_CLICKED
* wxEVT_COMMAND_CHOICE_SELECTED
* wxEVT_COMMAND_LISTBOX_SELECTED
* wxEVT_COMMAND_LISTBOX_DOUBLECLICKED
* wxEVT_COMMAND_TEXT_UPDATED
* wxEVT_COMMAND_TEXT_ENTER
* wxEVT_COMMAND_MENU_SELECTED
这些实际是一些枚举常量，说白了都是一些代表事件类型整形值。wx中事件类型就是一系列的整型值，这些值在事件系统中唯一的用于标识事件。

    //event.h
    typedef int wxEventType;
    //wxEvent构造函数参数有两个成员winid commmandType
    wxEvent(int winid = 0, wxEventType commandType = wxEVT_NULL )


（未完，有点长， 之后再加吧）








