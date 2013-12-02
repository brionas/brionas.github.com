---
layout: post
title: 颜色集算法浅议
description: 颜色集算法
category: 算法
tags: [颜色集]
refer_author: YunShui
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

颜色生成在项目运用中比较常见，项目中自定义的固定颜色库很容易被消耗殆尽，必须选取一个比较合适的算法来生成人眼辨识度大的颜色集。

##1. RGB、HSL颜色空间
###1.1 简介

RGB表示红（Red）、绿（Green）、蓝（Blue）三种颜色按照不同的比例混合，在屏幕上重现256 x 256 x 256种颜色，使用比较广泛。

HSL是工业颜色标准，通过对色调（Hue）、饱和度（Saturation）、亮度（Lightness）三种通道变化得到几乎包括了人类视力所能感知的所有颜色，在屏幕上也可以重现256 x 256 x 256种颜色。

###1.2. RGB、HSL颜色互转
**1.2.1 RGB转换为HSL**
* 把RGB值转成`[0, 1]`数值；
* 找出R、G和B中最大值；
* 计算亮度：L = (maxcolor + mincolor)/2
* 如果最大和最小的颜色值相同，即表示灰色，那么S定义为0，而H未定义并在程序中通常写成0；
* 否则，根据亮度L计算饱和度S：

    * `if L < 0.5, S = (maxcolor-mincolor) / (maxcolor + mincolor)`

    * `if L >= 0.5, S = (maxcolor-mincolor) / (2.0-maxcolor-mincolor)`

* 计算色调H： 

    * `if R = maxcolor, H = (G - B) / (maxcolor - mincolor)`

    * `if G=maxcolor, H=2.0 + (B - R) / (maxcolor - mincolor)`

    * `if B=maxcolor, H=4.0 + (R - G) / (maxcolor - mincolor)`

* H = H x 60.0，如果H为负值，则加360。
Java示例：

{% highlight cpp %}
public static int[] hueToRGB(double hue, double saturation, double lightness) {
  int[] rgb = new int[3];
  
  if (saturation == 0) {
    rgb[0] = (int) lightness * 255;
    rgb[1] = (int) lightness * 255;
    rgb[2] = (int) lightness * 255;
  } else {
    double var2;
    if (lightness < 0.5) {
      var2 = lightness * (1d + saturation);
    } else {
      var2 = (lightness + saturation) - (lightness * saturation);
    }
    
    double var1 = 2 * lightness - var2;
    rgb[0] = (int) Math.round(255d * hueToRGB(var1, var2, hue + (1d / 3d)));
    rgb[1] = (int) Math.round(255d * hueToRGB(var1, var2, hue));
    rgb[2] = (int) Math.round(255d * hueToRGB(var1, var2, hue - (1d / 3d)));
  }
  
  return rgb;
}
{% endhighlight %}

**1.2.3 HSL转换为RGB**
* if S = 0，表示灰色，定义R、G和B都为L；
* 否则，测试L:

    * `if L < 0.5 , temp2 = L x (1.0 + S)`

    * `if L >= 0.5, temp2 = L + S - L x S`

* `temp1 = 2.0 x L - temp2`
* 把H转换到0 ~ 1。
* 对于R、G、B，计算另外的临时值temp3。方法如下：

    * `for R, temp3 = H + 1.0 / 3.0`

    * `for G, temp3 = H`

    * `for B, temp3 = H - 1.0 / 3.0`

    * `if temp3 < 0, temp3 = temp3 + 1.0`

    * `if temp3 > 1, temp3 = temp3 - 1.0`

Java示例：

{% highlight cpp %}
private static double hueToRGB(double var1, double var2, double varHue) {
  if (varHue < 0) {
    varHue += 1d;
  }
  
  if (varHue > 1d) {
    varHue -= 1d;
  }
  
  if ((6d * varHue) < 1d) {
    return (var1 + (var2 - var1) * 6 * varHue);
  }
  
  if ((2d * varHue) < 1d) {
    return var2;
  }
  
  if ((3 * varHue) < 2) {
    return (var1 + (var2 - var1) * ((2d / 3d - varHue) * 6));
  }
  
  return var1;
}
{% endhighlight %}

