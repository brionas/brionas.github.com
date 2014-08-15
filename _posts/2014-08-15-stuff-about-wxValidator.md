---
layout: post
title: 输入容易，检验不易，且用且珍惜——wxValidator拾趣
description: "wxValidator 的使用"
category: wxWidget
tags: wxWidget
refer_author: Wenliang
refer_blog_addr:
refer_post_addr:
---


#什么是validator

我们设计界面程序的时候总要使用大量的控件，用户通过控件输入配置数据，最常见的就是文本输入框，在wxWidgets里面对应的类是wxTextCtrl。

大部分情况下，我们输入的数据要么是有范围的，要么遵循一定的规则。而人们在输入的时候往往忘记相应的规则，比如月份只有1月到12月，输入13就不对了。

旧式的检验数据合法性的做法是这样的，当我们准备关闭一个对话框的时候，我们逐个控件地得到wxString类型的数据，接着我们把它转换成long或者double的数值类型，检验的代码可能这样写：

{% highlight cpp  %}
wxTextCtrl * textctrl = new ...
double value1 = 0.0;
if (!textctrl1->GetValue().ToDouble(&value1))
{
    return false;
}
if (value1...)
{
    return false;
}
double value2;
if (!textctrl2->GetValue().ToDouble(&value2))
{    
    return false;
}
if (value2...)
{
    return false;
}
...
if (value100...)
{
    return false
}
{% endhighlight %}


这是多么冗繁的代码啊。字符串到数值转换的代码重复出现，复杂的判断条件堆积在一起，没有人愿意维护这样的代码。

幸运的是，wxWidgets提供wxValidator相关的类一定程度上解决了这个问题。它本身已经提供很多常用的类，我们也可以自己定义特定规则的validator。

validator其实就是一个嵌入到控件里面的对象，作为数据与控件的中介，检验它们的双向转换。它能够拦截控件产生的事件，提供数据过滤功能。

validator有两种典型的工作方式。第一种是在输入字符的时候，根据设定的规则检验字符。如果字符不符合规定，该字符将不能显示在界面上。比如一个文本框只能输入数字，当输入字母的时候，虽然有键盘按键，但是文本框没有显示输入，而且电脑会发出警报声。第二种方式是在包含数据的对话框快要消失的时候检验控件里面的数据，如果数据不合法，一个小型的对话框会弹出来提示输入不正确，这时无法关掉包含数据的对话框，直到你输入合法的数据。

#validator的使用
{% highlight cpp  %}
m_pTC = new wxTextCtrl(this, wxID_ANY, wxEmptyString, wxDefaultPosition, wxDefaultSize, 0, wxTextValidator(wxFILTER_ALPHA, str));
{% endhighlight %}

在第七个参数里面，我们创建了一个wxTextValidator的对象，只允许输入字母。m_pTC这个控件就与validator绑定在一起了。

深入分析内部实现，validator其实在控件里面被拷贝了一份，控件拥有一个自己的validator的对象：

{% highlight cpp  %}
void wxWindowBase::SetValidator(const wxValidator& validator)
{
    if ( m_windowValidator )
        delete m_windowValidator;
  
    m_windowValidator = (wxValidator *)validator.Clone();
  
    if ( m_windowValidator )
        m_windowValidator->SetWindow(this);
}
{% endhighlight %}

wxWindowBase是wxTextCtrl的的基类，控件的对象在构造过程中会调用SetValidator函数。

如果你需要重新配置控件里面的validator，可以这样得到它：
{% highlight cpp  %}
wxValidator* val = m_pTC->GetValidator();
{% endhighlight %}
validator在控件里面被拷贝也说明了只要建立一个validator对象，它就可以被很多控件使用，因为控件都保有自己的一份拷贝，可以自由地更改validator的配置，控件之间互不影响。


#validator是如何工作的
刚才我们提到validator有两种工作方式。

第一种就是禁止输入不合法的数据。

比如，在不允许输入数字的控件里面输入数字。

在我们输入数字的时候，文本框需要处理键盘事件，处理函数如下：
{% highlight cpp  %}
bool wxEvtHandler::ProcessEvent(wxEvent& event)
{
    ...
    if ( GetEvtHandlerEnabled() )
    {
        // if we have a validator, it has higher priority than our own event table
        if ( TryValidator(event) )
            return true;
        ...
    }
    ...
}
{% endhighlight %}
深入查看：
{% highlight cpp  %}
bool wxWindowBase::TryValidator(wxEvent& wxVALIDATOR_PARAM(event))
{
#if wxUSE_VALIDATORS
    // Can only use the validator of the window which
    // is receiving the event
    if ( event.GetEventObject() == this )
    {
        wxValidator *validator = GetValidator();
        if ( validator && validator->ProcessEvent(event) )
        {
            return true;
        }
    }
#endif // wxUSE_VALIDATORS
  
    return false;
}
{% endhighlight %}

最重要的代码是
{% highlight cpp  %}validator->ProcessEvent(event);{% endhighlight %}
接着我们可以通过validator的事件表知道它如何处理这个事件：
{% highlight cpp  %}BEGIN_EVENT_TABLE(wxTextValidator, wxValidator)
    EVT_CHAR(wxTextValidator::OnChar)
END_EVENT_TABLE()
  
