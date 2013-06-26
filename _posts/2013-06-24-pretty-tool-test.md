---
layout: post
title: "Pretty tool 介绍"
description: slab提出来是为了解决内部内存碎片的问题，在linux内核中与buddy system一起来解决内核内存管理。但是要看懂slab在linux内核中的实现当前有些困难，我们不如拿些容易阅读的代码来了解slab算法的运作过程。GLIB库实现非常clear，可以做为slab算法的实现学习的入门。
category: "memory_management"
tags: [slab, glib, 源码阅读]
refer_author: Zero
refer_blog_addr: http://zeroli.github.io/
refer_post_addr: http://zeroli.github.io/memory_management/2013/03/24/glib-slab-algo-explain/
---
{% include JB/setup %}

在软件开发调试过程中，经常会去查看某一对象的取值。但类之间复杂的层次关系，再加上数组（链表）、映射（字典）等多种数据结构，让我们难以一目了然。本文介绍的Pretty工具类，以缩进的方式突出类之间的层次关系，并且将对象一层层的整个结构pretty地打印出来！

在编写单元测试时，经常会去比较某一对象是否符合预先的期望值。但对于一个复杂类的对象，这种单元测试并不好写，容易片面化、复杂化。Pretty工具类，既能够完整的检测复杂类的对象，而且可读性好，便于理解代码。当检测出与期望结构不匹配时，不仅可以输出Diff信息，还能提醒用户是否需要自动更新case，简单易用。

本文实现了3种语言版本的Pretty工具类：Java版，Python版，Groovy版。这里对Java版的Pretty做了重点介绍，而其它版本只是简单带过，因为实现的原理都是一样的，只是换成不同语言而已。最后，对于同样广泛应用的C++，本文虽然没有给出具体实现，但也提供了一个设计思路，有兴趣的朋友可以试一试。

##1. Pretty之Java版

##1.1 调试中的问题
当我们在调试程序的时候，经常会查看某一变量的值。一般来说，有两种方法被经常用到：

1.     用调试器，如Eclipse  Debug，或者gdb/pdb。

2.     用print函数或者logger，直接将变量值打印出来。

这两种办法都有缺点，调试器需要一层层展开看，而且如果杯具碰到链表结构或者哈希表的时候，就不太容易看明白了。而print函数其实只是toString方法的返回值，取决于toString函数的实现，其实并不可靠。

可能有聪明的读者会想到，那我们在定义类的时候，都override一下toString方法，让它可读，而不是Object类的缺省实现（JVM中的地址）。

这是一个很天真的想法：

1.     首先，不是所有的类都能够由我们控制，如Java类库，第三方库，其他开发团队的代码，等等。

2.     类的toString方法可能有它的业务价值，而不只是为了方便调试。

3.     额外工作量：这不是必须的，若是其中要求每个类都去override toString方法，会增加很多没有必要的工作量，产生很多不必要的代码，反而会增大维护代码的工作量，甚至引起bug。而且，

当类每次增加/修改/删除成员变量时，都要去修改toString方法，否则print出来的信息也就不可靠了，但这是很难保证的。
##1.2 单元测试中的问题

有下面一个类A，聚合了类B，而B又聚合类C，如下代码：
{% highlight cpp %}

class A {
    private int id;
    private File path;
    private Integer[] array;
    private List<String> list;
    private B b; 
    // ...
} 
class B {
private String desc;
private Map map;
private C c = new C();
// ...
}

class C {
private double v1;
private BigDecimal v2;
// ...
}

{% endhighlight %}

现在我们先测试一下类 A 的对象 a 是不是所期待的，一般容易想到下面几个方法：

把所有成员变量都 get 出来比较：

{% highlight cpp %}
assertEquals(xxx, a.getId());
assertEquals(xxx, a.getPath());
...
assertEquals(xxx, a.getB().getDesc());
...
assertEquals(xxx, a.getB().getC().getV1());
...
{% endhighlight %}
这种办法的问题显而易见：如果不小心漏了一个重要成员变量的get，那测试就不够全面了。而且并不是所有成员变量都有get方法，需不需要get方法得看具体需要，而不能只为unit test专门提供。而且，像这么简单的类，都需要这么多行assertEqual，如果有更多的成员变量或者有很深的聚合层次，那将无法想象。如果A类的结够在做个调整，那改动的地方就很多了。这样的unit test维护成本之高可想而知，还有谁有动力写unit test，因为那是在给自己找麻烦！而且，不是所有的代码我们都能控制的，比如第三方库。

