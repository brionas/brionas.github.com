---
layout: post
title: Prepare complex input and check complex output for unit test
description: Prepare complex input and check complex output for unit test
category: Java
tags: Unit-test
refer_author: Wen-Zhe
refer_blog_addr:
refer_post_addr:
---
Prepare complex input for unit test
===

Problem
---

In TDD, we write unit test before code. This is good. However, we don't follow TDD each time, maybe based on some reasons: tight schedule, no clear expected output, GUI related, etc.

After the tight schedule when we have time to look back, we find the code coverage of unit test is in a low level. We need to add more and more unit test cases and more conditions to improve the code coverage.

But we find some test case is not easy to write, for example, we need to test the following two methods:
{% highlight java %}
public static String validate(PatternInputSet inputSet) {...}
{% endhighlight %}
The input object is very complex and have multiple levels.
{% highlight java %}
PatternInputSet
|-- private boolean supportXOverlay;
|-- ...
|-- private boolean supportYOverlay;
|-- private List<OvlPatternDefInputNew> patternDefSet
            |-- private String yBottomPatternId;
            |-- ...
            |-- private String yBottomRoleName;
            |-- private List<LayerPatternDef> layerPatternDefList;
                                |-- private LayerSetting xLayerSetting;
                                |-- private LayerSetting yLayerSetting;
                                                |-- private String perpendPitch;
                                                |-- private String perpendCd;
                                                |-- private Orientation orientation;
                                                |-- ...
{% endhighlight %}
You see, it is not easy to construct it.

And it is also difficult to mock it, because the to-be-test method will call many methods of input parameter.

And the validate rules are full of domain knowledge, we don't have enough knowledge to input.

Also, we are not allow to refactory the complex class, because the release day is very closed and stability is the most important care.

Then, how to prepare the complex input for unit test?

And how to prepare enough input data to maximize the code coverage?


Solutions:
---
I have an idea: when running GUI/Application (we have many real cases from QA,PEG), serialize the input parameter to a readable string (better with pretty format, so that we can easily to read and modify). In unit test, we can de-serialize the readable string to a Java object as the input of unit test.

For the readable string format, json is a suitable choice. We can use a json third party lib (such as Jackson, Google Gson, etc) to serialize and de-serialize between Java object and json string.

For example, we can add one of the following code to the to-be-test method "validate" temperately:
{% highlight java %}
public static String validate(PatternInputSet inputSet) {
  log.debug("json pretty string of inputSet:\n"
      + Gsons.toJson(input, new File("~/testdata/validate_input.json"))); // Optional solution 1
  log.debug("json pretty string for code of inputSet:\n"
      + Gsons.toCode(input));                                             // Optional solution 2
  ... // original code
}
{% endhighlight %}
In the above code, we log input parameter to json string. Gsons class is my wrap for gson lib, so that we can use it simply. I will show the class in the end of the article.

We have two way to generate Java object from json:

**Optional solution 1:from json file.**

Let's see Optional solution 1 in the above example. After running GUI/Application, we will see a readable pretty json format for input parameter inputSet:
{% highlight json %}
{
  "supportXOverlay": true,
  "supportYOverlay": true,
  "patternDefSet": [
    {
      "mainPitch": "500, 550",
      "isLineOnLine": false,
      "name": "New Target 1",
      "patternUniqueId2DevLayerName": {
        "M1_Pattern_18": "M1",
        "V0_Pattern_6": "V0"
      },
      "layerPatternDefList": [
        {
          "patternId": "V0_Pattern_6",
          "roleName": "role1",
          "xLayerSetting": {
            "primaryPitch": "550, 500",
            "primaryCd": "250",
            ...    // skip several lines
          },
          ...    // skip several lines
        },
        ...    // skip several lines
      ],
      ...    // skip several lines
    }
  ],
  ...    // skip several lines
}
{% endhighlight %}

Then, in our unit test code, we can load json file and de-serialize to Java object, and then test.

{% highlight java %}
@Test
public void testValidate() {
  PatternInputSet inputSet = Gsons.fromJson(PatternInputSet.class,
      new File("~/testdata/validate_inputSet.json"));
  assertEquals("", Validator.validate(inputSet));
  ...
}
{% endhighlight %}

**Optional solution 2: embed json string to code directly.**