##2. 随机颜色生成

采用随机数生成RGB颜色，可调节R、G、B基数使颜色的可辨识度大，但不能保证生成的颜色间差异性。
Java示例：

{% highlight cpp %}
int red = ((int) Math.round(new Double(Math.random() * 128))) + 128;
int green = ((int) Math.round(new Double(Math.random() * 128))) + 128;
int blue = ((int) Math.round(new Double(Math.random() * 128))) + 128;
{% endhighlight %}

得到R、G、B颜色后可根据1.2.1小节转换成HSL颜色空间，也可以随机生成HSL颜色空间。

##3. 互补色集

###3.1 互补色生成步骤

* 转换颜色为HSL；
* 转换Hue为角度值（0⁰ ~ 360⁰），得到Hue值的相反值（如：50⁰ 相反值为230⁰）；
* 保持S和L值，转换为回原颜色空间

##3.2互补色集设计

* 根据互补色思路可知，如果算法固定S、L值，只调节H值：
* 初始值H为0；
* 下一个值为0⁰ 的互补色，即为180⁰，此时色盘被划分为2个部分。如果下一个值的互补色已经被使用，颜色的选取为距离上一个颜色值最远的最大空间中值。

Java实现如下：


{% highlight cpp %}
public synchronized int[] getNext(String key) {
    int[] rgb = colorMap.get(key);
    if (rgb != null) {
        return rgb;
    }
    
    index ++;
    double hue = getNextHueValue();
    rgb = ColorConverter.toRGB(hue, hsl[1], hsl[2]);
    colorMap.put(key, rgb);
    
    return rgb;
}

private double getNextHueValue() {
    if(ranges.isEmpty()) {
        init();
        ranges.add(new Range(0.0, 0.5));
        ranges.add(new Range(0.5, 1.0));
    }
    
    if (index == 1) {
        return lastValue = 0;
    } else if (index == 2) {
        return lastValue = 0.5;
    }
    
    Range range = fetchMaxDistanceRange();
    int idx = ranges.indexOf(range);
    ranges.remove(idx);
    
    double halfRange = range.getDistance() / 2.0;
    ranges.add(idx, new Range(range.getStart() + halfRange, range.getEnd()));
    ranges.add(idx, new Range(range.getStart(), range.getEnd() - halfRange));
    
    return lastValue = range.getStart() + halfRange;
}

private void init() {
    if (hsl != null) return;

    Random rGen = new Random();
    double hue = rGen.nextDouble();
    double saturation = 0.48 + 0.2 * rGen.nextDouble();
    double brightness = 0.72 + 0.08 * rGen.nextDouble();

    hsl = new double[] { hue, saturation, brightness };
}
{% endhighlight %}

色图：

![image](/assets/image/colour-set/colour-graph.png)

随着划分的增加，区间越来越小，颜色的差异性也越来越小。另外此算法中还可动态修改S、L值以获取更多、差异更大的颜色值。

##4. 自定义颜色集

当需要的颜色较多时，互补色的设计思路得到的结果颜色集区分度很小，很难达到要求。一种比较可控的思路是在程序中自定义颜色通道值集，生成颜色集：

* 定义Hue值集为：`[0, 30, 60, 90, 120, 150, 180, 210, 240, 270, 300, 330]`
* 定义Saturation值集为：`[1/4, 2/4, 3/4, 1]`，S值为0时颜色为灰色，区分度小；
* 定义Lightness值集为：`[1/8, 2/8, 3/8, 5/8, 6/8, 7/8]`，L值为0和1时，颜色为白色和黑色，1/2和3/8、4/8的颜色太接近；

根据自定义颜色集，可得到288中颜色值，色图如下：

![image](/assets/image/colour-set/colour-graph2.png)

从上面的结果来还看，生成的颜色人眼区分度比较大，而且颜色可选取288种，满足一般的项目需求。

## 5. 总结

颜色集设计需考虑人眼的可区分度，理论算法中的机器颜色差异性规则在人眼中不一定适用，必须经过大量的数据进行人眼分析，才能得到更恰当的算法。