为A类实现equals方法，那assertEquals就只有一个了：

{% highlight cpp %}
A expected = new A();
expected.setId(xxx);
expected.setPath(xxx);
...
expected.setB(new B());
expected.getB().setDesc(xxx);
...
expected.getB().setC(new C());
expected.getB().getC().setV1(xxx);
...
assertEquals(expected, a);
{% endhighlight %}

虽然assertEquals只有一个，但为了建立一个期待的expected作为标尺来比较，需要为提供大量的set方法。这么多set方法带来的问题其实并不比那么多get少。
而且，为A类实现equals方法也是有风险的，因为equals方法本身也需要测试（只要是人写的代码本质上都需要测试！），也需要时间成本。很多类其实没必要去override equals方法。写代码就得维护，没必要写的代码坚决不写，否则维护量更多。同样的，不是所有的代码都能控制的。

##1.3 使用Pretty

Pretty类可以很pretty的解决以上调试和单元测试中的问题。在给出Pretty类之前，先从使用者的角度看看她的pretty：

{% highlight cpp %}

package org.wenzhe.jvlib.debug.test;

import static org.junit.Assert.*;

import java.io.File;
import java.io.IOException;
import java.math.BigDecimal;
import java.text.DateFormat;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.junit.Test;
import org.wenzhe.jvlib.debug.Pretty;

/**
?* @author liuwenzhe2008@qq.com
?*
?*/
public class PrettyTest {
  private static class A {
    private int id = 100;
    private File path = new File("/home/wenzhe/code/pretty");
    private Integer[] array = {86, 755, 1234, 5678};
    private List<String> list = Arrays.asList("My", "name", "is", "Wenzhe");
    private B b = new B("This is my Pretty Test"); 
  }

  private static class B {
    private String desc;
    private Map<String, Date> map = new HashMap<String, Date>();
    private C c = new C();

    public B(String desc) {
      this.desc = desc;
      SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
      try {
        map.put("Today", dateFormat.parse("2013-06-15"));
        map.put("Earth Doomsday", dateFormat.parse("2012-12-21"));
      } catch (ParseException e) {
        throw new RuntimeException(e);
      }
    }
  }

  private static class C {
    private double v1 = 3.14;
    private BigDecimal v2 = new BigDecimal(
        "3.141592653589793238462643383279502884197169399");
  }

  @Test
  public void test1() throws IOException {
    A a = new A();
    assertTrue(Pretty.equalsGolden("test1", a));
  }

  public static void main(String[] args) throws IOException {
    Pretty.setDebugMode(true);
    PrettyTest test = new PrettyTest();
    test.test1();
  }
}

{% endhighlight %}
##1.3.1 Pretty结构

这是一个带有main方法的单元测试类。先撇开单元测试，我们把它当成一个普通java文件来运行（即从main方法开始运行），在屏幕上会打印出对象 a 的pretty结构：
{% highlight cpp %}
org.wenzhe.jvlib.debug.test.PrettyTest$A {
  array : [86, 755, 1234, 5678]
  b : org.wenzhe.jvlib.debug.test.PrettyTest$B {
     c : org.wenzhe.jvlib.debug.test.PrettyTest$C {
     v1 : 3.14
     v2 : 3.141592653589793238462643383279502884197169399
    }
    desc : This is my Pretty Test
    map : {Earth Doomsday=Fri Dec 21 00:00:00 PST 2012, Today=Sat Jun 15 00:00:00 PDT 2013}
  }
  id : 100
  list : [My, name, is, Wenzhe]
  path : pretty
}
{% endhighlight %}

根据pretty结构的缩进，可以很容易看出，对象a的类是： org.wenzhe.jvlib.debug.test.PrettyTest类的内部类A，其成员array是一个数组，值为[86, 755, 1234, 5678]。另一个成员 b 是类 （PrettyTest的内部类B）的对象，b 里面的成员变量c 是类（PrettyTest内部类C）的对象……根据缩进，各种成员变量及其嵌套聚合类的对象也都轻易可见。这在开发调试过程中非常好用！
如果以单元测试的方式运行，屏幕上没有任何输出（No news is Good news），JUnit View中出现大家喜爱的绿色条，祝贺你表测试通过了。（一般对于unittest来说，正确的时候是没输出信息的。）
那么程序怎么知道对象a是期望的呢？注意到第61行，Pretty.equalsGolden("test1", a);  对象a实际上是跟一个名字为test1的golden文件做了比较。这个golden文件的所在的目录为： ${project_root}/src/test/resources/golden/pretty/，这是pretty工具的一个convention，当然也可以改成别的目录，但我不推荐改，很多时候遵从“约定优于配置”的原则总是更好的。打开test1文件，你会发现这也是一个pretty结构，跟之前屏幕上输出的完全一样。

