---
layout: post
title: "[wxWidgetԴ���Ķ�] ���wxWindow�����Ӻ�ȥ��/�����Ӵ��ڵĹ���"
description: ""
category: "wx"
tags: [���򿪷�, wxWidgets]
---
{% include JB/setup %}

wxWindow�����Ӻ�ȥ��/�����Ӵ��ڵĹ��̣�

##����һ���Ӵ���
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
##�������е��Ӵ���
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
����ֱ��deleteһ���Ӵ��ڵ�ָ�롣���ᴥ���Ӵ��ڵ�������
{% highlight cpp %}
   // reset the dangling pointer our parent window may keep to us
    if ( m_parent )
    {
        m_parent->RemoveChild(this);
    }
{% endhighlight %}
���������л���ø����ڵ�RemoveChild����������ȥ�����Ӵ��ڣ��Ӹ����ڵ��Ӵ��������С�
{% highlight cpp %}
void wxWindowBase::RemoveChild(wxWindowBase *child)
{
    wxCHECK_RET( child, wxT("can't remove a NULL child") );


    GetChildren().DeleteObject((wxWindow *)child);
    child->SetParent(NULL);
}
{% endhighlight %}
##wxWindows��reparent
���û��ָ��parent����ô�ͽ����������ӵ�ȫ�ֵ�toplevel windows�б��С�
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