void wxTextValidator::OnChar(wxKeyEvent& event)
{
    if ( m_validatorWindow )
    {
        int keyCode = event.GetKeyCode();
        // we don't filter special keys and Delete
        if (
             !(keyCode < WXK_SPACE || keyCode == WXK_DELETE || keyCode > WXK_START) &&
             (
              ((m_validatorStyle & wxFILTER_INCLUDE_CHAR_LIST) && !IsInCharIncludes(wxString((wxChar) keyCode, 1))) ||
              ((m_validatorStyle & wxFILTER_EXCLUDE_CHAR_LIST) && !IsNotInCharExcludes(wxString((wxChar) keyCode, 1))) ||
              ((m_validatorStyle & wxFILTER_ASCII) && !isascii(keyCode)) ||
              ((m_validatorStyle & wxFILTER_ALPHA) && !wxIsalpha(keyCode)) ||
              ((m_validatorStyle & wxFILTER_ALPHANUMERIC) && !wxIsalnum(keyCode)) ||
              ((m_validatorStyle & wxFILTER_NUMERIC) && !wxIsdigit(keyCode)
                                && keyCode != wxT('.') && keyCode != wxT(',') && keyCode != wxT('-') && keyCode != wxT('+') && keyCode != wxT('e') && keyCode != wxT('E'))
             )
           )
        {
            if ( !wxValidator::IsSilent() )
                wxBell();
            // eat message
            return;
        }
    }
    event.Skip();
}
{% endhighlight %}
比如对validator设置了wxFILTER_ALPHA属性，只允许输入字母，代码
{% highlight cpp  %}((m_validatorStyle & wxFILTER_ALPHA) && !wxIsalpha(keyCode)){% endhighlight %}

就为true。这就是validator的秘密所在。

由于第二种工作方式使用较少，下面只是简单提及，有兴趣的同学可以仔细阅读源码。

对于validator的第二种工作方式，当一个dialog初始化的时候，会进行如下操作：

{% highlight cpp  %}
bool wxWindowBase::TransferDataToWindow()
  
{
    ...
    for ( node = m_children.GetFirst(); node; node = node->GetNext() )
    {
        if ( validator && !validator->TransferToWindow() )
        {
            ...
        }
    }
    ...
}
{% endhighlight %}

当一个dialog将要关闭的时候，会进行如下操作：
{% highlight cpp  %}
bool wxWindowBase::TransferDataFromWindow()
  
{
    ...
    for ( node = m_children.GetFirst(); node; node = node->GetNext() )
    {
        if ( validator && !validator->TransferFromWindow() )
        {
            ...
        }
    }
    ...
}
{% endhighlight %}
dialog会逐个控件检测数据的合法性：
{% highlight cpp  %}
bool wxWindowBase::Validate()
{
    ...
    for ( node = m_children.GetFirst(); node; node = node->GetNext() )
    {
        wxWindowBase *child = node->GetData();
        wxValidator *validator = child->GetValidator();
        if ( validator && !validator->Validate((wxWindow *)this) )
        {
            return false;
        }
        ...
    }
    ...
}
{% endhighlight %}
如果遇到错误，validator负责弹出小型对话框来警告：

{% highlight cpp  %}
bool wxTextValidator::Validate(wxWindow *parent)
{
    ...
    if ( (m_validatorStyle & wxFILTER_ALPHA) && !wxIsAlpha(val) )
    {
        ok = false;
        errormsg = _("'%s' should only contain alphabetic characters.");
    }
    ...
    wxMessageBox(buf, _("Validation conflict"),
                     wxOK | wxICON_EXCLAMATION, parent);
    ...
}
{% endhighlight %}


#定义满足自己需要的validator
下面只举例说明第一种工作方式。

如果我们觉得wxWidgets提供的validator不够用，我们可以定义一个新的validator：
{% highlight cpp  %}
class MyValidator : public wxValidator
{
    ...
};
{% endhighlight %}
接着改写OnChar函数：
{% highlight cpp  %}  
void CommonValidator::OnChar(wxKeyEvent& event)
{
    ...
    if (!m_pValidatorStrategy->Validate(val))
    {
        ::wxBell();
        return;
    }
    else
    {
        event.Skip();
    }
    ...
}
{% endhighlight %}
m_pValidatorStrategy是CommonValidatorStrategy类型。检验策略的类图如下：

<img width="800px" src="/assets/image/2014-08/wxValidatorUML.png" type="image/png" />

RegexValidatorStrategy是这样实现的：
{% highlight cpp  %}
bool RegexValidatorStrategy::Validate(const wxString& val)
{
    return boost::regex_match(val.c_str(), m_regularExp);
}
{% endhighlight %}
这样我们可以设置不同的策略进行不同的检验，添加新的检验规则，只需要添加一个CommonValidatorStrategy的子类，这样的设计相对灵活。

以上这种方式只是禁止某个字符的输入，功能可能还不够强大。

其实我们还可以对目前已经输入的字符串进行检验：

~~~ { cpp }
void CommonValidator::OnChar(wxKeyEvent& event)
{
    ...
    wxChar ch(keyCode);
    wxString val;
    if (wxTextCtrl* ctrlText = wxDynamicCast(event.GetEventObject(), wxTextCtrl))
    {
        GetTCInputVal(ctrlText, ch, val);
    }
    if (!m_pValidatorStrategy->Validate(val))
    {
        ::wxBell();
        return;
    }
    else{
        event.Skip();
    }
    ...
}

~~~
我们通过6-9行得到控件里面的字符串，接着在第10行检验其合法性。

关于validator的使用，这里只是抛砖引玉，欢迎各位同学挖掘更多应用的技巧与我们分享。