你可能奇怪为什么作为普通java类运行屏幕上会打印，而作为unit test却不会打印呢？其实区别并不在于用哪种运行方式，唯一的区别在于是否选择了Pretty类的debug模式。（一般来说，unit test下不启动debug模式，而在开发调试过程中启动）。注意到main函数刚开始的时候（第65行），debug mode设置为true，当Pretty工具要得到对象a的pretty结构时，会将它打印出来，方便调试，省得在代码里面加入print函数的麻烦。debug mode缺省是关的，所以unit test就没有打印出来了。（有兴趣可以阅读后面的源代码）。
##1.3.2 Pretty Diff

如果unit test测到对象a与golden文件不同，那会怎样？假如有个大老粗不小心把C类中的成员变量v1删除了，又不小心增加了成员变量v3（取值为true），更是不小心把A类的成员变量list里面insert了一个“NOT”，不管是不是在Pretty的debug模式，屏幕上都会输出：

{% highlight cpp %}
Diff from Expected to Actual: 
-:       v1 : 3.14
+:       v3 : true
<:   list : [My, name, is, Wenzhe]
>:   list : [My, name, is, NOT, Wenzhe]
{% endhighlight %}

Pretty工具的错误输出，够pretty吧，大老粗干了哪些坏事这里一目了然。
##1.3.3 Pretty Golden
如果大老粗是故意这么修改的（背后有大老板支持，用软件行业的语言讲就是“需求变了”），那么golden文件也就过时了，需要更新才能让unit test通过。

需要手动更改golden文件吗？我可不干！因为Pretty让我越来越懒了。

懒人都喜欢Pretty，因为Pretty提供了自动更新golden文件的功能。这时候你开启Pretty的debug模式，运行，屏幕上除了输出对象a的pretty结构和Diff信息之外，Pretty还会问你“Overwrite (Y/N)? ”，回答Y即可自动更新golden文件test1。
有了Pretty，你永远不需要手动写golden文件：当golden不存在时，Pretty会帮你创建；当golden存在但有Diff时会提醒你是否需要更新。
##1.4 Pretty原理及源码

也许你已经迫不及待地想知道Pretty类是怎么实现的，原理其实也简单，就是通过Java的“反射”机制，把类的成员变量拿出来，放进一个Map里，key为成员变量名，value为成员变量的值，然后递归地输出到一个具有缩进层次的代表pretty结构的字符串里。这是一个既美丽又好用的字符串，在debug模式下打印到标准输出，在unit test下就是与golden文件进行字符串比较，从而避免了做对象比较的麻烦，同时golden文件的pretty结构记录了期待对象完整的层层信息，有助于理解代码，^_^。

{% highlight cpp %}
package org.wenzhe.jvlib.debug;

import java.io.File;
import java.io.IOException;
import java.lang.reflect.Field;
import java.math.BigDecimal;
import java.math.BigInteger;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

import org.wenzhe.jvlib.file.FileUtil;

/**
 * @author liuwenzhe2008@qq.com
 *
 */
public class Pretty {

  private static final String TAB = "  ";

  private static boolean debugMode = false;
  private static boolean showFileAbsPath = false;

  public static void setDebugMode(boolean toDebug) {
    debugMode = toDebug;
    Golden.setDebugMode(debugMode);
  }

  public static void setShowFileAbsPath(boolean toShowFileAbsPath) {
    showFileAbsPath = toShowFileAbsPath;
  }

  private static Map<String, Object> obj2map(Object o) {
    Map<String, Object> props = new TreeMap<String, Object>();
    Class<?> c = o.getClass();
    for (Field field : c.getDeclaredFields()) {
      String name = field.getName();
      Object value = null;
      boolean originalAccessible = field.isAccessible();
      if (!originalAccessible) {
        field.setAccessible(true);
      }
      try {
        value = field.get(o);
      } catch (IllegalArgumentException e) {
        throw new RuntimeException("Should not reach!", e);
      } catch (IllegalAccessException e) {
        throw new RuntimeException("Should not reach!", e);
      } finally {
        if (!originalAccessible) {
          field.setAccessible(false);
        }
      }
      props.put(name, value);
    }
    return props;
  }

