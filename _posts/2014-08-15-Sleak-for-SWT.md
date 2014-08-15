---
layout: post
title: SWT应用内存泄漏Sleak分析
description: SWT应用内存泄漏Sleak分析
category: Java
tags: SWT Sleak
refer_author: YunShui
refer_blog_addr:
refer_post_addr:
---

Sleak 是一个用于检测 SWT 图形资源的简单工具，它能探测到 SWT 应用中资源代码泄露问题。
源码： Sleak.java

简介
===

在使用此工具前，必须把 Sleak 加入到当前工程 Class Path 中，并找到应用程序代码创建 Display 的地方：
{% highlight java  %}
Display display = new Display();
{% endhighlight %}

具体步骤如下：

1. 在创建 Display 之前，创建一个 DeviceData 并设置 data.tracking 标志为 true 。
1. 使用此 DeviceData 创建 Display 。
1. 创建 Sleak 实例并调用其 open 方法。

代码示例如下：
{% highlight java  %}
DeviceData data = new DeviceData();
data.tracking = true;
Display display = new Display(data);
Sleak sleak = new Sleak();
sleak.open();
{% endhighlight %}
运行应用程序代码时， Sleak 也会随之启动，界面如下：

<img width="480px" src="/assets/image/2014-08/sleak-1.png" type="image/png" />

1. Snap 快照当前应用中资源使用情况

    <img width="480px" src="/assets/image/2014-08/sleak-2.png" type="image/png" />

1. Diff 比较当前资源和 Snap 资源中使用情况

    <img width="480px" src="/assets/image/2014-08/sleak-3.png" type="image/png" />

1. Stack 显示资源堆栈调用

    <img width="480px" src="/assets/image/2014-08/sleak-4.png" type="image/png" />


代码示例
===

 RCP 工程代码需修改 Application.java

55 和 56 行为修改 SWT 的 Policy 为 Debug 级别。调用 PlatformUI 的 createDisplay 方法创建 Display 对象，然后创建 Sleak 对象调用其 open 方法。

案例分析
===

Image 示例
---
{% highlight java  %}
private Image getImage(String imageName) {
  Image image = ResourceManager.getPluginImage("com. gui.client.resources",
      "icons/sgmt/" + imageName + ".png");
  if (!model.isVertical ()) {
    return ImageUtil.reducedTo(image,
        (int) ((double) image.getBounds().width / 4),
        (int) ((double) (image.getBounds().height) / 4));
  } else {
    Image nImage = ImageUtil.reducedTo(image,
        (int) ((double) image.getBounds().width / 6),
        (int) ((double) (image.getBounds().height) / 6));
    nImage = ImageUtil.reducedTo(image, nImage.getBounds().width,
        nImage.getBounds().height * 2, false);
    return ImageUtil.rotate(nImage, 90);
  }
}
{% endhighlight %}

该方法根据 Image 名字和 resource 工程中已保存的图片动态创建所需 Size 的 Image ，实现过程中操作包含动态调整图片大小转换、 Image 的旋转等操作。运行时间和打开关闭频率比较低时问题不大，但这些操作比较频繁时，应用会占用大量的内存，系统反应迟钝。这是因为临时创建的 Image 会一直驻留在系统内存中，直到应用进程退出。此 UI 页面的每一次关闭和重新打开，都会创建这些 Image ，造成 Image 内存持续占用。

解决方案可为：

1. 不动态创建 Image ，创建好 Image 放在 resource 工程中，延迟加载。
1. 动态创建 Image ，但使用完关闭。

参考实现如下：
{% highlight java  %}
private Image getSegmentationImage(String imageName) {
  return ImageResource.getImage("icons/sgmt/" + imageName
      + (model.isVertical() ? "_v" : "_h") + ".png");
}

public static final Image getImage(String imagePath) {
  Image image = IMAGE_MAP.get(imagePath);
  if (image == null || image.isDisposed()) {
    image = ResourceManager.getPluginImage("com.gui.client.resources", imagePath);
    IMAGE_MAP.put(imagePath, image);
  }
  return image;
}
{% endhighlight %}
Image 采用 Map 来 Cache ，并延迟加载。

GC 、 Color 、 Font 示例
---

{% highlight java  %}
public class ExampleCanvas extends Canvas {
  private Color inactiveTabColor = new Color(null, 204, 204, 204);
  private Color activeTabColor = new Color(null, 134, 206, 244);
  private Color borderColor = new Color(null, 255, 255, 255);
  private Color fontColor = new Color(null, 255, 255, 255);

  private Font activeFont = null;
  private Font inactiveFont = null;

  public AdvNavBar(Composite parent, int style) {
    // ...
    addDisposeListener(new DisposeListener() {
      @Override
      public void widgetDisposed(DisposeEvent e) {
        inactiveTabColor.dispose();
        activeTabColor.dispose();
        borderColor.dispose();
        fontColor.dispose();
      }
    });
    // ...
  }

  @Override
  public Point computeSize(int wHint, int hHint, boolean changed) {
    GC gc = new GC(this);
    // ...
    gc.dispose();
    return new Point(width, height);
  }

  @Override
  public void dispose() {
    inactiveTabColor.dispose();
    activeTabColor.dispose();
    borderColor.dispose();
    fontColor.dispose();
    if (activeFont != null) {
      activeFont.dispose();
    }
    if (inactiveFont != null) {
      inactiveFont.dispose();
    }
    super.dispose();
  }
}
{% endhighlight %}
示例处理方法为：

1. 增加 Dispose 监听回收 Color 资源
1. 使用完 GC 调用 GC.dispose() 方法回收 GC
1. 覆盖父类 dispose 方法回收 Font 和 Color 资源

总结
===
SWT 框架内核代码也有资源泄漏问题，所以我们在定义外部资源时更应该做回收处理，以免造成不必要的资源内存消耗。典型情况如下：
{% highlight java  %}
ResourceManager.getPluginImage("resources", "icons/error_12x12.png");
GC gc = new GC(this)
Color color = new Color(null, 204, 204, 204);
Font f ont = new Font(null, gc.getFont().getFontData()[0]);
{% endhighlight %}
书写代码时应该特别注意。

