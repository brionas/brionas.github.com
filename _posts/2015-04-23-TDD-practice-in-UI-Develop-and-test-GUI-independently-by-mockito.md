---
layout: post
title: TDD practice in UI&#58; Develop and test GUI independently by mockito
description: TDD practice in UI&#58; Develop and test GUI independently by mockito
category: 测试 TDD 
tags: 测试 TDD
refer_author: Wenzhe
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

##TDD

Test-driven development (TDD), a very good practice in software development, has completely turned traditional development around. It enables you to take small steps to finish customer's requirement and improve your code. TDD can be described as the following formula:

    TDD = Refactoring + TFD (Test-first Development)

Test-first Development (TFD) requires us to write test first and write code step by step to make the test pass.
In GUI development, it is not easy to write unit test. But we can also use TDD to develop.
When we develop and test UI before, we depend on data which comes from model and domain. But the question is when model and domain code are not ready, how can we develop our UI independently?
In my practice, I use mock technology to mock model data for UI to use. Here is my practice in a new project.

##Requirement

We need to develop a GUI for showing the session information, the mock up GUI is shown as the picture below.

![mock up GUI][1]

##Draw UI

First step, draw the mock up GUI in UI tool (WindowBuilder):

![draw mock up GUI][2]

##TFD (Test-first Development) in UI

Next, we need data to fill in GUI, and we also need some GUI logic to handle user's input and selections. But now model and domain layers are not ready, and GUI should not wait for them to finish.
Actually, in Modular Design point, GUI layer should not depend on the implementation of other modules (model and domain).
In this point, for UI layer development and testing, we can mock the implementation of other modules. Interface is the only thing in other modules we need to care.
As we don't care the implementation of other modules, we have to mock their implementation. In Java, Mockito is very readable and easy to use. It is generally used in unit test. 
We develop GUI code in the concept of TFD (Test-first Development): write test first, and then write UI code. Actually they are written at the same time. Test code drives us to write UI code; on the other hand, when UI code uses some dependency, mock the dependency in test code.
We need to notice the "test" here is not unit test, because it is not easy to write a unit test for UI code (unit test is only test for one class, not integrating test, not ATF (Auto-test Framework) for GUI). The "test" here, is run a standalone GUI with only the tested UI we developed. In the "test", we can do anything similar to the final integrated products.

##Mockito

As Mockito is designed by proxy pattern, it can not only be used in unit test, but also in any where, including the standalone GUI testing code. The usual usage in such cases is:

 1. create a mock object

{% highlight java %} 
Model model = mock(Model.class);
{% endhighlight %}

2.  mock return of a method

{% highlight java %}
when(model.getPath()).thenReturn("http://brionas.github.io/liuwenzhe");
{% endhighlight %}

3. mock the implementation of a method

{% highlight java %}

doAnswer(new Answer<Void>() {
  @Override
  public Void answer(InvocationOnMock invocation) throws Throwable {
    Object[] args = invocation.getArguments();
    if (args != null && args[0] instanceof Item) {
      Item item = (Item)args[0];
      for (Item it : model.getItems()) {
        when(it.isSetTo()).thenReturn(it == item);
      }
    }
    return null;
  }
}).when(model).setTo(any(Item.class));

{% endhighlight %}

