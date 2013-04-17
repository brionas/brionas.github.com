---
layout: post
title : 从组合框控件看SWT与JFace的区别
description : Combo是SWT的组合框，ComboViewer是JFace的组合框，都是组合框，ComboViewer其实就是在Combo上面加上MVC的封装。记住下面的公式
category : SWT/JFace
tags : [SWT,Java]
---
{% include JB/setup %}

**【感谢 wenzhe 投稿,原文[链接](http://blog.csdn.net/liuwenzhe2008/article/details/8670757)】**

Combo是SWT的组合框，ComboViewer是JFace的组合框，都是组合框，ComboViewer其实就
是在Combo上面加上MVC的封装。记住下面的公式：

    JFace = SWT + MVC

比如说要实现如下图的任务，选中一本书，显示它的价格。

![jface1](/assets/image/jface1.png)


首先定义领域类（Domain Class），如下所示：


<pre>
/**
 * @author liuwenzhe2008@qq.com
 */
public class Book {
  private String name = "";
  private int price = 0;
  
  public Book(String name, int price) {
    this.name = name;
    this.price = price;
  }
  public String getName() {
    return name;
  }
  public int getPrice() {
    return price;
  }
}
</pre>

现在有个书架，摆了好几本书：

<pre>
  Book[] books = new Book[] {
      new Book("Java", 20),
      new Book("SWT", 30),
      new Book("JFace", 40),
      new Book("Eclipse RCP", 50)
  };
</pre>

下面第一种方案采用SWT的Combo：

<pre>
final Combo combo = new Combo(shell, SWT.READ_ONLY);
combo.setLayoutData(new GridData(SWT.FILL, SWT.CENTER, true, false, 1, 1));
</pre>

把书架的书放在combo里面：

    for (Book book : books) {
      combo.add(book.getName());
    }

考虑到每一本书有不同的价格，换另一本书时就需要在更新价格控件，这样需要为Combo增加事件响应：

    combo.addSelectionListener(new SelectionAdapter() {
      @Override
      public void widgetSelected(SelectionEvent e) {
        int index = combo.getSelectionIndex();
        Book book = books[index];
        text.setText(Integer.toString(book.getPrice()));
      }
    });

可以看到，SWT的Combo主要是通过索引（index）来访问item的，这样不够直接，也不够方便。
下面看看JFace的ComboViewer是怎么实现以上功能的。

首先定义一个ComboViewer控件：

    ComboViewer comboViewer = new ComboViewer(shell, SWT.NONE);

需要的话，你也可以从ComboViewer中拿到SWT的Combo控件：

    Combo combo = comboViewer.getCombo();
    combo.setLayoutData(new GridData(SWT.FILL, SWT.CENTER, true, false, 1, 1));

由于JFace采用MVC模式，需要我们先回答3个问题：

* 1、你要往控件输入（Input）什么东西？这相当于MVC模式中的模型（Model），
可以是任意引用类型（即Object及其子类，只要不是int这些基本类型就行）。

这里我们需要往comboViewer输入书架上的书本，即：

    comboViewer.setInput(books);

* 2、控件如何组织它的内容？也就是需要找一个内容提供者（ContentProvider），
这相当于MVC模式中的控制器（Controller），必须实现IContentProvider接口。

作为输入的书本已经是数组了，因此我们可以考虑使用JFace专门为数组（array，List，Vector）
提供的实现类ArrayContentProvider，即：

    comboViewer.setContentProvider(new ArrayContentProvider());

* 3、控件如何显示？也就需要找一个标签提供者（LabelProvider），这相当于MVC模式中的视图（Viewer），
必须实现ILabelProvider接口。

这里我们只需要显示书名就可以了，即：

    comboViewer.setLabelProvider(new LabelProvider() {
      @Override
      public String getText(Object element) {
        assert element instanceof Book;
        Book book = (Book)element;
        return book.getName();
      }
    });

好啦，通过回答以上3个问题，总算把书架上的书放到ComboViewer上了。累吧？你是不是觉得还
不如SWT的Combo方便，一个for循环就搞定了，MVC搞得这么麻烦，是吧？有得必有失，有失必有得，
MVC给我们带来更多的灵活性和可扩展性。先喝杯咖啡吧，听我慢慢道来，待会你会永远抛弃Combo，
而衷心地爱上ComboViewer的。废话少说，我们先继续吧。

同样地，考虑到每一本书有不同的价格，换另一本书时就需要在更新价格控件，这样需要为ComboViewer
增加事件响应：

    comboViewer.addSelectionChangedListener(new ISelectionChangedListener() {
      @Override
      public void selectionChanged(SelectionChangedEvent event) {
        ISelection selection = event.getSelection();
        assert selection instanceof StructuredSelection;
        StructuredSelection ss = (StructuredSelection)selection;
        Object obj = ss.getFirstElement();
        if (obj instanceof Book) {
          Book book = (Book)obj;
          text.setText(Integer.toString(book.getPrice()));
        }
      }
    });

看起来比SWT Combo的事件响应要复杂点，但你发现没有，这里不再通过索引（index）来访问元素（item）
，而是直接访问。你可以会想，这又有什么所谓呢？

这么简单的需求，现在当然用哪种方法都无所谓。但客户总是得寸进尺的，软件总是善变的，好的软件设计一定要拥抱变化。
好啦，客户A提出第一个改进的需求：他看书喜欢带书名号，如“《SWT》”。
采用SWT方案的工程师说简单，只要在把书架的书放进Combo时加上书名号就OK了，即：

combo.add("<<" + book.getName() + ">>");

采用JFace方案的工程师也说简单，这个只是显示的问题，就交给视图（Viewer）吧，只要在LabelProvider函数输出加书名
号就OK，即：

    comboViewer.setLabelProvider(new LabelProvider() {
      @Override
      public String getText(Object element) {
        assert element instanceof Book;
        Book book = (Book)element;
        return "<<" + book.getName() + ">>";
      }
      
每个人的兴趣爱好都是不同的，那问题来了，客户B认为简单才是最美，就是不喜欢显示书名号，但考虑到客户A的感受，
希望我们加一个复选框（CheckBox），勾上显示书名号，不勾就不显示书名号。

![jface2](/assets/image/jface2.png)

这时候SWT方案就头痛了，那把书架的书放进Combo是究竟要不要加书名号，两个客户都不能得罪啊。不要怪客户的需求太BT，
这就是SWT方案没有视图的后果。

JFace方案笑了，可以很轻松的解决：

    comboViewer.setLabelProvider(new LabelProvider() {
      @Override
      public String getText(Object element) {
        assert element instanceof Book;
        Book book = (Book)element;
        if (btnShowBookMark.getSelection()) {
          return "<<" + book.getName() + ">>";
        } else {
          return book.getName();
        }
      }

客户C来了，他希望书名可以排序，比较书"Eclipse"总是排在"SWT"前面。但为了兼顾其他客户，
我们需要再加一个复选框（CheckBox），勾上为排序，不勾不排序。

![jface3](/assets/image/jface3.png)

这个需求对于SWT方案又是重拳一击，他快晕倒了，这就是依赖于索引（index）的结果。

JFace方案又笑了，简单啊，谁叫JFace控件天生就有比较器（Comparator）呢？看看下面的吧，
该羡慕嫉妒恨了吧。

    comboViewer.setComparator(new ViewerComparator() {
      @Override
      public int compare(Viewer viewer, Object e1, Object e2) {
          return 0;
        if (!btnSort.getSelection()) {
        }
        return super.compare(viewer, e1, e2);
      }
    });

客户D来了，她是个富婆，看不起穷人，只卖贵的，不买对的，她不希望看到35块钱以下的书
（包括35块也不行），把这些书通通过滤调。但为了兼顾一下像我们这样的苦孩子，
我们需要再加一个复选框（CheckBox），勾上为过滤，不勾不过滤。

![jface4](/assets/image/jface4.png)

这个需求对于SWT方案更加的BT，SWT方案这回彻底疯掉了，为什么有这么多刁钻的客户呀？
JFace方案还是笑了，这样的客户多多益善啊，谁叫JFace控件天生就有过滤器（Filter）呢？

    comboViewer.addFilter(new ViewerFilter() {
      @Override
      public boolean select(Viewer viewer, Object parentElement, Object element) {
        if (!btnFilter.getSelection()) {
          return true;
        }
        assert element instanceof Book;
        Book book = (Book)element;
        return book.getPrice() > 35;
      }
    });
    
看看上面的实现吧，SWT方案们，JFace的大门永远为你们打开的，过来学学JFace吧！