  public static void println(Object obj, int level) {
    System.out.println(str(obj, level));
  }

  public static String str(Object obj, int level) {
    return str(obj, level, debugMode);
  }

  public static String str(Object obj, int level, boolean debugMode) {
    String result = str(obj, 0, level);
    if (debugMode) {
      System.out.println(result);
    }
    return result;
  }

  @SuppressWarnings("unchecked")
  private static String str(Object obj, int tabCnt, int level) {
    if (obj == null) {
      return "";
    }
    else if (tabCnt > level ||
        obj instanceof String || 
        obj instanceof BigDecimal || 
        obj instanceof BigInteger ||
        obj instanceof Integer || 
        obj instanceof Short ||
        obj instanceof Long ||
        obj instanceof Float ||
        obj instanceof Double ||
        obj instanceof Boolean ||
        obj instanceof Class ||
        obj instanceof Date
        ) {
      return obj.toString();
    }
    else if (obj instanceof File) {
      File file = (File)obj;
      if (showFileAbsPath) {
        return FileUtil.unixPath(file.getAbsoluteFile());
      }
      else {
        return file.getName();
      }
    }

    else if (obj instanceof Iterable) {
      List<String> results = new ArrayList<String>();
      for (Object o : (Iterable<?>)obj) {
        results.add(str(o, tabCnt + 1, level));
      }
      return results.toString();
    }
    else if (obj instanceof Object[]) {
      List<String> results = new ArrayList<String>();
      for (Object o : (Object[])obj) {
        results.add(str(o, tabCnt + 1, level));
      }
      return results.toString();
    }
    else if (obj instanceof Map) {
      Map<String, String> results = new TreeMap<String, String>();
      for (Map.Entry<Object, Object> entry : ((Map<Object, Object>)obj).entrySet()) {
        String key = str(entry.getKey(), tabCnt + 1, level);
        String value = str(entry.getValue(), tabCnt + 1, level);
        results.put(key, value);
      }
      return results.toString();
    }
    else {
      Map<String, Object> m = obj2map(obj);
      StringBuilder sb = new StringBuilder();
      sb.append(obj.getClass().getName() + " {\n");

      String nTabs = tabs(tabCnt + 1);
      for (Map.Entry<String, Object> entry : m.entrySet()) {
        sb.append(nTabs);
        sb.append(entry.getKey());
        sb.append(" : ");
        sb.append(str(entry.getValue(), tabCnt + 1, level));
        sb.append("\n");
      }
      sb.append(tabs(tabCnt));

      sb.append("}");
      return sb.toString();
    }
  }

  private static String tabs(int count) {
    StringBuilder sb = new StringBuilder();
    while (count-- > 0) {
      sb.append(TAB);
    }
    return sb.toString();
  }

  public static boolean equalsGolden(String goldenFileName, Object obj) throws IOException {
    return equalsGolden(goldenFileName, obj, 5);
  }

  public static boolean equalsGolden(String goldenFileName, Object obj, int level) throws IOException {
    File goldenFile = new File("src/test/resources/golden/pretty", goldenFileName);
    return equalsGolden(goldenFile, obj, level);
  }

  public static boolean equalsGolden(File goldenFile, Object obj, int level) throws IOException {
    String actual = str(obj, level, false).trim();
    return Golden.equals(goldenFile, actual);
  }
}
{% endhighlight %}

在调试过程中，Pretty的str方法和println方法是很常用的；而在unit test中，equalsGolden方法更加方便。
##1.5 Pretty姐妹篇：Golden原理及源码
Pretty类用到了另一个相当实用的工具类：Golden，是Pretty的好姐妹，如果golden文件不存在则帮你创建，如果存在了则帮你把字符串跟golden文件做比较，一旦发现差异，则将差异部分打印出来。在Golden类的调试模式下（debugMode=true）还会提示你是否需要overwirte你的golden文件。这是很实用的功能，试想一下如果有上千个golden文件，维护的工作量是很大的。需求变了，代码结构也变了，原先的golden不再正确时就需要更新。要是每次都得手动去文件里查找哪些不同，手动去修改golden文件，那也是相当麻烦的事。Golden类可以给你“一键搞定”的成就感！

{% highlight cpp %}
package org.wenzhe.jvlib.debug;

import java.io.File;
import java.io.IOException;

import org.wenzhe.jvlib.diff.DiffUtil;

import com.google.common.base.Charsets;
import com.google.common.io.Files;

/**
 * @author liuwenzhe2008@qq.com
 *
 */
public class Golden {

  private static boolean debugMode = false;

  public static void setDebugMode(boolean toDebug) {
    debugMode = toDebug;
  }

  public static boolean equals(String goldenFileName, String actual) throws IOException {
    File goldenFile = new File("src/test/resources/golden", goldenFileName);
    return equals(goldenFile, actual);
  }

  public static boolean equals(File goldenFile, String actual) throws IOException {
    if (debugMode) {
      System.out.println(actual);
    }
    goldenFile = goldenFile.getAbsoluteFile();
    if (!goldenFile.isFile()) {
      System.out.println("Generate golden file: " + goldenFile);
      Files.createParentDirs(goldenFile);
      Files.write(actual, goldenFile, Charsets.UTF_8);
      return true;
    }
    String expected = Files.toString(goldenFile, Charsets.UTF_8);
    if (actual.equals(expected)) {
      return true;
    } else {
      // need 3'rd party: diffutils {
      System.out.println("Diff from Expected to Actual: ");
      System.out.println(DiffUtil.diff(expected, actual));
      if (debugMode) {
        System.out.print("Overwrite (Y/N)? ");
        char in = (char)System.in.read();
        if (in == 'Y' || in == 'y') {
          Files.write(actual, goldenFile, Charsets.UTF_8);
          return true;
        }
      }
      // }
     ?return false;
    }
  }
}
{% endhighlight %}
Pretty之Python版

Python的实现方法非常简单，自带的pprint方法就可以实现pretty print，因此要做到主要是将object转换成dict（即Java里的Map），而Python自带的vars函数返回的就是成员变量的dict。源码如下：
{% highlight cpp %}
# author: liuwenzhe2008@qq.com
import pprint
from StringIO import StringIO

def obj2map(o):
    """ if o doesn't have __dict__, not return map """
    if hasattr(o, "__dict__"):
        m = vars(o)
        return obj2map(m)
    elif type(o) == dict:
        m = {}
        for k,v in o.items():
            key = obj2map(k)
            if not key.__hash__:
                key = str(key)
            m[key] = obj2map(v)
        return m
    elif type(o) == list:
        arr = []
        for item in o:
            arr.append(obj2map(item))
        return arr
    elif type(o) == tuple:
        return tuple(obj2map(list(o)))
    else:
        return o

def printObj(o, stream=None):
    """Pretty-print a mapped Python object to a stream [default is sys.stdout]."""
    pprint.pprint(obj2map(o), stream)

def obj2str(o):
    s = StringIO()
    printObj(o, s)
    return s.getvalue()
{% endhighlight %}

由于Python的动态脚本语言特性，我们可以在运行时导入Pretty，然后打印感兴趣的对象。下面是Pretty在pdb调试中的例子。
{% highlight cpp %}
pdb> import Pretty
pdb> Pretty.printObj(xxx)
{% endhighlight %}

Python版的Pretty，输出结果也是同样pretty，请看下面的unit test文件，特别是复杂类A的对象a所对应的pretty结构，即字符串expectedStrA
{% highlight cpp %}
# author: liuwenzhe2008@qq.com
import unittest
import Pretty

class A:
    def __init__(self):
        self.a1 = "a"
        self.a2 = 2
        self.b = B()
        self.c = [C(1), C(2)]

class B:
    def __init__(self):
        self.bm = {3: C(3), "4": C(4), C(7):C(8)}
        self.bt = (C(5), C(6))

class C:
    def __init__(self, c):
        self.c = c
        self.cs = str(c)

expectedMapA = \
{'a1': 'a',
 'a2': 2,
 'b': {'bm': {3: {'c': 3, 'cs': '3'},
              '4': {'c': 4, 'cs': '4'},
              "{'cs': '7', 'c': 7}": {'c': 8, 'cs': '8'}},
       'bt': ({'c': 5, 'cs': '5'}, {'c': 6, 'cs': '6'})},
 'c': [{'c': 1, 'cs': '1'}, {'c': 2, 'cs': '2'}]}

expectedStrA = """
{'a1': 'a',
 'a2': 2,
 'b': {'bm': {3: {'c': 3, 'cs': '3'},
              '4': {'c': 4, 'cs': '4'},
              "{'cs': '7', 'c': 7}": {'c': 8, 'cs': '8'}},
       'bt': ({'c': 5, 'cs': '5'}, {'c': 6, 'cs': '6'})},
 'c': [{'c': 1, 'cs': '1'}, {'c': 2, 'cs': '2'}]}
"""