As Java doesn't support multiple line string, we have to write unit test in this way:
{% highlight java %}
@Test
public void testValidate() {
  PatternInputSet inputSet = Gsons.fromJson(PatternInputSet.class,
    "{\n" +
    "  \"supportXOverlay\": true,\n" +
    "  \"supportYOverlay\": true,\n" +
    "  \"patternDefSet\": [\n" +
    "    {\n" +
    "      \"mainPitch\": \"500, 550\",\n" +
    "      \"isLineOnLine\": false,\n" +
    "      \"name\": \"New Target 1\",\n" +
    "      \"patternUniqueId2DevLayerName\": {\n" +
    "        \"M1_Pattern_18\": \"M1\",\n" +
    "        \"V0_Pattern_6\": \"V0\"\n" +
    "      },\n" +
    "      \"layerPatternDefList\": [\n" +
    "        {\n" +
    "          \"patternId\": \"V0_Pattern_6\",\n" +
    "          \"roleName\": \"role1\",\n" +
    "          \"xLayerSetting\": {\n" +
    "            \"primaryPitch\": \"550, 500\",\n" +
    "            \"primaryCd\": \"250\",\n" +
                 ...    // skip several lines
    "          },\n" +
               ...    // skip several lines
    "        },\n" +
             ...    // skip several lines
    "      ],\n" +
           ...    // skip several lines
    "    }\n" +
    "  ],\n" +
       ...    // skip several lines
    "}");
assertEquals("", Validator.validate(inputSet));
{% endhighlight %}
on string in code, is not written manually, but copied from the generated code by Optional solution 2 in the above example when running GUI/Application. In this way, unit test doesn't depend on json file outside.

The above two solutions, which one is better and suitable, that depends on you. I prefer the second one, because we don't have to jump to other json file outside.

Check complex output for unit test
===

Problem
---

Assume we are going to write unit test for the following method:
{% highlight java %}
public PatternInputSet generateBy(TargetType targetType) {...}
{% endhighlight %}
In order to check result carefully, we may write unit test in this way:
{% highlight java %}
@Test
public void testGenerateBy() {
  // create test data "builder", skip code here ...
  PatternInputSet result = builder.generateBy(TargetType.C10);
  // check result is expected or not
  assertTrue(result.getSupportXOverlay());
  assertTrue(result.getSupportYOverlay());
  assertEquals(2, result.getPatternDefSet().size());
  assertEquals("500, 550", result.getPatternDefSet().get(0).getMainPitch());
  assertEquals("M1", result.getPatternDefSet().get(0).getPatternUniqueId2DevLayerName().get("M1_Pattern_18"));
  assertEquals("V0", result.getPatternDefSet().get(0).getPatternUniqueId2DevLayerName().get("V0_Pattern_6"));
  ...
  // skip more than 100 assert ...
  ...
}
{% endhighlight %}
n order to check result carefully, we have to check each field and sub field in it. So hundreds of thousands of assert will be added to unit test. Further more, we also need the class has getter for each field and sub fields.

It is so boring, isn't it?

Then, how can we check complex output for unit test?

Solutions:
---

In stead of checking each field, I think we can check the whole object in the format of readable string with pretty format. When there is something different between expected and actual value, we can see the difference very easily.

There are two kinds of pretty string: one is json format, the other is my pretty format.

**Optional solution 1: serialize result to json format, then compare with golden json file.**

{% highlight java %}
@Test
public void testGenerateBy() {
  // create test data "builder", skip code here ...
  PatternInputSet result = builder.generateBy(TargetType.C10);
  // Gsons.printJson(result);  // print json format, it is useful to create golden file at the first time
  assertEquals(FileUtils.readFileToString(new File("golden.json"), Charsets.UTF_8), Gsons.toJson(result));
}
{% endhighlight %}
The following is the golden file "golden.json", which is copied (or modify) from the output of Gsons.printJson(result) at the first time.
{% highlight json %}
{
  "supportXOverlay": true,
  "supportYOverlay": true,
  "patternDefSet": [
    {
      "mainPitch": "500, 550",
      "isLineOnLine": false,
      "name": "New Target 1",
      "patternUniqueId2DevLayerName": {
        "M1_Pattern_18": "M1",
        "V0_Pattern_6": "V0"
      },
      "layerPatternDefList": [
        {
          "patternId": "V0_Pattern_6",
          "roleName": "role1",
          "xLayerSetting": {
            "primaryPitch": "550, 500",
            "primaryCd": "250",
            ...    // skip several lines
          },
          ...    // skip several lines
        },
        ...    // skip several lines
      ],
      ...    // skip several lines
    }
  ],
  ...    // skip several lines
}
{% endhighlight %}

**Optional solution 2: serialize result to json format, then compare with json string embedded in code.**
{% highlight java %}
@Test
public void testGenerateBy() {
  // create test data "builder", skip code here ...
  PatternInputSet result = builder.generateBy(TargetType.C10);
  // Gsons.printJsonToCode(result);  // print json to code embedded, it is useful to copy into Java code
  assertEquals(
    "{\n" +
    "  \"supportXOverlay\": true,\n" +
    "  \"supportYOverlay\": true,\n" +
    "  \"patternDefSet\": [\n" +
    "    {\n" +
    "      \"mainPitch\": \"500, 550\",\n" +
    "      \"isLineOnLine\": false,\n" +
    "      \"name\": \"New Target 1\",\n" +
    "      \"patternUniqueId2DevLayerName\": {\n" +
    "        \"M1_Pattern_18\": \"M1\",\n" +
    "        \"V0_Pattern_6\": \"V0\"\n" +
    "      },\n" +
    "      \"layerPatternDefList\": [\n" +
    "        {\n" +
    "          \"patternId\": \"V0_Pattern_6\",\n" +
    "          \"roleName\": \"role1\",\n" +
    "          \"xLayerSetting\": {\n" +
    "            \"primaryPitch\": \"550, 500\",\n" +
    "            \"primaryCd\": \"250\",\n" +
                 ...    // skip several lines
    "          },\n" +
               ...    // skip several lines
    "        },\n" +
             ...    // skip several lines
    "      ],\n" +
           ...    // skip several lines
    "    }\n" +
    "  ],\n" +
       ...    // skip several lines
    "}",
    Gsons.toJson(result));
}
{% endhighlight %}

The embedded json string in code, is not written manually, but copied (or modified) from the output of Gsons.printJsonToCode(result) at the first time.

When the expected is different from the actual result, Eclipse IDE will show the difference quite clearly.

<img width="707px" src="/assets/image/2014-08/unitest.png" type="image/png" />


**Optional solution 3: use Pretty Utility Class.**

The idea is similar, they are both comparing pretty string. The difference is Pretty Utility also contains class information but json don't. So the pretty utility is more readable than json, while json is more general.

For more detail of Pretty tool, you can refer to: [Pretty Utility Class](http://blog.csdn.net/liuwenzhe2008/article/details/9104331).

Gsons class: utility to use json with wrapping for gson lib
===
In the end, I will show my Gsons class, which is used in the above code.
{% highlight java %}
/**
 * Utility for Gson
 *
 * @author Wen-Zhe.Liu@asml.com
 *
 */
public class Gsons {
 
  /**
   * create T object from json string
   *
   * @param clazz
   * @param json
   * @return
   */
  public static <T> T fromJson(Class<T> clazz, String json) {
    return getPrettyGson().fromJson(json, clazz);
  }
   
  /**
   * create T object from json file
   *
   * @param clazz
   * @param json
   * @return
   * @throws IOException
   */
  public static <T> T fromJson(Class<T> clazz, File jsonFile) throws IOException {
    return fromJson(clazz, FileUtils.readFileToString(jsonFile, Charsets.UTF_8));
  }
   
  /**
   * convert object to json string
   *
   * @param obj
   * @return
   */
  public static String toJson(Object obj) {
    return getPrettyGson().toJson(obj);
  }
   
  /**
   * convert object to json string and save to jsonFile
   *
   * @param obj
   * @param jsonFile
   * @return
   * @throws IOException
   */
  public static String toJson(Object obj, File jsonFile) throws IOException {
    String json = toJson(obj);
    FileUtils.writeStringToFile(jsonFile, json, Charsets.UTF_8);
    return json;
  }
   
  /**
   * for using json in code
   *
   * @param json
   * @return
   */
  public static String jsonToCode(String json) {
    if (Strings.isNullOrEmpty(json)) {
      return "";
    }
    return "\"" + json.replace("\"", "\\\"").replace("\n", "\\n\" +\n\"") + "\"";
  }
   
  /**
   * for using json in code
   *
   * @param object
   * @return
   */
  public static String toCode(Object obj) {
    return jsonToCode(toJson(obj));
  }
   
  /**
   * print Json To Code
   *
   * @param obj
   */
  public static void printJsonToCode(Object obj) {
    System.out.println(toCode(obj));
  }
   
  /**
   * print Json
   *
   * @param obj
   */
  public static void printJson(Object obj) {
    System.out.println(toJson(obj));
  }
   
  private static Gson getPrettyGson() {
    return new GsonBuilder()
    .enableComplexMapKeySerialization()
    .setPrettyPrinting()
    .create();
  }
}
{% endhighlight %}



