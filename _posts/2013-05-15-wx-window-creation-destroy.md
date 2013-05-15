---
layout: post
title: "[wxWidget源码阅读] 理解wxWindow中增加和去除/销毁子窗口的过程"
description: ""
category: "wx"
tags: [程序开发, wxWidgets]
---
{% include JB/setup %}

wxWindow中增加和去除/销毁子窗口的过程：

##增加一个子窗口
{% highlight cpp %}
void wxWindowBase::AddChild(wxWindowBase *child)
{
    wxCHECK_RET( child, wxT("can't add a NULL child") );


    // this should never happen and it will lead to a crash later if it does
    // because RemoveChild() will remove only one node from the children list
    // and the other(s) one(s) will be left with dangling pointers in them
    wxASSERT_MSG( !GetChildren().Find((wxWindow*)child), _T("AddChild() called twice") );


    GetChildren().Append((wxWindow*)child);
    child->SetParent(this);
}
{% endhighlight %}
##销毁所有的子窗口
{% highlight cpp %}
bool wxWindowBase::DestroyChildren()
{
    wxWindowList::compatibility_iterator node;
    for ( ;; )
    {
        // we iterate until the list becomes empty
        node = GetChildren().GetFirst();
        if ( !node )
            break;


        wxWindow *child = node->GetData();


        // note that we really want to call delete and not ->Destroy() here
        // because we want to delete the child immediately, before we are
        // deleted, and delayed deletion would result in problems as our (top
        // level) child could outlive its parent
        delete child;


        wxASSERT_MSG( !GetChildren().Find(child),
                      wxT("child didn't remove itself using RemoveChild()") );
    }


    return true;
}
{% endhighlight %}
上面直接delete一个子窗口的指针。它会触发子窗口的析构。
{% highlight cpp %}
   // reset the dangling pointer our parent window may keep to us
    if ( m_parent )
    {
        m_parent->RemoveChild(this);
    }
{% endhighlight %}
析构过程中会调用父窗口的RemoveChild函数，用来去除该子窗口，从父窗口的子窗口链表中。
{% highlight cpp %}
void wxWindowBase::RemoveChild(wxWindowBase *child)
{
    wxCHECK_RET( child, wxT("can't remove a NULL child") );


    GetChildren().DeleteObject((wxWindow *)child);
    child->SetParent(NULL);
}
{% endhighlight %}
##wxWindows的reparent
如果没有指定parent，那么就将这个窗口添加到全局的toplevel windows列表中。
{% highlight cpp %}
bool wxWindowBase::Reparent(wxWindowBase *newParent)
{
    wxWindow *oldParent = GetParent();
    if ( newParent == oldParent )
    {
        // nothing done
        return false;
    }


    // unlink this window from the existing parent.
    if ( oldParent )
    {
        oldParent->RemoveChild(this);
    }
    else
    {
        wxTopLevelWindows.DeleteObject((wxWindow *)this);
    }


    // add it to the new one
    if ( newParent )
    {
        newParent->AddChild(this);
    }
    else
    {
        wxTopLevelWindows.Append((wxWindow *)this);
    }


    return true;
}

{% endhighlight %}
