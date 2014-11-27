---
layout: post
title:  "App Extensions 学习笔记 --- 简介"
date:   2014-11-26 15:47:00
categories: iOS

---


应用扩展(App Extension)是iOS8最值得人期待的功能之一。它们让开发者在整个操作系统的其他部分扩展应用程序的内容和功能。iOS8是一个开放的平台允许用户在他们的设备上进行更多的交互。应用扩展使开发人员能够在他们没有自己的应用程序的地方提供自定义功能的权利，甚至包括Apple的股票应用。

应用程序扩展会对用户如何创建，修改和分享他们的设备的内容产生了巨大影响。事实上，应用程序扩展的实现十分完美虽然终端用户不会有任何体会，但是这个功能是革命性的。


###扩展类型

应用扩展有六种不同的类型，每一个提供了从你的应用扩展到其他应用或者操作系统的功能。

####Today Extension

通常被成为"组件",Today Extension出现在通知中心的今日视点。这些小视图控制器包含着信息片段让用户一目了然。我们已经看到过苹果自带股票和天气等小"组件"(Widget)的形式。

下图展示了一个跟踪比特币价格的应用，在应用中以及TodayExt中展示相同信息的样式。

![ext iamge](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112605.png)


####Share Extension

如名字所示，分享扩展让应用程序和其他服务之间共享内容。此功能引入于iOS5,但是之前也局限于分享图片，分享到Tweet,FB,Weibo等。

下图为新的服务集成例子，一个图片快速分享到imgur.com。

![ext iamge](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112606.png)

####Action Extension

行为扩展一般在分享扩展的下面面那行。一些快速的行为。
例如生成短链接在无需离开Safari的情况下。

![ext iamge](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112607.png)