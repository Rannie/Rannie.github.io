---
layout: post
title:  "App Extensions 学习笔记 --- Today Extension"
date:   2014-12-09 11:26:00
categories: iOS

---


这篇文章通过编写一个 Today Extension 来呈现基于美元的 Bitcoin ([https://bitcoin.org/en/][1]) 价格。

###Starter Project

初始项目在可以去 [Today Extension][2] 下载。
在这个项目里，我们不会更多的关注于 container app 的开发，而是更多的关注于编写 Today Extension 。 如果不太清除 container app 的概念，可以看上一篇关于 App Extension 的文章：[App Extensions 学习笔记 --- 简介][3] 。

运行程序，会得到即时的价格通过 web 服务。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120901.png)

###BTC Widget

BTC 即为 Bitcoin 的缩写。而即将编写的 Today Extension 则是这个应用视图的预览版本。

####Add a Today Extension Target

首先我们要在项目中新添加一个 Target ，选择 *Application Extension* 模板集合，然后找到 *Today Extension* 。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120902.png)

按步建立完成后，可以看到项目中的 Targets 中，新增加了一个我们命名为 *BTC Widget* 的 target 。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120903.png)

接下来需要将涉及到获取数据的框架连接到项目中。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120904.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120905.png)

最后我们将注意力集中到 **BTC Widget** 这个文件夹上。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120906.png)

打开 *MainInterface.storyboard* 会发现有一个高度很小的视图，其中嵌套着一个 text 为 Hello World 的 label.

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120907.png)

运行程序， 选择 Today , 然后可以发现设备（或者模拟器）会显示今日面板，其中有我们新建立的 Today Extension.

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120908.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120909.png)

接下来我们进行一些页面布置工作，让视图看起来如下所示：

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120910.png)

按钮的图片选择 assets 中的图片 ， 同时要确保 assets 能够给这个 target 提供图片。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120911.png)

接下来，我们让 *TodayViewController* 继承 *CurrencyDataViewController* 。

	import CryptoCurrencyKit
	
	class TodayViewController: CurrencyDataViewController, NCWidgetProviding {
		//...
	}

然后连接 IB 中的 Label 到 Controller 的属性，并设置约束。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120912.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120913.png)


![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120914.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120915.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120916.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120917.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120918.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120919.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120920.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120921.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120922.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120923.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120924.png)




[1]:https://bitcoin.org/en/
[2]:https://github.com/Rannie/AppExtensions/tree/master/Today%20Extension
[3]:http://rannie.github.io/ios/2014/11/26/app-extension-introducing.html