class Test(unittest.TestCase):
    def setUp(self):
        self.a = A()

    def testObj2map(self):
        m = Pretty.obj2map(self.a)
        self.assertEqual(expectedMapA, m)
        self.assertEqual(type(B()), type(self.a.b))

    def testObj2Str(self):
        s = Pretty.obj2str(self.a)
        self.assertEqual(expectedStrA.strip(), s.strip())

        Pretty.printObj(self.a)

if __name__ == "__main__":
    unittest.main()
{% endhighlight %}

##3.Pretty之Groovy版
Groovy是语法简化、但却功能扩展的Java，思路是一样的，只是代码写起来简单一些（比如反射、格式化等）。源码如下：

{% highlight cpp %}

package org.wenzhe.gvlib

/**
 * Pretty print an object detailed to string, console
 * 
 * @author liuwenzhe2008@qq.com
 *
 */
class Pretty {
  private static final String TAB = " " * 2

  static String str(obj) {
    return str(obj, false)
  }
  static String str(obj, boolean recursive) {
    return strLevel(obj, (recursive ? 1 : 0))
  }
  private static String strLevel(obj, int tabLevel) {
    if (obj == null ||
        obj instanceof String || 
        obj instanceof BigDecimal || 
        obj instanceof BigInteger ||
        obj instanceof Integer || 
        obj instanceof Short ||
        obj instanceof Long ||
        obj instanceof Float ||
        obj instanceof Double ||
        obj instanceof Boolean ||
        obj instanceof Class
        ) {
      return obj.toString()
    }
    if (  obj instanceof List || 
        obj instanceof Object[] ||
        obj instanceof Set
        ) {
      List<String> prettyList = obj.collect {
        if (tabLevel <= 0) {
          return str(it)
        } else {
          return strLevel(it, tabLevel + 1)
        }

      }
      return prettyList.toString()
    }
    if (obj instanceof Map) {
      if (!obj) {
        return obj.toString()
      }
      List<String> list = obj.collect() { key, val ->
        if (tabLevel <= 0) {
          return str(key) + ' : ' + str(val)
        } else {
          return str(key) + ' : ' + strLevel(val, tabLevel + 1)
        }
      }
      return str(list)
    }

    Map<String, Object> props = obj.getProperties()

    if (tabLevel <= 0) {
      return props.inject(obj.class.name + ' {\n') { buf, entry ->
        if (entry.key == "class") {
          return buf;
        } else {
          return buf + TAB + "$entry.key : $entry.value\n"
        }
      } + "}"
    } else {
      return props.inject(obj.class.name + ' {\n') { buf, entry ->
        if (entry.key == "class") {
          return buf;
        } else {
          return buf + strFormat(entry.key, entry.value, tabLevel)
        }
      } + TAB * (tabLevel - 1) + "}"
    }
  }

  private static String strFormat(String key, Object value, int tabLevel) {
    String s = strLevel(value, tabLevel + 1)
    return TAB * tabLevel + "$key : $s\n";
  }

  static String listMethodObjs(obj) {
    return Pretty.str(obj.metaClass.methods)
  }

  static String listMethodDescs(obj) {
    List<String> methods = obj.metaClass.methods.cachedMethod*.toString()
    return formatMethodList(obj, methods)
  }

  static String listMethodNames(obj) {
    List<String> methods = obj.metaClass.methods.cachedMethod*.name.sort().unique()
    return formatMethodList(obj, methods)
  }

  static private String formatMethodList(obj, List<String> methods) {
    return """${obj.class.name} {
    ${methods.join("\n    ")}
}"""
  }
}

{% endhighlight %}
##4. Pretty之C++设计思路
由于C++没有“反射”机制，要想获取类的所有私有（或公有）成员变量的名字与类型并不容易。但思路还是有的，比如可以通过分析C++类的源代码来获得，可以借助第三方库，如Clang来实现。Clang由Apple开发，BSD开源授权，支持C，C++，Object C，Object C++等编程语言，能够对源代码进行词法和语意分析，结果为抽象语法树。通过抽象语法树，我们可以模仿类似与Java中“反射”机制，来得到类的成员信息（名字，类型，取值等）。只是一个思路，有兴趣的朋友不妨一试。