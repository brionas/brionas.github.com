---
layout: post
title: wxWidgets Sizing & Sizer
description: 介绍wxWidgets 中的Size 和Sizer
category: wxWidgets
tags: [wxWidgets]
refer_author: Jeremy Peng
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

wxWidgets Sizing & Sizer
========================

我们在学习和使用wxWidgets时，经常会被其中的各种Size (size、client、best
size …) 及各种Sizer(wxBoxSizer、wxGridSizer) 搞得晕头转向;
而且它们相关有些函数如Fit， Layout 等在wxWindow
和Sizer类中都有实现，而在做编码过程中也时常不清楚哪种情况下应该用哪个类的哪个函数，而干脆”宁可杀错不可放过”胡乱调用一通，这样往往会得到让人”意外”的结果。如果亲们和我一样具有类似经历，那么我们就很有必要对这些知识进行梳理和总结了。最后我会介绍一种，
在Dialogblock中利用wxGridBagSizer 实现类似于VS拖拽的界面设计方法。 
Let’s go！！ 

目录
----

一、  wxWidgets Sizer简单介绍  

二、  wxWidgets 的各种Size及相关函数  

三、  Dialogblock设计新方法  

一. wxWidgets Sizer简单介绍
------------------

1.1 Sizer具有的优点
-------------------

        首先我们讨论下为什么需要Sizer,总结起来主要有以下三点优点:

         1. 平台无关, 我们不需要考虑不同的控件在不同平台显示的大小差异;

         2. 灵活, 利用Sizer我们可以方便设计出复杂的布局结构;

         3. 简单,
在改变窗口大小时,我们不需要为窗口内控件添加额外处理代码,即可实现控件大小随窗口大小改变而改变.

      ![](/assets/image/2014-01/size_files/2.jpg)

  具有这些好特性,当然也不是wxWidgets这个图形库所特有,Qt里也有Layout的概念,具体它们谁山寨谁,我们不去深究.但把wxWidgets和Qt的”Sizer”放在比较我们就会发现它们具有很多类似的地方.

            ![](/assets/image/2014-01/size_files/3.jpg)

1.2 Sizer的一般特性
-------------------

wxWidgets具有多种Sizer,但是它们的公共特性一些概念是通用的.我们一起回忆下下书上对于这些特性的描述.

**● minimal size:**
布局控件中的每个元素都有计算自己的最小大小的能力(这往往是通过每个元素的DoGetBestSize函数计算出来的).这是这个元素的自然大小.举例来说,一个复选框的自然大小等于其复选框图形的大小加上其标签的最合适大小.                                                         

**● border:** 用于各个独立控件的间距大小,
我们在进行Sizer::Add可以通过**wxTOP, wxBOTTOM, wxLEFT, wxRIGHT,
wxALL**进行设置.

**●alignment:** 对齐方式, 设置在Sier中的对齐方式,
主要有上下左右,居中等对齐.
对齐既可以是水平方向的也可以是垂直方向的,但是对于大多数布局控件来说,只有一个方向是有效的.比如对于水平布局控件来说,只有垂直方向是有效的,因为水平方向的空间是被所有的子元素分割的,因此设置水平对齐方式是没有意义的(当然,为了达到水平对齐的效果,我们可能需要插入一个水平方向的空白区域,关于这点我们不作太多的说明).

●**stretch factor:**
伸缩因子,如果一个布局控件的空间大于它所有子元素所需要的空间,那么我们需要一个机制来

分割多余的空间.为了实现这个目的,布局控件中的每一个元素都可以指定一个伸缩因子,如果这个因子设置为默认值0,那么子元素将保持其原本的大小,大于0的值用来指定这个子元素可以分割的多余空间的比例,因此如果两个子元素的伸缩因子为1,其它子元素的伸缩机制为0,那么这两个子元素将会各占用多余空间的50%的大小.


