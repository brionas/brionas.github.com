layout: post
title : "Pretty�����ࣺ��������������뵥Ԫ���Ը�happy!"
description :pretty ����
category : "Java"
tags : [Java]
refer_author: Wenzhe
refer_blog_addr:http://blog.csdn.net/liuwenzhe2008
refer_post_addr:http://blog.csdn.net/liuwenzhe2008/article/details/9104331

{% include JB/setup %}

    ������������Թ����У�������ȥ�鿴ĳһ�����ȡֵ������֮�临�ӵĲ�ι�ϵ���ټ������飨������ӳ�䣨�ֵ䣩�ȶ������ݽṹ������������һĿ��Ȼ�����Ľ��ܵ�Pretty�����࣬�������ķ�ʽͻ����֮��Ĳ�ι�ϵ�����ҽ�����һ���������ṹpretty�ش�ӡ������

    �ڱ�д��Ԫ����ʱ��������ȥ�Ƚ�ĳһ�����Ƿ����Ԥ�ȵ�����ֵ��������һ��������Ķ������ֵ�Ԫ���Բ�����д������Ƭ�滯�����ӻ���Pretty�����࣬���ܹ������ļ�⸴����Ķ��󣬶��ҿɶ��Ժã����������롣�������������ṹ��ƥ��ʱ�������������Diff��Ϣ�����������û��Ƿ���Ҫ�Զ�����case�������á�

    ����ʵ����3�����԰汾��Pretty�����ࣺJava�棬Python�棬Groovy�档�����Java���Pretty�����ص���ܣ��������汾ֻ�Ǽ򵥴�������Ϊʵ�ֵ�ԭ����һ���ģ�ֻ�ǻ��ɲ�ͬ���Զ��ѡ���󣬶���ͬ���㷺Ӧ�õ�C++��������Ȼû�и�������ʵ�֣���Ҳ�ṩ��һ�����˼·������Ȥ�����ѿ�����һ�ԡ�

##1. Pretty֮Java��

 ##1.1 �����е�����
    �������ڵ��Գ����ʱ�򣬾�����鿴ĳһ������ֵ��һ����˵�������ַ����������õ���

	1.     �õ���������Eclipse  Debug������gdb/pdb��

	2.     ��print��������logger��ֱ�ӽ�����ֵ��ӡ������

    �����ְ취����ȱ�㣬��������Ҫһ���չ�������������������������ṹ���߹�ϣ���ʱ�򣬾Ͳ�̫���׿������ˡ���print������ʵֻ��toString�����ķ���ֵ��ȡ����toString������ʵ�֣���ʵ�����ɿ���

    �����д����Ķ��߻��뵽���������ڶ������ʱ�򣬶�overrideһ��toString�����������ɶ���������Object���ȱʡʵ�֣�JVM�еĵ�ַ����

    ����һ����������뷨��

	1.     ���ȣ��������е��඼�ܹ������ǿ��ƣ���Java��⣬�������⣬���������ŶӵĴ��룬�ȵȡ�

	2.     ���toString��������������ҵ���ֵ������ֻ��Ϊ�˷�����ԡ�

	3.     ���⹤�������ⲻ�Ǳ���ģ���������Ҫ��ÿ���඼ȥoverride toString�����������Ӻܶ�û�б�Ҫ�Ĺ������������ܶ಻��Ҫ�Ĵ��룬����������ά������Ĺ���������������bug�����ң�
    
    ����ÿ������/�޸�/ɾ����Ա����ʱ����Ҫȥ�޸�toString����������print��������ϢҲ�Ͳ��ɿ��ˣ������Ǻ��ѱ�֤�ġ�

##1.2 ��Ԫ�����е�����
 
    ������һ����A���ۺ�����B����B�־ۺ���C�����´��룺
 
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
	private Map<String, Date> map;
	private C c = new C();
	// ...
   }
  
   class C {
    private double v1;
    private BigDecimal v2;
    // ...
   }
   
   {% endhighlight %}

   ���������Ȳ���һ���� A �Ķ��� a �ǲ������ڴ��ģ�һ�������뵽���漸��������

   1. �����г�Ա������ get �����Ƚϣ�
   
   {% highlight cpp %}
   assertEquals(xxx, a.getId());
   assertEquals(xxx, a.getPath());
   ...
   assertEquals(xxx, a.getB().getDesc());
   ...
   assertEquals(xxx, a.getB().getC().getV1());
   ...
   {% endhighlight %}
   ���ְ취�������Զ��׼��������С��©��һ����Ҫ��Ա������get���ǲ��ԾͲ���ȫ���ˡ����Ҳ��������г�Ա��������get�������費��Ҫget�����ÿ�������Ҫ��������ֻΪunit testר���ṩ�����ң�����ô�򵥵��࣬����Ҫ��ô����assertEqual������и���ĳ�Ա���������к���ľۺϲ�Σ��ǽ��޷��������A��Ľṻ�������������ǸĶ��ĵط��ͺܶ��ˡ�������unit testά���ɱ�֮�߿����֪������˭�ж���дunit test����Ϊ�����ڸ��Լ����鷳�����ң��������еĴ������Ƕ��ܿ��Ƶģ�����������⡣
   
   2. ΪA��ʵ��equals��������assertEquals��ֻ��һ���ˣ�

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
   
   ��ȻassertEqualsֻ��һ������Ϊ�˽���һ���ڴ���expected��Ϊ������Ƚϣ���ҪΪ�ṩ������set��������ô��set����������������ʵ��������ô��get�١�
   ���ң�ΪA��ʵ��equals����Ҳ���з��յģ���Ϊequals��������Ҳ��Ҫ���ԣ�ֻҪ����д�Ĵ��뱾���϶���Ҫ���ԣ�����Ҳ��Ҫʱ��ɱ����ܶ�����ʵû��Ҫȥoverride equals������д����͵�ά����û��Ҫд�Ĵ�������д������ά�������ࡣͬ���ģ��������еĴ��붼�ܿ��Ƶġ�

