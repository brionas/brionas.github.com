---
layout: post
title : wxWidgets源码分析-事件机制（下）
description : wxWidgets是一款开源的跨平台图形程序开发库，其中事件机制是GUI程序开发的一个重中之重，通过阅读源代码分析wx中事件机制的实现。具体包括到事件的定义、分发、处理以及跨平台的实现
category : wxWidgets
tags : [wxWidgets, 源码分析]
---
{% include JB/setup %}

##1、事件哈希表的实现
在`wxEventHashTable`构造函数里面，并没有构建哈希表，而是用一个布尔变量标识哈希表尚未重建，采用这种延迟重建的方式来实现哈希表，
哈希表真正建立，发生发生在第一次查找事件表时，重建以后，设置布尔变量，以后只要对哈希表查找即可，从而加速了事件表的查找。
哈希函数采用除以size取余的方法。

以下是`wxEventTable` -> `wxEventHashTable`的过程

1. 取当前事件表table
2. table为空结束
3. 遍历table中所有事件表条目，把事件表条目加入哈希表
4. `table = table->base`,转1继续对父类事件表哈希

{% highlight cpp linenos %}
void wxEventHashTable::InitHashTable()
{
    // Loop over the event tables and all its base tables.
    const wxEventTable *table = &m_table;
    while (table)
    {
        // Retrieve all valid event handler entries
        const wxEventTableEntry *entry = table->entries;
        while (entry->m_fn != 0)
        {
            // Add the event entry in the Hash.
            AddEntry(*entry);

            entry++;
        }

        table = table->baseTable;
    }
 //....
}

void wxEventHashTable::AddEntry(const wxEventTableEntry &entry)
{
     //..
    EventTypeTablePointer *peTTnode = &m_eventTypeTable[entry.m_eventType % m_size];
    EventTypeTablePointer  eTTnode = *peTTnode;

    if (eTTnode)
    {
        if (eTTnode->eventType != entry.m_eventType)
        {
            // Resize the table!
            GrowEventTypeTable();
            // Try again to add it.
            AddEntry(entry);
            return;
        }
    }
    else
    {
        eTTnode = new EventTypeTable;
        eTTnode->eventType = entry.m_eventType;
        *peTTnode = eTTnode;
    }

    // Fill all hash entries between entry.m_id and entry.m_lastId...
    eTTnode->eventEntryTable.Add(&entry);
}{% endhighlight %}


对于上面的弟3步，把事件条目加入哈希表，`wxEventTableEntry -> wxEventHashTable`的过程

1. 由事件表表条中事件类型对哈希表长度取模得到i&nbsp;
2. 根据i的值，得到事件类型表地址数组`m_eventTypeTable`的下标&nbsp;
3. `m_eventTypeTable[i] = 0`,说明该事件类型事件类型表尚未填入哈希表，转4 ，否则转5&nbsp;
4. 新建一个事件类型表,将表地址填入哈希表，转6&nbsp;
5. 该事件类型表已经存在，冲突，扩容哈希表，重新回到1&nbsp;
6. 将事件表条目的地址填入事件类型表的`eventEntryTable`，结束&nbsp;



##2、事件表有关宏的背后
`wxEvtHandler`中有三个静态成员，`sm_eventTable` `sm_eventHashTable`分别代表当前事件处理类事件表和事件哈希表,
要被子类连接事件表所有为protected。所有要处理事件的类继承`wxEvtHandler`,需要重新定义这个两个静态成员。
因此两个虚函数分别返回当前类中事件表和事件哈希表。另一个私有的静态成员`sm_eventTableEntries`表示当前类的
事件条目数组，用于构成事件表。

{% highlight cpp linenos %}
class WXDLLIMPEXP_BASE wxEvtHandler : public wxObject
{
//..
private:
    static const wxEventTableEntry sm_eventTableEntries[];
protected:
 static const wxEventTable sm_eventTable;
    virtual const wxEventTable *GetEventTable() const;

    static wxEventHashTable   sm_eventTable;
    virtual wxEventHashTable& GetEventHashTable() const;
//..
}{% endhighlight %}

在类的声明中使用`DECLARE_EVENT_TABLE()`，实际上市覆盖`wxEvtHandler`中的事件表和事件哈希表，并重写返回当前
类事件表和事件哈希表的虚函数

{% highlight cpp linenos %}
#define DECLARE_EVENT_TABLE() \
    private: \
        static const wxEventTableEntry sm_eventTableEntries[]; \
    protected: \
        static const wxEventTable        sm_eventTable; \
        virtual const wxEventTable*      GetEventTable() const; \
        static wxEventHashTable          sm_eventHashTable; \
        virtual wxEventHashTable&        GetEventHashTable() const;
{% endhighlight %}