Sizer的通用特性介绍完了,下面我们开始对各个Sizer进行简单介绍,真的会很简单,主要是介绍了下各个Sizer特地及作用,和一些常用的函数,
其余的很多内容请自觉参考手册.

### 1.3 Sizer介绍

      首先我们看下wxWidgets提供给我们的多种Sizer的继承关系图,这样我们对各个Sizer会有个大体的了解:
![](/assets/image/2014-01/size_files/4.png)
    

   wxSizer是所有sizer的基类, 它是一个抽象类,我们一般是使用其子类:
wxBoxSizer,wxStaticBoxSizer,wxGridSizer, wxFlexGridSizer,
wxGridBagSizer,因此不对其做过多介绍.

**● wxBoxSizer:** 最基础的一个sizer,
具有水平及垂直两种方式,通过在构造函数中传递`wxVERTICAL`或
`wxHORIZONTAL`初始化.  构造函数为`wxBoxSizer(int orient);`
 例: `wxBoxSizer *sizer =new wxBoxSizer(wxHORIZONTAL);`

在初始化后,通过Add 函数将控件添加进Sizer中.

	     wxSizerItem* Add(wxWindow* window, int proportion = 0, int flag =
	0, int border = 0)
	
	     window:需要添加进Sizer的控件, 也可以是其他Sizer;
	
	     proportion: 同2.2节讲的stretch factor,伸缩因子
	
	     flag:标志位,决定控件在Sizer中的表现形式,具体可参考手册.
	
	     border:同2.2节介绍的border.

我们可以创建出嵌套的Sizer布局方式,如下: 

![](/assets/image/2014-01/size_files/5.jpg)
 

**● wxStaticSizer:** 基本同wxBoxSize一样,
只是在原本不可见的Sizer外框,添加一个Static Label, 给人更清晰的提示.

    ![](/assets/image/2014-01/size_files/5.png)
   

*● wxGridSizer:*提供一个N行N列的布局区域, 期间每个单元的大小是一样的.

`wxGridSizer(int rows, int cols, int vgap, int hgap)`

  
构造一个rows行cols列的Sizer矩阵.vgap及hgap定义了其中各个控件的间距.** **

     ![](/assets/image/2014-01/size_files/6.png)

**●wxFlexGridSizer:**
有时我们需要在N行N列的wxGridSizer中的某行,某列或者某个单元,具有不同的大小,
这时候wxFlexGridSizer就派上用场了.

`wxFlexGridSizer(int rows, int cols, int vgap, int hgap) ;`

构造函数基本同wxGridSizer, 但我们可以通过:
	
	void AddGrowableCol (size_t idx, int proportion = 0);
	
	void AddGrowableRow (size_t idx, int proportion = 0);

改变指定行列的元素可以具有随窗口增长的能力.如设置:

	AddGrowableRow(1);
	
	AddGrowableCol(2);

我们可以使第二行,第3列的按钮随窗口改变大小.  

![](/assets/image/2014-01/size_files/7.png)

**●wxGridBagSizer:** 这个Sizer从wxFlexGridSizer继承,
wxFlexGridSizer只有在添加控件才能显示排布方式不同,
wxGridBagSizer提供虚拟的网格,使得我们能在这些网格中通过wxGBPosition及wxGBSpan改变任意改变控件的位置和大小.构造函数:

`wxGridBagSizer (int vgap = 0 , int hgap = 0 )`

我们可以通过`void SetEmptyCellSize ( const wxSize& sz)`
函数设置网格中空白单元的大小,如下添加10\*10网格.

    ![](/assets/image/2014-01/size_files/8.png)

通过函数
`wxSizerItem *Add(wxWindow* window, const wxGBPosition& pos,
const wxGBSpan& span = wxDefaultSpan, int flag = 0, int border = 0,
wxObject* userData =
NULL)`,我们向GridBagSizer中添加元素.下面演示在上述的单元网格中添加按键的:

`Add(itemButton4, wxGBPosition(5, 4), wxGBSpan(1, 2),
wxALIGN_CENTER_HORIZONTAL| wxALIGN_CENTER_VERTICAL| wxALL, 5)`

         ![](/assets/image/2014-01/size_files/9.png)\

     Sizer的简单介绍就到这里,
具体每个Sizer的使用方法,大家最好还是参考手册.

二.  Window 的各种Size
----------------------

### 2.1 wxWidgets 的Size

wxWidgets中有许多的Size，
要清楚理解它们的具体含义，而不再编码时造成混淆，
是我们设计及编写GUI程序的基础. 其中主要的Size有:

​1. Size

​2. Client Size

​3. Best Size

​4. Best Client Size

​5. Minimal & Maximum Size

​6. Initial Size

​7. Virtual Size 

 ● “Size”: 最基础也最好理解， 按官方文档的介绍----this is the current
size of the window and it can be explicitly set or fetched with the
[wxWindow::SetSize()](http://docs.wxwidgets.org/trunk/classwx_window.html#a180312d5ad4a4a5ad805b8d52d67a74e)
or
[wxWindow::GetSize()](http://docs.wxwidgets.org/trunk/classwx_window.html#a124c12cff1e7b6e96a5e1fd3e48dca34)
methods. 它是当前窗口的大小， 可以通过GetSize及SetSize获得及设置.

 ● “Client Size”: the client size represents the widget's area inside of
any borders belonging to the widget and is the area that can be drawn
upon in a EVT\_PAINT event. 这个字面的理解就是在边框内的大小.
对于wxFrame， 就是除掉了Menu， tool Bar 及status
bars，且没有”边框”的大小.

我们在初始化窗口时经常会通过style参数设置窗口的显示风格(需不需要标题、最大、最小按钮等)，
那么假设我们在构造window时设置Style = 0，没有了标题， 最大最小按钮，
那么我们会得到怎么样的Size 和Client Size 呢？
先别急着说会得到Size和Client Size 相等的结果，我们通过实际程序测试下。

`wxFrame *dialog = new dialog(this,ID_DIALOG_SIZE,_T("Frame
Size"),wxDefaultPosition,wxSize(200,200),0 );`

         假设我们有如上构造函数，注意到我们设置Style = 0，
那么运行程序，我们可以得到下面没有了标题,菜单,状态栏的对话框,那么此时添加左键单击事件,通过GetSize及GetClientSize获取当前的Size及ClientSize.

![](/assets/image/2014-01/size_files/1.jpg)

  
          出乎意料, 就算没有了菜单,状态栏及标题, Size依然不等于Client
Size, 那么究竟什么原因呢?

        
眼尖的你,可能会发现原来那个对话框外面还是有两条线,构成了对话框的borders,
上下左右各1像素, 所以最终显示的Size是200,200, 而ClientSize 是198,198.
原来通过设置`style=0`并不能使得`Size==ClientSize`. 我们需要通过设置`style =wxBORDER_NONE`,使得窗口四周的黑线都去掉，这样才能得到`Size==ClientSize`的结果.

         继续讨论我们的Size:

         ● “Best Size”: the best size of a widget depends on what kind
of widget it is, and usually also on the contents of the widget.
这个Best Size可以理解为显示的最好大小, 根据不同的窗口(Frame,Dialog, MID
Fame)及不同的控件,具有不同的Best Size, 通常通过DoGetBestSize 计算所得.

         DoGetBestSize是wxWindowBase的一个虚函数,
所有继承自wxWindow的类都需要重写这个函数.我们来看看这个函数具体干了什么(代码比较长,不全部贴出,只抽取出其中的主要部分,有兴趣可以自行参考wxWidget源码,
我的源码版本是2.8.12,这篇文章参考的源码都是这个版本的): 


	virtual wxSize DoGetBestSize () const;
	
	wxSize wxWindowBase :: DoGetBestSize() const
	
	{
	
	     …
	
	     if( has sizer )
	
	           BestSize = sizer->GetMinSize ()  +  non-client
	
	     else if( has child window)
	
	            BestSize  = fitting window Size + non-client
	
	      else
	
	            BestSize = GetMinSize() | GetSize()
	
	     …
	
	}


 从源码我们可以看出, wxWindow在计算Best
Size时会根据不同的情况计算不同的best size:

a)假设窗口有一个sizer，那么获取sizer的minSize。也就是能容纳sizer里面所有子窗口的最小的size，然后加上non-client的size，就是窗口期望的best
size，因为这样我们就能看到它所含的sizer里面的所有的子窗口。

​b)
考虑自身没有sizer，但是有子窗口的情况,loop所有的子窗口，计算它们的size，overlap所有的窗口矩形，求解最小的fitting
window size。针对这种情况，还需要加上non-client 的size.

