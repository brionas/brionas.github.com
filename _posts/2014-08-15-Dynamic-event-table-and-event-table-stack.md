---
layout: post
title: wxWidgets源码分析-动态事件表与事件表栈
description: wxWidgets源码分析-动态事件表与事件表栈
category: wxWidget
tags: [
        wxWidget
        event table
      ]
refer_author: Tank
refer_blog_addr:
refer_post_addr:

---
之前分析过wxWidgets事件表的具体实现，为了理解事件表的核心原理，简化事件处理模型，我们所有的讨论都是基于静态事件表， 忽略了动态事件表和事件表栈,这里我们来讨论下动态事件表和事件表栈。

先从区别说起:

- 静态事件表： BEGIN_EVENT_TABLE和END_EVENT_TABLE之间事件映射宏，这里一堆宏实际上是给静态成员事件条目数组sm_eventTableEntries初始化，可见这些都是编译期就确定的。
- 动态事件表：表现在可以在程序运行期间，动态添加删除事件表条目
- 事件表链： 所有具备事件处理能力的类都是wxEvtHandler，默认这个wxEvtHandler就是this,但是程序运行期间我们可以链接多个wxEvtHandler,组成一个栈，栈顶的优先响应。

动态事件表
===

先看看我们怎么使用动态事件表,wxEvtHandler主要提供了两个接口
{% highlight cpp  %}
void Connect(int winid,
             int lastId,
             int eventType,
             wxObjectEventFunction func,
             wxObject *userData = (wxObject *) NULL,
             wxEvtHandler *eventSink = (wxEvtHandler *) NULL);
bool Disconnect(int winid,
                int lastId,
                wxEventType eventType,
                wxObjectEventFunction func = NULL,
                wxObject *userData = (wxObject *) NULL,
                wxEvtHandler *eventSink = (wxEvtHandler *) NULL);
{% endhighlight %}
预料之中啊，要动态添加删除事件表条目，肯定要传五元组过来，所以这两个接口参数就是事件表条目五元组，程序里面用就更简单了,我们可以在窗口构造函数或事件响应了中connect过来， 这些都是运行期间完成的。
{% highlight cpp  %}
frame->Connect(ID_About, wxEVT_COMMAND_MENU_SELECTED, wxCommandEventHandler(MyFrame::OnAbout));
frame->Connect(ID_Quit, wxEVT_COMMAND_MENU_SELECTED, wxCommandEventHandler(MyFrame::OnQuit));
{% endhighlight %}
这样就可以和静态事件表一样的响应事件了，下面来分析下，动态事件表示怎么实现的吧
{% highlight cpp  %}
    class WXDLLIMPEXP_BASE wxEvtHandler : public wxObject
    {
    public:
     // ....
    wxList* m_dynamicEvents;
     // ...
    };
{% endhighlight %}
wxEvtHandle维护一个链表指针m_dynamicEvents,这里实际上是一个事件表条目wxEventTableEntry构成的链表，构造函数里面初始化为NULL.调用connect的时候把事件表条目插入链表表头，调用disconect就是从链表中删除节点了。
{% highlight cpp  %}
void wxEvtHandler::Connect( int id, int lastId, int eventType,
                            wxObjectEventFunction func,
                            wxObject *userData,
                            wxEvtHandler* eventSink )
{
#if WXWIN_COMPATIBILITY_EVENT_TYPES
    wxEventTableEntry *entry = new wxEventTableEntry;
    entry->m_eventType = eventType;
    entry->m_id = id;
    entry->m_lastId = lastId;
    entry->m_fn = func;
    entry->m_callbackUserData = userData;
#else // !WXWIN_COMPATIBILITY_EVENT_TYPES
    wxDynamicEventTableEntry *entry =
    new wxDynamicEventTableEntry(eventType, id, lastId, func, userData, eventSink);
#endif // WXWIN_COMPATIBILITY_EVENT_TYPES/!WXWIN_COMPATIBILITY_EVENT_TYPES
    if (!m_dynamicEvents)
        m_dynamicEvents = new wxList;
    // Insert at the front of the list so most recent additions are found first
    m_dynamicEvents->Insert( (wxObject*) entry );
}
bool wxEvtHandler::Disconnect( int id, int lastId, wxEventType eventType,
                               wxObjectEventFunction func,
                               wxObject *userData,
                               wxEvtHandler* eventSink )
{
   if (!m_dynamicEvents)
   return false;
   wxList::compatibility_iterator node = m_dynamicEvents->GetFirst();
   while (node)
   {
#if WXWIN_COMPATIBILITY_EVENT_TYPES
        wxEventTableEntry *entry = (wxEventTableEntry*)node->GetData();
#else // !WXWIN_COMPATIBILITY_EVENT_TYPES
        wxDynamicEventTableEntry *entry = (wxDynamicEventTableEntry*)node->GetData();
#endif // WXWIN_COMPATIBILITY_EVENT_TYPES/!WXWIN_COMPATIBILITY_EVENT_TYPES
        if ((entry->m_id == id) &&
            ((entry->m_lastId == lastId) || (lastId == wxID_ANY)) &&
            ((entry->m_eventType == eventType) || (eventType == wxEVT_NULL)) &&
            ((entry->m_fn == func) || (func == (wxObjectEventFunction)NULL)) &&
            ((entry->m_eventSink == eventSink) || (eventSink == (wxEvtHandler*)NULL)) &&
            ((entry->m_callbackUserData == userData) || (userData == (wxObject*)NULL)))
        {
            if (entry->m_callbackUserData)
                delete entry->m_callbackUserData;
            m_dynamicEvents->Erase( node );
            delete entry;
            return true;
        }
        node = node->GetNext();
    }
    return false;
}
{% endhighlight %}