##1.3 ʹ��Pretty

   Pretty����Ժ�pretty�Ľ�����ϵ��Ժ͵�Ԫ�����е����⡣�ڸ���Pretty��֮ǰ���ȴ�ʹ���ߵĽǶȿ�������pretty��
   
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
	
  ##1.3.1 Pretty�ṹ

    ����һ������main�����ĵ�Ԫ�����ࡣ��Ʋ����Ԫ���ԣ����ǰ�������һ����ͨjava�ļ������У�����main������ʼ���У�������Ļ�ϻ��ӡ������ a ��pretty�ṹ��
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
	
	����pretty�ṹ�����������Ժ����׿���������a�����ǣ� org.wenzhe.jvlib.debug.test.PrettyTest����ڲ���A�����Աarray��һ�����飬ֵΪ[86, 755, 1234, 5678]����һ����Ա b ���� ��PrettyTest���ڲ���B���Ķ���b ����ĳ�Ա����c ���ࣨPrettyTest�ڲ���C���Ķ��󡭡��������������ֳ�Ա��������Ƕ�׾ۺ���Ķ���Ҳ�����׿ɼ������ڿ������Թ����зǳ����ã�
    ����Ե�Ԫ���Եķ�ʽ���У���Ļ��û���κ������No news is Good news����JUnit View�г��ִ��ϲ������ɫ����ף��������ͨ���ˡ���һ�����unittest��˵����ȷ��ʱ����û�����Ϣ�ġ���
    ��ô������ô֪������a���������أ�ע�⵽��61�У�Pretty.equalsGolden("test1", a);  ����aʵ�����Ǹ�һ������Ϊtest1��golden�ļ����˱Ƚϡ����golden�ļ������ڵ�Ŀ¼Ϊ�� ${project_root}/src/test/resources/golden/pretty/������pretty���ߵ�һ��convention����ȻҲ���Ըĳɱ��Ŀ¼�����Ҳ��Ƽ��ģ��ܶ�ʱ����ӡ�Լ���������á���ԭ�����Ǹ��õġ���test1�ļ�����ᷢ����Ҳ��һ��pretty�ṹ����֮ǰ��Ļ���������ȫһ����

    ��������Ϊʲô��Ϊ��ͨjava��������Ļ�ϻ��ӡ������Ϊunit testȴ�����ӡ�أ���ʵ���𲢲��������������з�ʽ��Ψһ�����������Ƿ�ѡ����Pretty���debugģʽ����һ����˵��unit test�²�����debugģʽ�����ڿ������Թ�������������ע�⵽main�����տ�ʼ��ʱ�򣨵�65�У���debug mode����Ϊtrue����Pretty����Ҫ�õ�����a��pretty�ṹʱ���Ὣ����ӡ������������ԣ�ʡ���ڴ����������print�������鷳��debug modeȱʡ�ǹصģ�����unit test��û�д�ӡ�����ˡ�������Ȥ�����Ķ������Դ���룩��

  ##1.3.2 Pretty Diff
  
    ���unit test�⵽����a��golden�ļ���ͬ���ǻ������������и����ϴֲ�С�İ�C���еĳ�Ա����v1ɾ���ˣ��ֲ�С�������˳�Ա����v3��ȡֵΪtrue�������ǲ�С�İ�A��ĳ�Ա����list����insert��һ����NOT���������ǲ�����Pretty��debugģʽ����Ļ�϶��������

	{% highlight cpp %}
	Diff from Expected to Actual: 
    -:       v1 : 3.14
    +:       v3 : true
    <:   list : [My, name, is, Wenzhe]
    >:   list : [My, name, is, NOT, Wenzhe]
	{% endhighlight %}

	Pretty���ߵĴ����������pretty�ɣ����ϴָ�����Щ��������һĿ��Ȼ��

  ##1.3.3 Pretty Golden
    ������ϴ��ǹ�����ô�޸ĵģ������д��ϰ�֧�֣��������ҵ�����Խ����ǡ�������ˡ�������ôgolden�ļ�Ҳ�͹�ʱ�ˣ���Ҫ���²�����unit testͨ����

    ��Ҫ�ֶ�����golden�ļ����ҿɲ��ɣ���ΪPretty����Խ��Խ���ˡ�

    ���˶�ϲ��Pretty����ΪPretty�ṩ���Զ�����golden�ļ��Ĺ��ܡ���ʱ���㿪��Pretty��debugģʽ�����У���Ļ�ϳ����������a��pretty�ṹ��Diff��Ϣ֮�⣬Pretty�������㡰Overwrite (Y/N)? �����ش�Y�����Զ�����golden�ļ�test1��
    ����Pretty������Զ����Ҫ�ֶ�дgolden�ļ�����golden������ʱ��Pretty����㴴������golden���ڵ���Diffʱ���������Ƿ���Ҫ���¡�

##1.4 Prettyԭ��Դ��
 
    Ҳ�����Ѿ��Ȳ���������֪��Pretty������ôʵ�ֵģ�ԭ����ʵҲ�򵥣�����ͨ��Java�ġ����䡱���ƣ�����ĳ�Ա�����ó������Ž�һ��Map�keyΪ��Ա��������valueΪ��Ա������ֵ��Ȼ��ݹ�������һ������������εĴ���pretty�ṹ���ַ��������һ���������ֺ��õ��ַ�������debugģʽ�´�ӡ����׼�������unit test�¾�����golden�ļ������ַ����Ƚϣ��Ӷ�������������Ƚϵ��鷳��ͬʱgolden�ļ���pretty�ṹ��¼���ڴ����������Ĳ����Ϣ�������������룬^_^��

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

	�ڵ��Թ����У�Pretty��str������println�����Ǻܳ��õģ�����unit test�У�equalsGolden�������ӷ��㡣

##1.5 Pretty����ƪ��Goldenԭ��Դ��
    Pretty���õ�����һ���൱ʵ�õĹ����ࣺGolden����Pretty�ĺý��ã����golden�ļ�����������㴴��������������������ַ�����golden�ļ����Ƚϣ�һ�����ֲ��죬�򽫲��첿�ִ�ӡ��������Golden��ĵ���ģʽ�£�debugMode=true��������ʾ���Ƿ���Ҫoverwirte���golden�ļ������Ǻ�ʵ�õĹ��ܣ�����һ���������ǧ��golden�ļ���ά���Ĺ������Ǻܴ�ġ�������ˣ�����ṹҲ���ˣ�ԭ�ȵ�golden������ȷʱ����Ҫ���¡�Ҫ��ÿ�ζ����ֶ�ȥ�ļ��������Щ��ͬ���ֶ�ȥ�޸�golden�ļ�����Ҳ���൱�鷳���¡�Golden����Ը��㡰һ���㶨���ĳɾ͸У�

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
	
2. Pretty֮Python��

   Python��ʵ�ַ����ǳ��򵥣��Դ���pprint�����Ϳ���ʵ��pretty print�����Ҫ������Ҫ�ǽ�objectת����dict����Java���Map������Python�Դ���vars�������صľ��ǳ�Ա������dict��Դ�����£�
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

      ����Python�Ķ�̬�ű��������ԣ����ǿ���������ʱ����Pretty��Ȼ���ӡ����Ȥ�Ķ���������Pretty��pdb�����е����ӡ�
	  {% highlight cpp %}
	  pdb> import Pretty
      pdb> Pretty.printObj(xxx)
	  {% endhighlight %}
	    
	  Python���Pretty��������Ҳ��ͬ��pretty���뿴�����unit test�ļ����ر��Ǹ�����A�Ķ���a����Ӧ��pretty�ṹ�����ַ���expectedStrA
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
	 
##3.Pretty֮Groovy��
    Groovy���﷨�򻯡���ȴ������չ��Java��˼·��һ���ģ�ֻ�Ǵ���д������һЩ�����練�䡢��ʽ���ȣ���Դ�����£�
   
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
	
##4. Pretty֮C++���˼·
  ����C++û�С����䡱���ƣ�Ҫ���ȡ�������˽�У����У���Ա���������������Ͳ������ס���˼·�����еģ��������ͨ������C++���Դ��������ã����Խ����������⣬��Clang��ʵ�֡�Clang��Apple������BSD��Դ��Ȩ��֧��C��C++��Object C��Object C++�ȱ�����ԣ��ܹ���Դ������дʷ���������������Ϊ�����﷨����ͨ�������﷨�������ǿ���ģ��������Java�С����䡱���ƣ����õ���ĳ�Ա��Ϣ�����֣����ͣ�ȡֵ�ȣ���ֻ��һ��˼·������Ȥ�����Ѳ���һ�ԡ�
