---
layout: post
title: "Using Eclipse Spy in GUI products based on RCP"
description: ""
category: Eclipse插件
tags: [Spy]
refer_author: Wenzhen
refer_blog_addr: 
refer_post_addr: 
---



Like Code Tips or wxSpy++, how to find out the code related to a window of the current cursor in a GUI product based on Eclipse RCP technology?

In Eclipse IDE, we can press Alt+Shift+F1 to open Eclipse-Plugin Spy Window to show the window class of the current cursor. See the following screenshot:

![](/assets/image/2014-11/espy-1.png)

My idea is that we can re-use Eclipse-Plugin Spy in our own project/product when we are developing, or even as an optional dynamic plugin into our final release package, so that we can see more class information in GUI directly when developing or released.

We can see Eclipse-Plugin Spy is defined in plugin org.eclipse.pde.runtime. Therefore, the solution is just to include this plugin and its dependency (org.eclipse.core.runtime, org.eclipse.ui, and org.eclipse.ui.forms) into your application.

When developing, we can open Debug/Run Configuration and add the plugin org.eclipse.pde.runtime and its required plugins.

![](/assets/image/2014-11/espy-2.png)

After product released, if we want to use it, we can also add the plugin and its dependencies into the plugins folder.

Here is the screenshot in D4C GUI after using the spy.

![](/assets/image/2014-11/espy-3.png)

We can see the dialog class is AnalysisChartSettingDialog, when clicking the hyper-link, the class file will be opened in our D4C GUI application (not Eclipse IDE), so that we can read the file directly even without Eclipse IDE.


![](/assets/image/2014-11/espy-4.png)