In the following example, SessionInfoComposite is the To-Be-Tested GUI class, and we will mock its model data by method `mockSessionInfoModel()`. The standalone GUI test code is:

    public class SessionInfoCompositeTest {
      public static void main(String[] args) {
        SWTTestUtil.openShell("Information", new Function<Shell, Control>() {
          @Override
          public Control apply(Shell shell) {
            SessionInfoComposite comp = new SessionInfoComposite(shell, SWT.NONE);
            comp.setSessionInfoModel(mockSessionInfoModel());
            return comp;
          }
        });
      }
      private static SessionInfoModel mockSessionInfoModel() {
        TypeModel mk31 = mock(TypeModel.class);
        when(mk31.toString()).thenReturn("mk 3.1");
         
        TypeModel mk33 = mock(TypeModel.class);
        when(mk33.toString()).thenReturn("mk 3.3");
         
        TypeModel mkx = mock(TypeModel.class);
        when(mkx.toString()).thenReturn("mk x");
         
        SimulationModel simulationModel = mockSimulationModel();
         
        SessionInfoModel model = mock(SessionInfoModel.class);
        when(model.getDesc()).thenReturn("This is a mock description.\nVery good!");
        when(model.getSupportedSensorTypes()).thenReturn(Lists.newArrayList(mk31, mk33, mkx));
        when(model.getSensorType()).thenReturn(mk33);
        when(model.getProcessPath()).thenReturn("http://brionas.github.io/liuwenzhe");
        when(model.getName()).thenReturn("Hello world");
        when(model.getSimulationModel()).thenReturn(simulationModel);
         
        return model;
      }
      private static SimulationModel mockSimulationModel() {
        List<SimulationItem> items = mockSimulationItems();
         
        final SimulationModel simulationModel = mock(SimulationModel.class);
        when(simulationModel.getItems()).thenReturn(items);
         
        mockOnSetOptimized(simulationModel);
        mockOnSetInsection(simulationModel);
         
        return simulationModel;
      }
      private static List<SimulationItem> mockSimulationItems() {
        StepModel pt0 = mock(StepModel.class);
        when(pt0.toString()).thenReturn("P=/home/weliu/p0");
         
        StepModel pt1 = mock(StepModel.class);
        when(pt1.toString()).thenReturn("P=/home/weliu/p1");
        StepModel et = mock(StepModel.class);
        when(et.toString()).thenReturn("E90");
         
        StepModel dep = mock(StepModel.class);
        when(dep.toString()).thenReturn("C3.4");
         
        StepModel planar = mock(StepModel.class);
        when(planar.toString()).thenReturn("P_MOST");
         
        StepModel substrate = mock(StepModel.class);
        when(substrate.toString()).thenReturn("S");
         
        SimulationItem item3 = mockItem3(Lists.newArrayList(et, planar, substrate, pt0, dep));
        SimulationItem item2 = mockItem2(Lists.newArrayList(planar, pt1, dep));
        SimulationItem item1 = mockItem1(Lists.newArrayList(et, pt0, dep));
        SimulationItem item0 = mockItem0(Lists.newArrayList(substrate));
        List<SimulationItem> items = Lists.newArrayList(item3, item2, item1, item0);
        for (final SimulationItem item : items) {
          doAnswer(new Answer<Void>() {
            @Override
            public Void answer(InvocationOnMock invocation) throws Throwable {
              Object[] args = invocation.getArguments();
              if (args != null && args[0] instanceof StepModel) {
                when(item.getStep()).thenReturn(StepModel.class.cast(args[0]));
              }
              return null;
            }
          }).when(item).setStep(any(StepModel.class));
        }
        return items;
      }
       
      private static void mockOnSetInsection(final SimulationModel simulationModel) {
        doAnswer(new Answer<Void>() {
          @Override
          public Void answer(InvocationOnMock invocation) throws Throwable {
            Object[] args = invocation.getArguments();
            if (args != null && args[0] instanceof SimulationItem) {
              SimulationItem item = (SimulationItem)args[0];
              for (SimulationItem it : simulationModel.getItems()) {
                when(it.isInspected()).thenReturn(it == item);
              }
            }
            return null;
          }
        }).when(simulationModel).setInsectionTo(any(SimulationItem.class));
      }
      private static void mockOnSetOptimized(final SimulationModel simulationModel) {
        doAnswer(new Answer<Void>() {
          @Override
          public Void answer(InvocationOnMock invocation) throws Throwable {
            Object[] args = invocation.getArguments();
            if (args != null && args[0] instanceof SimulationItem) {
              SimulationItem item = (SimulationItem)args[0];
              if (item.supportOptimized()) {
                for (SimulationItem it : simulationModel.getItems()) {
                  if (it.supportOptimized()) {
                    when(it.isOptimized()).thenReturn(it == item);
                  }
                }
              }
            }
            return null;
          }
        }).when(simulationModel).setOptimizedTo(any(SimulationItem.class));
      }
      private static SimulationItem mockItem3(List<StepModel> items) {
        SimulationItem item = mock(SimulationItem.class);
        when(item.getLayerName()).thenReturn("Name3");
        when(item.getText()).thenReturn("");
        when(item.getName()).thenReturn("");
        when(item.supportOptimized()).thenReturn(false);
        when(item.isOptimized()).thenReturn(true);
        when(item.isInspected()).thenReturn(true);
        when(item.getSupportedTypes()).thenReturn(items);
        when(item.getStep()).thenReturn(items.get(1));
        return item;
      }
      private static SimulationItem mockItem2(List<StepModel> items) {
        SimulationItem item = mock(SimulationItem.class);
        when(item.getLayerName()).thenReturn("Name2");
        when(item.getText()).thenReturn("exp2");
        when(item.getName()).thenReturn("AM");
        when(item.supportOptimized()).thenReturn(true);
        when(item.isOptimized()).thenReturn(true);
        when(item.isInspected()).thenReturn(false);
        when(item.getSupportedTypes()).thenReturn(items);
        when(item.getStep()).thenReturn(items.get(0));
        return item;
      }
      private static SimulationItem mockItem1(List<StepModel> items) {
        SimulationItem item = mock(SimulationItem.class);
        when(item.getLayerName()).thenReturn("Name1");
        when(item.getText()).thenReturn("exp1");
        when(item.getName()).thenReturn("Sh");
        when(item.supportOptimized()).thenReturn(false);
        when(item.isOptimized()).thenReturn(false);
        when(item.isInspected()).thenReturn(false);
        when(item.getSupportedTypes()).thenReturn(items);
        when(item.getStep()).thenReturn(items.get(2));
        return item;
      }
      private static SimulationItem mockItem0(List<StepModel> items) {
        final SimulationItem item = mock(SimulationItem.class);
        when(item.getLayerName()).thenReturn("Name0");
        when(item.getText()).thenReturn("");
        when(item.getName()).thenReturn("AM");
        when(item.supportOptimized()).thenReturn(true);
        when(item.isOptimized()).thenReturn(false);
        when(item.isInspected()).thenReturn(false);
        when(item.getSupportedTypes()).thenReturn(items);
        when(item.getStep()).thenReturn(items.get(0));
        return item;
      }
    }

##Result

Run the tested code after the implementation of GUI logic, we can see our GUI is filled with data, GUI selection behavior and allow input. The behavior is the same with the final product, and is as expected in requirement.


![Test Result][3]

Now, GUI is finished as an independent module. Next time when we develop model module, we can keep the UI interface and fill in the implementation, following TDD principle with a unit test file XXXModelTest.


  [1]: /assets/image/2015-04/TDD_practice_in_UI_img1.png
  [2]: /assets/image/2015-04/TDD_practice_in_UI_img2.png
  [3]: /assets/image/2015-04/TDD_practice_in_UI_img3.gif
