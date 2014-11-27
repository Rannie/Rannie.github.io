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

####Photo Editing Extension

现在，用户可以使用照片编辑扩展来对照片进行快速编辑。照片编辑扩展可以让我们在编辑照片时使用外部程序，一旦满意，即可以点击保存在完成编辑。而且用户也不用担心外部程序会破坏原有照片，编辑扩展可以让用户恢复原样，甚至提供了一个完整的编辑堆栈，可以一步一步回退。

下图为图片编辑扩展示例。

![ext iamge](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112701.png)

####Document Provider Extension

在iOS8中，应用程序可以提供一个文件选择器(Document Picker)视图控制器。这个选择器允许用户导入、导出、打开外部和移动本地文件。作为一个开发者，你可能希望你的应用程序的文件可以提供给其他应用。想做到这一点，你需要写一个**DocumentProviderExtension**,一旦你的扩展被安装，用户可以从您的应用程序选择文件进行交互或他们的文件导出到你的存储。

举个例子，一个日志应用程序可能让用户保存他们的日记条目到他们的Dropbox账户。iOS8之前，你需要利用Dropbox API和SDK来实现这一功能。但如果Dropbox提供这个扩展，那么日志应用程序可以使用这项服务在内部没有使用任何API和SDK。通过这个扩展，那些云提供商通过提供扩展可以让服务变得更加有价值和实用。

下图示例，展示将图片上传到亚马逊S3，或者iCloud等。

![ext iamge](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112702.png)

####Custom Keyboard Extension

在iOS8里，我们可以创建系统级别的自定义键盘，像搜狗输入法，百度输入法做的那样。在以前的键盘中，只是可以选择语言和类似Emoji的表情符号，但是现在可以有选择第三方键盘的能力。

下图展示了一个自定义键盘。

![ext iamge](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014112703.png)


###探究 App Extensions 的真实面目

上面的描述都是通过终端用户的视角来观察应用扩展的。但是他们如何工作呢？如何打包和交付？

正如它的名字所描述，一个 App Extension 就是你的应用的功能扩展,*他们并不是独立的工具或者插件*,扩展们提供于你的App Bundle而不是独立的app,我们可以视作其为程序的内部应用，或者将程序称为应用容器。

不过即使应用扩展是伴随着你的应用程序的，不过他们运行并不是与应用运行在同一个进程中。相反，每一个扩展运行在自己一个独立的进程中。不同于应用程序在 iOS 上的工作方式，一个扩展可以启动多次在不同的进程中。

考虑以下情形：您的用户正在使用照片，并希望通过您分享扩展来分享图片。然后，用户切换到 Safari 浏览器，并选择使用相同的扩展分享不同的内容。这在系统资源允许的情况下，是完全可以实现的一个扩展的两个并行在系统中的实例。iOS 保证这些实例在不同进程中所以你不需要来管理这两个实例之间的状态。

应用扩展提供自己的二进制文件（独立于应用程序的二进制文件）。在 Xcode 中，需要为一个 app extension target 也需要一个 identifier。创建很简单，如下图所示：