在类的实现中使用`BEGIN_EVENT_TABLE`和`END_EVENT_TABLE()`实际上初始化当前类的事件表（一个静态成员），
事件表父指针指向父类事件表，事件表的条目指针事件等于事件条目数组首地址。事件哈希表有事件表生成，
这里还没有构造，值是分配了31个为0的空间。中间的一堆事件映射宏实际上是用一堆5元组初始化当前类的事件条目表。

{% highlight cpp linenos %}
#define BEGIN_EVENT_TABLE(theClass, baseClass) \
    const wxEventTable theClass::sm_eventTable = \
        { &baseClass::sm_eventTable, &theClass::sm_eventTableEntries[0] }; \
    const wxEventTable *theClass::GetEventTable() const \
        { return &theClass::sm_eventTable; } \
    wxEventHashTable theClass::sm_eventHashTable(theClass::sm_eventTable); \
    wxEventHashTable &theClass::GetEventHashTable() const \
        { return theClass::sm_eventHashTable; } \
    const wxEventTableEntry theClass::sm_eventTableEntries[] = { \

 EVT_MENU(Minimal_About, MyFrame::OnAbout)
 DECLARE_EVENT_TABLE_ENTRY(evt, id1, id2, fn, NULL),

#define END_EVENT_TABLE() DECLARE_EVENT_TABLE_ENTRY( wxEVT_NULL, 0, 0, 0, 0 ) };
{% endhighlight %}

这样每个事件处理的类会用事件映射宏的5元组，构造事件条目表，事件条目表首地址，
放在事件表中，同时事件表会记录父类的事件表地址。哈希表初始化为一堆0地址，搜索一次后，
会把当前事件表和父类所有的事件表哈希到当前事件哈希表中，这样以后该类的对象，
事件表查找，只要查找已经构建好的事件哈希表， 事件复杂度近似O(1)。


##3、引发处理事件的方式
上面已经分析过事件处理的流程，那么什么情况会触发一个事件被处理呢？有以下几种方式
* 窗口或控件感应到操作（来自用户的操作引发事件处理流程）&nbsp;
* 调用`wxTheApp->ProcessPendingEvents()`
* 调用`wxEvtHandler::ProcessEvent(wxEvent &event)`（应用程序自己调用）

wx保存了一个全局的指针链表，里面保存当前应用程序所有事件处理类`wxEvtHandler`对象的指针

{% highlight cpp linenos %}
/* code: common/event.cpp:144  */
wxList *wxPendingEvents = (wxList *)NULL;
{% endhighlight %}

对于所有事件类的基类`wxEvtHandler`,有个成员`m_pendingEvents`保存当前`wxEvtHandler`对象的未决事件

{% highlight cpp linenos %}
class WXDLLIMPEXP_BASE wxEvtHandler : public wxObject
{
     //..
     wxList*             m_pendingEvents;
     //..
}
{% endhighlight %}

遍历全局链表未决事件链表`wxPendingEvents`，取出所有的`wxEvtHandler`，然后调用每个`wxEvtHandler`上的`ProcessPendingEvents()`方法

{% highlight cpp linenos %}
/* code: src/common/Appbase.cpp:267 */
void wxAppConsole::ProcessPendingEvents()
{

    // ...
  
   // iterate until the list becomes empty
    wxList::compatibility_iterator node = wxPendingEvents->GetFirst();
    while (node)
    {
        wxEvtHandler *handler = (wxEvtHandler *)node->GetData();
        wxPendingEvents->Erase(node);

        // In ProcessPendingEvents(), new handlers might be add
        // and we can safely leave the critical section here.
        wxLEAVE_CRIT_SECT( *wxPendingEventsLocker );

        handler->ProcessPendingEvents();

        wxENTER_CRIT_SECT( *wxPendingEventsLocker );

        node = wxPendingEvents->GetFirst();
    }
    
     // ...
}
{% endhighlight %}

`wxEvtHandler`上的`ProcessPendingEvents()`方法中`wxEvtHandler`遍历处理自己未决链表`m_pendingEvents`的事件

{% highlight cpp linenos %}
void wxEvtHandler::ProcessPendingEvents()
{
     //..
    size_t n = m_pendingEvents->size();
    for ( wxList::compatibility_iterator node = m_pendingEvents->GetFirst();
          node;
          node = m_pendingEvents->GetFirst() )
    {
        wxEventPtr event(wx_static_cast(wxEvent *, node->GetData()));

        // It's important we remove event from list before processing it.
        // Else a nested event loop, for example from a modal dialog, might
        // process the same event again.

        m_pendingEvents->Erase(node);

        wxLEAVE_CRIT_SECT( Lock() );

        ProcessEvent(*event);

        wxENTER_CRIT_SECT( Lock() );

        if ( --n == 0 )
            break;
    }
     //..
}
{% endhighlight %}

`wxEvtHandler`提供了向自己接未决链表加入事件的方法，`wxEvtHandler:AddPendingEvent`，
注意这里只是把event clone一份，然后加入未决链表， 并不处理就返回。

{% highlight cpp linenos %}
void wxEvtHandler:AddPendingEvent(wxEvent& event)
{% endhighlight %}

wxEvtHandler::ProcessEvent用于立即处理这个事件。
{% highlight cpp linenos %}
bool wxEvtHandler::ProcessEvent(wxEvent& event)
{% endhighlight %}



##3、消息分发机制
上述所有`wxEvent`相关的东西都是跨平台的，wxEvent是wx定义出来的一个wx事件类，
wx程序中处理的都是wxEvent，并不设计到具体平台上的消息处理。然后实际上，
对于GUI程序设计每个平台都有消息循环、消息分发机制，wx为了跨平台提供了一个抽象层，
屏蔽了平台相关的消息处理部分。在include/wx下的头文件提供了wx对外公共的接口，
利用里面的接口我们可以编写平台无关的GUI的程序，然而include/wx中头文件接口的实现却依赖于具体的平台，
实际上上平台相关的接口位于/include/wx/gtk /include/wx/msw等，但用户不必关心这里平台相关的细节，
只要利用提供的公共接口编程即可。下面分析wx中消息分发以及在不同平台的实现。

wx程序启动流程如下：
* 建立个wxApp子类MyApp的实例
* 调用MyApp中重写的虚函数`wxApp::OnInit`完成初始化（主要是创建顶层窗口）,返回false结束
* 调用`AppBase::OnRun->wxAppBase::MainLoop`进入消息循环

{% highlight cpp linenos %}
/* src/common/appcmn.cpp::357 */
int wxAppBase::OnRun()
{
    // see the comment in ctor: if the initial value hasn't been changed, use
    // the default Yes from now on
    if ( m_exitOnFrameDelete == Later )
    {
        m_exitOnFrameDelete = Yes;
    }
    //else: it has been changed, assume the user knows what he is doing

    return MainLoop();
}

// .....

int wxAppBase::MainLoop()
{
    wxEventLoopTiedPtr mainLoop(&m_mainLoop, new wxEventLoop);

    return m_mainLoop->Run();
}

/* inclucde/wx/app.h:329 */
class WXDLLIMPEXP_CORE wxAppBase : public wxAppConsole
{
     //..
     wxEventLoop *m_mainLoop;
     //..
}
{% endhighlight %}

`m_mainLoop`是个事件循环类`wxEventLoop`的对象指针,`wxEventLoop`是个平台相关的事件循环类，从此进入不同平台的消息循环


##4、MSW版本中消息分发机制

    WinMain -> wxEntry -> wxEntryReal -> wxAppBase::OnRun -> wxAppBase::MainLoop ->
    wxEventLoopManual::Run -> wxEventLoop::Dispatch -> wxEventLoop::ProcessMessage

{% highlight cpp linenos %}
/* src/common.Evtloopcmn.cpp:65 */
int wxEventLoopManual::Run()
{ 
                while ( !Pending() && (wxTheApp && wxTheApp->ProcessIdle()) )
                    ;

                if ( m_shouldExit )
                {
                    while ( Pending() )
                        Dispatch();

                    break;
                }
}
{% endhighlight %}

程序进入一个无限的循环，果没消息处理，就处理idle消息，有消息就分发消息

{% highlight cpp linenos %}
bool wxEventLoop::Dispatch()
{
     // ..

    MSG msg;
    BOOL rc = ::GetMessage(&msg, (HWND) NULL, 0, 0);

    ProcessMessage(&msg);


     
     //..
}
{% endhighlight %}

取消息处理消息，这里的消息是`WXMSG *msg`
`typedef struct tagMSG   WXMSG;`
在Windows程序中，消息是由`MSG`结构体来表示的。MSG结构体的定义如下（参见MSDN）：

{% highlight cpp linenos %}
typedef struct tagMSG {     // msg 
   HWND hwnd;               //标识窗口过程接收消息的窗口
   UINT message;          //指定消息号
   WPARAM wParam;          //指定有关消息的附加信息。 确切含义取决于 message 成员的值
   LPARAM lParam;          //指定有关消息的附加信息。 确切含义取决于 message 成员的值。
   DWORD time;               //指定消息已传递的时间
   POINT pt;               //当消息已传递了，指定光标位置，在屏幕坐标
} MSG;
{% endhighlight %}

这里的消息就是windows中的消息了

{% highlight cpp linenos %}
void wxEventLoop::ProcessMessage(WXMSG *msg)
{
    // give us the chance to preprocess the message first
    if ( !PreProcessMessage(msg) )
    {
        // if it wasn't done, dispatch it to the corresponding window
        ::TranslateMessage(msg);
        ::DispatchMessage(msg);
    }
}
{% endhighlight %}

这里就是win32 SDK中的消息循环了，说白了wx中的事件处理，最终还是用了平台上的消息机制，
wx只是完成了跨平台的封装，对用户屏蔽了平台相关的是实现细节，抽象出一个用户直接利用的抽象层。

* windows中的消息`WXMSG`怎么转换成`wxEvent*`

    wxEntryReal -> wxEntryStart -> wxApp::Initialize() -> wxApp:: RegisterWindowClasses()

注册一个窗口类，这个窗口类绑定了窗口处理过程

{% highlight cpp linenos %}
/* src/msw/app.cpp */
bool wxApp::RegisterWindowClasses()
{
    WNDCLASS wndclass;
    wxZeroMemory(wndclass);

    // for each class we register one with CS_(V|H)REDRAW style and one
    // without for windows created with wxNO_FULL_REDRAW_ON_REPAINT flag
    static const long styleNormal = CS_HREDRAW | CS_VREDRAW | CS_DBLCLKS;
    static const long styleNoRedraw = CS_DBLCLKS;

    // the fields which are common to all classes
    wndclass.lpfnWndProc   = (WNDPROC)wxWndProc;
    wndclass.hInstance     = wxhInstance;
    wndclass.hCursor       = ::LoadCursor((HINSTANCE)NULL, IDC_ARROW);

    // register the class for all normal windows
    wndclass.hbrBackground = (HBRUSH)(COLOR_BTNFACE + 1);
    wndclass.lpszClassName = wxCanvasClassName;
    wndclass.style         = styleNormal;

    if ( !RegisterClass(&wndclass) )
    {
        wxLogLastError(wxT("RegisterClass(frame)"));
    }

     //..
}
{% endhighlight %}

然后里创建窗口

{% highlight cpp linenos %}
/* src/msw/window.cpp */
bool wxWindowMSW::MSWCreate(const wxChar *wclass,
                            const wxChar *title,
                            const wxPoint& pos,
                            const wxSize& size,
                            WXDWORD style,
                            WXDWORD extendedStyle)
{
     wxString className(wclass);
    if ( !HasFlag(wxFULL_REPAINT_ON_RESIZE) )
    {
        className += wxT("NR");
    }

    // do create the window
    wxWindowCreationHook hook(this);

    m_hWnd = (WXHWND)::CreateWindowEx
                       (
                        extendedStyle,
                        className,
                        title ? title : m_windowName.c_str(),
                        style,
                        x, y, w, h,
                        (HWND)MSWGetParent(),
                        (HMENU)controlId,
                        wxGetInstance(),
                        NULL                        // no extra data
                       );

    if ( !m_hWnd )
    {
        wxLogSysError(_("Can't create window of class %s"), className.c_str());

        return false;
    }

    SubclassWin(m_hWnd);
}
{% endhighlight %}

其中，`void wxWindowMSW::SubclassWin(WXHWND hWnd)`用于设置新窗口的窗口处理过程 

{% highlight cpp linenos %}
WXLRESULT wxWindowMSW::MSWWindowProc(WXUINT message, WXWPARAM wParam, WXLPARAM lParam)
{
     //...
     switch ( message )
    {
        case WM_CREATE:

         case WM_LBUTTONDOWN:
        case WM_LBUTTONUP:
        case WM_LBUTTONDBLCLK:
        case WM_RBUTTONDOWN:
        case WM_RBUTTONUP:
        case WM_RBUTTONDBLCLK:
        case WM_MBUTTONDOWN:
        case WM_MBUTTONUP:
        case WM_MBUTTONDBLCLK:
}
{% endhighlight %}

可见MSW 版本中的事件wxEvent由windows中消息`WN_XX`而来


##5、GTK版本中消息分发机制
程序启动后，GTK版本进入的gtk的wxEventLoop事件循环类,GTK是一种事件驱动工具包，这意味着它将在`gtk_main`函数
中一直等待，直到事件发生和控制权被传递给相应的函数。`gtk_main()`是在每个GTK应用程序都要调用的函数。
当程序运行到这里时, Gtk将进入等待态，等候X事件(比如点击按钮或按下键盘的某个按键)、Timeout 或文件输入/输出发生。

{% highlight cpp linenos %}
/* src/gtk/evtloop.cpp */
int wxEventLoop::Run()
{
    // event loops are not recursive, you need to create another loop!
    wxCHECK_MSG( !IsRunning(), -1, _T("can't reenter a message loop") );

    wxEventLoopActivator activate(this);

    m_impl = new wxEventLoopImpl;

    gtk_main();

    OnExit();

    int exitcode = m_impl->GetExitCode();
    delete m_impl;
    m_impl = NULL;

    return exitcode;
}
{% endhighlight %}