事件表栈
===
wxWidgets里面每个具备事件处理能力的类都是wxEvtHandler，即想处理事件必须直接或间接继承于wxEvtHandler，我们的wxWindow，wxApp都是继承了wxEvtHandler，在wxEvthandler中记录了事件表。然而wxEvthandler并不仅仅单独存在的，而是可以构成一个双向链表，里面有一个前向指针和一个后向指针， wxEvthandler构造函数默认初始化为NULL，即只有当前一个节点。

    class WXDLLIMPEXP_BASE wxEvtHandler : public wxObject
    {
    public:
     // ....
    wxEvtHandler* m_nextHandler;
    wxEvtHandler* m_previousHandler;
     // ...
    };

对于窗口类来说，由wxEvtHandler节点构成一个栈，维护一个m_eventHandler栈顶指针，栈上的每个wxEvtHandler都具有处理该窗口事件的能力，只是优先级不一样，栈顶优先。wxWindowBase构造函数中m_eventHandler初始化为this.这也是绝大多数的情况，这个栈只有一个节点构成，即m_eventHandler指向窗口本身this, wxWindowbase 提供接口wxWindowBase::GetEventHandler返回栈顶的wxEventhandler.
 {% highlight cpp  %}
    class WXDLLEXPORT wxWindowBase : public wxEvtHandler
    {
    public:
     // creating the window
     // -------------------
     // event handler for this window: usually is just 'this' but may be
     // changed with SetEventHandler()
    wxEvtHandler *m_eventHandler;
     // -------------------
    }；
    wxWindowBase::wxWindowBase()
    {
     // ----------------------
     // the default event handler is just this window
    m_eventHandler = this;
     // ----------------------
    }
{% endhighlight %}

再来看看窗口类提供事件表栈的两个接口,简单易懂啊，就是入栈出栈操作了。
再说事件处理流程
===

事件表查找一般流程是：
当前窗口事件表—>父类事件表–>父类的父类事件表–>......–>父窗口事件表 一旦查找成功，匹配结束，调用事件处理函数,返回一个true，不再继续向上查找事件

引入动态事件表和事件表栈后完整事件表查找流程

1. 取当前窗口的栈顶wxEvtHandler handler
1. 查找hanlder的动态事件表
1. 查找hanlder的静态事件表(handler本身事件表->handler父类事件表->…->handler父类的父类的事件表->..)
1. handler = nexthanlder, 如果不为NULL回到（2）
1. 查找handler的parent事件表(窗口类的parent是父窗口，其他的是查wxTheApp)
{% highlight cpp  %}
bool wxEvtHandler::ProcessEvent(wxEvent& event)
{
    // allow the application to hook into event processing
    if ( wxTheApp )
    {
        int rc = wxTheApp->FilterEvent(event);
        if ( rc != -1 )
        {
            wxASSERT_MSG( rc == 1 || rc == 0,
                    _T("unexpected wxApp::FilterEvent return value") );
            return rc != 0;
        }
        //else: proceed normally
    }
    // An event handler can be enabled or disabled
    if ( GetEvtHandlerEnabled() )
    {
        // if we have a validator, it has higher priority than our own event
        // table
        if ( TryValidator(event) )
            return true;
        // Handle per-instance dynamic event tables first
        if ( m_dynamicEvents && SearchDynamicEventTable(event) )
            return true;
        // Then static per-class event tables
        if ( GetEventHashTable().HandleEvent(event, this) )
            return true;
    }
    // Try going down the event handler chain
    if ( GetNextHandler() )
    {
        if ( GetNextHandler()->ProcessEvent(event) )
            return true;
    }
    // Finally propagate the event upwards the window chain and/or to the
    // application object as necessary
    return TryParent(event);
}
{% endhighlight %}
 
