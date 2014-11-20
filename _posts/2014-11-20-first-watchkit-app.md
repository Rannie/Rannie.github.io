---
layout: post
title:  "The First AppleWatch Demo"
date:   2014-11-20 16:09:00
categories: iOS

---


苹果昨天放出了Xcode6.2Beta，其中放出了iOS8.2的SDK，以及苹果新产品**AppleWatch**的开发包**WatchKit**，今天有时间就尝试了下WatchKit开发。

###WatchKit初探


AppleWatch上的应用有点类似于iOS8推出的一系列Extensions，我们可以将其看做是iOS APP Extension。所以创建一个WatchKit应用也要基于一个手机上的应用才可以。

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112008.png)

创建一个WatchKit App Target:

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112001.png)

可以看到左侧在*Other*下面新增了一个AppleWatch类别，其中有Watch App模板。

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112002.png)

创建完成后,左侧的文件目录多了两个文件夹*WatchDemo WatchKit Extension*和*WatchDemo Watch App*

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112003.png)

前者负责存放一些代码文件，后者有一个storyboard负责UI展示布局。在WatchKit中，苹果不允许使用代码来创建UI，只允许通过操作InterfaceBuilder来进行。


好了，来进入WatchKit Framework来一探究竟。

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112004.png)

里面的头文件并不多,大多是我们平时打交道的视图元素的Watch版。比如我们去看看常用的Label，它继承自*WKInterfaceObject*,而它自己独有的方法也并不多，设置文字，颜色，以及AttributedString.

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112005.png)

*WKInterfaceLabel*的父类*WKInterfaceObject*是大多数视图元素的父类，他有一些常用的视图方法，显示/隐藏，设置透明度，宽高等。需要注意的是，它的初始化构造器被标记为了*NS_UNAVAILABLE*,也就是上文提到的，我们没有办法在Runtime时去初始化一个视图元素，我们只能去设置他们的显示逻辑，比如文字，颜色等。

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112006.png)


接下来看下InterfaceBuilder,Storyboard中包含了一个MainInterface和一个Glance Interface.

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112007.png)

MainInterface用于我们布局主页面，而Glance主要负责显示，正如其意『一目了然』，在这个页面上我们不允许交互，只用来展示一些信息。

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112014.png)

> Glance是Watch app和WatchKit扩展的一部分，你的glance界面位于Watch app的storyboard文件当中，并且这个界面被自定义的WKInterfaceController对象管理。需要注意的是，这个glance界面控制器只负责设置glance中的内容，Glance不支持互动操作，触摸glance将会自动启动对应的Watch app。

####Glance界面指南

Xcode提供几种固定的布局来安排glance中的内容，在选定适合你的一种布局后，遵循下面的指南来填充内容：

· Glance的设计目的在于快速的传达信息。不要显示一堆文字。适当的使用图像、颜色和动画来快速传达信息。

· 聚焦在最重要的数据上。Glance不是你的应用的替代。就像Watch app是对应的iOS app的缩水版，你也可以把glance看做Watch app的缩水版。

· 不要在glance界面中包含交互控件。比如按钮、选择器、滑动器和菜单。

· 避免使用表格和地图。尽管并没有禁止你这么做，手表上有限的控件让表格和地图不是那么有用。

· 让显示的信息保持及时。使用所有可用的资源，包括时间和地理位置，来向用户提供有用的信息。并且注意更新你的glance，以避免因为glance初始化到显示花费的时间而让信息过时。

一个app只允许有一个glance界面控制器，因此你需要在这一个控制器中显示所有你希望展示的内容。


###生命周期问题

正如iOS开发中，第一个需要了解的是Controller的生命周期一样，在WatchKit中Controller的生命周期也是我们编写应用的生命线。
模板里默认给出了一些方法，类似于*viewDidLoad*,*viewWillAppear*和*viewDidDisappear*.

![watch image](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112015.png)


###第一个AppleWatch应用