​c) 最后一种情况，就是没有子窗口，是一个树叶窗口节点。

首先它获取minsize，如果是个有效的size(width != -1 && height !=
-1)，那么就直接返回它。否则就用GetSize返回的结果。

而对于不同的控件,它们都重写了DoGetBestSize这个虚函数,
对于Button控件是显示所有字符Label的合适大小,
对于wxListBox则根据具有的item计算显示大小.
从下面wxButton重载的DoGetBestSize我们看出wBtn
及hBtn会根据具体的Char的宽度和高度计算BestSize 的方法和原理. 

	
	wxSize wxButton::DoGetBestSize() const
	
	{
	
	    wxClientDC dc(wx_const_cast(wxButton *, this));
	
	    dc.SetFont(GetFont());
	
	
	
	    wxCoord wBtn,
	
	            hBtn;
	
	    dc.GetMultiLineTextExtent(GetLabelText(), &wBtn, &hBtn);
	
	
	
	    // add a margin -- the button is wider than just its label
	
	    wBtn += 3*GetCharWidth();
	
	    hBtn = BUTTON_HEIGHT_FROM_CHAR_HEIGHT(hBtn);
	
	
	
	    // all buttons have at least the standard size unless the user explicitly
	
	    // wants them to be of smaller size and used wxBU_EXACTFIT style when
	
	    // creating the button
	
	    if ( !HasFlag(wxBU_EXACTFIT) )
	
	    {
	
	        wxSize sz = GetDefaultSize();
	
	        if (wBtn > sz.x)
	
	            sz.x = wBtn;
	
	        if (hBtn > sz.y)
	
	            sz.y = hBtn;
	
	        return sz;
	
	    }
	
	    wxSize best(wBtn, hBtn);
	
	    CacheBestSize(best);
	
	    return best;
	
	}


● ”Best Client Size”: 这个很好理解, 就是在计算完Best
Size后,除去Border的size的大小. 同Best Size关系,如同Client
Size及Size的关系.

● ”Minimal & Maximum Size”: 最小及最大尺寸,
这两个Size定义了窗口的最小及最大的显示尺寸,
我们在改变窗口大小时,改变的大小需要在这个最大最小范围内.
它们可以通过[wxWindow::](http://docs.wxwidgets.org/trunk/classwx_window.html)[SetMinSize](http://docs.wxwidgets.org/trunk/classwx_window.html)[()](http://docs.wxwidgets.org/trunk/classwx_window.html)
及[wxWindow::](http://docs.wxwidgets.org/trunk/classwx_window.html)[SetMaxSize](http://docs.wxwidgets.org/trunk/classwx_window.html)[()](http://docs.wxwidgets.org/trunk/classwx_window.html)分别设置,
或者通过[wxWindow::](http://docs.wxwidgets.org/trunk/classwx_window.html)[SetSizeHints](http://docs.wxwidgets.org/trunk/classwx_window.html)[()](http://docs.wxwidgets.org/trunk/classwx_window.html)一起设置.

● ”Initial Size”:即是窗口的初始化大小,
如果我们在构造窗口时有通过size参数传入,
则使用传入的大小作为窗口的初始化大小. 而如果没有定义size,
或者定义不完整如wxSize(150,-1), 那么则会利用Best Size的大小作为Initial
Size, 对于wxSize(150,-1)则会设置为wxSize(150, BestSize.height).

● “Virtual Size”: the virtual size is the size of the potentially
viewable area of the widget. 字面理解为虚拟尺寸,
即是我们设置整个”画布”大小(请允许我做这个比喻, 我们通过”窗口”
大小改变和移动, 变换可视的大小和范围区域).Virtual Size并非只有Scroll window所独有， 但是非Scroll window 尽量不要设置Virtual Size， 否则可能引起一些窗口内控件显示不全的bug。

至此,我们的Size基本梳理完毕,希望各位看官对各个Size的会有更加清晰的理解,
如果想有更深入的理解可以参考官方文档[http://docs.wxwidgets.org/trunk/overview\_windowsizing.html](http://docs.wxwidgets.org/trunk/overview_windowsizing.html)及阅读相关的源码.

### 2.2 Size相关函数
        
上一节我们几种Size,下面选取几个常用的Size相关的函数进行介绍,这些函数在wxWindow及wxSizer中名字是相同的,
使用时经常会造成混淆和不解, 所以很有必要拿出来一起比较说明.

●wxWindow::Fit(void): this method sets the size of a window to fit
around its children.
这个窗口调用这个函数使得窗口大小刚好能容纳他的子窗口的best size,
如果没有子窗口,那么调用这个函数不起作用.

●wxSizer::Fit(wxWindow \*window): Tell the sizer to resize the *window*
to match the sizer's minimal
size.这个函数由Sizer调用,传入一个window指针作为参数.
如果这个Sizer是通过wxWindow::SetSizer函数设置的,那么调用这个函数的效果等同于调用wxWindow::Fit.
应尽量保持窗口通过SetSizer设置的Sizer,
与wxSizer::Fit传入的窗口指针的协调.如frame-\>SetSizer(top\_sizer); 
top\_sizer-\>Fit(frame).

●wxWindow::Layout(void):如果这个窗口有sizer,那么sizer将在窗口现在具有的空间内调用wxSizer::Layout.如果窗口没有Sizer而有layout
constraints
,那么将调用constraints算法,对窗口元素进行排列.(注:这个constraints
文档已建议不要使用, 而用Sizer代替).

●wxSizer::Layout(void):当Sizer内进行添加、删除控件或窗口操作后，调用该函数更新Sizer 的布局。

●wxWindow::SetSizeHints: 这个函数在介绍”Minimal & Maximum Size”时提及,
它是用来设置,最小及最大窗口尺寸的.
建议使用[wxWindow::](http://docs.wxwidgets.org/trunk/classwx_window.html)[SetMinSize](http://docs.wxwidgets.org/trunk/classwx_window.html)[()](http://docs.wxwidgets.org/trunk/classwx_window.html)
和[wxWindow::](http://docs.wxwidgets.org/trunk/classwx_window.html)[SetMaxSize](http://docs.wxwidgets.org/trunk/classwx_window.html)[()](http://docs.wxwidgets.org/trunk/classwx_window.html)代替.这个函数不允许非toplevel窗口（wxDialog或wxFrame）调用。这个函数虽然和wxSizer中的SetSizeHints同名,但是功能不相同, wxWindow的SetSizeHints是用来设置最大、最小尺寸的,而wxSizer功能随后介绍。  

●wxSizer::SetSizeHints(wxWindow \*window):调用这个函数首先会调用wxSizer::Fit ,这样传递的window就会调整窗口大小刚好容纳子窗口；然后调用wxWindow::SetSizeHints设置最大和最小尺寸。通常只有像wxFrame或者wxDialog,这种继承自”wxTopLevelWinodw”的顶层窗口,函数调用才有效.一般的窗口或者控件调用无作用. 这个函数比较有用, 我们在设计好窗口控件及排列后, 可以通过调用这个函数, 使得我们的窗口满足大小要求. 可以说wxSizer版的SetSizeHints是wxWindow的升级加强版，具有调整窗口大小的功能。  


三. Dialogblock设计新方法
-------------------------

工欲善其事，必先利其器。在MFC上开发，有VS系列强大的可视化工具，基于wx也有许多辅助UI设计的工具软件。我们这里使用的是Dialogblock， 详细的Dialogblock介绍，大家可以参考官方[网址](http://www.anthemion.co.uk/dialogblocks/)。

wxWidgets不同于微软提供的MFC, 不可以在设计界面时,利用简单的拖动进行完成.虽然我们有一些wxWidgets的UI设计软件提供帮助,但是包括常用的Dialogblock这些设计软件,都不能简单通过拖拽完成界面设计.有时为了使控件具有合适的大小,并在合适的位置放置,我们必须通过不断调整参数进行尝试.

但是在尝试和探索中,我发现了一种在Dialogblock中也能进行拖动控件的设计方法,从而实现类MFC的界面设计方法.这种方法主要是结合wxGridBagSizer的网格特性及利用spacer占位功能.  

下面首先对这种方法进行简单演示,我们要设计如下一个具有textCtrl及button的对话框: 

![](/assets/image/2014-01/size_files/10.png)  
 
1. 添加wxGridBagSizer, 勾选窗口中Fit to content属性,使得窗口随控件改变大小.

![](/assets/image/2014-01/size_files/11.png)  
2. 在GridBagSizer中添加Spacer,并将Spacer的Grid X,y设置为大概的窗口大小(之后添加控件窗口大小会改变).

![](/assets/image/2014-01/size_files/12.jpg)  
3. 开始添加控件如Button,Text , 并拖动控件到窗口合适的位置.

![](/assets/image/2014-01/size_files/13.png)   
4. 继续添加控件, 如TextCtrl 控件,调整控件大小和占用Sizer数 Span.

![](/assets/image/2014-01/size_files/14.jpg)  
5. 继续调整按钮位置及大小,如果调整完成,则移动Spacer到适当位置,改变窗口大小.运行测试窗口.

![](/assets/image/2014-01/size_files/15.jpg)  
6. 如果希望控件随窗口改变大小,可以通过设置GrowableColumns及GrowableRows指定需要增长的控件大小,如下处于Grid(1,1)位置的文本框及Grid(1,9)的Button随窗口Expand.

![](/assets/image/2014-01/size_files/16.jpg)  
7. Gridbag Sizer 中还能添加其他Sizer这样布局就可以做到更复杂了,以下窗口还添加了水平的wxBoxSizer,这样我们可以设计出复杂多变的界面布局.

![](/assets/image/2014-01/size_files/14.png)

总结下具体的步骤:

  1.Add wxGridbagSizer

  2.Add Spacer

  3.Add Controls and set their size and position.

  4.Add other Sizer if you want more complex design

  5.Set Growable Colums and Row if need.

  6.Drag the Space to the right place to set the final window size. 

      当然设计不同的界面具有不同方法,
我们用前面介绍的Sizer(除wxGridBagSizer)组合使用一样能设计出同样的界面.
这里主要是给大家介绍的一种新的设计方法和思路,
希望你在以后的工作中能想起还有这种方法,
并通过方法给你的界面设计程序带来便利, 那我就感觉很满足了. 

总结
----

通过这篇文章希望读者能对wxWidgets中的各种Size和Sizer有一个较为清晰的理解.
我也通过这篇文章对之前的学习做一次简单总结,如果您发现文章有什么错漏的地方,请及时和我联系:
[jeremy.peng@asml.com](mailto:jeremy.peng@asml.com) 谢谢!

