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

给按钮在代码文件中添加引用以及 Action .

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120912.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120913.png)

然后连接 IB 中的 Label 到 Controller 的属性，并设置约束。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120914.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120915.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120916.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120917.png)

我们把按钮距离底部的约束的优先级降低。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120918.png)

给底部的 Line Chart 视图添加约束，并获取其中高度约束的引用。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120919.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120920.png)

之后我们在 TodayViewController 视图加载后进行一些初始赋值。

    override func viewDidLoad() {
        super.viewDidLoad()
        
        lineChartHeightConstraint.constant = 0
        
        lineChartView.delegate = self
        lineChartView.dataSource = self
        
        priceLabel.text = "--"
        priceChangeLabel.text = "--"
    }
    
视图显示后再加载数据源，并进行视图显示更新。

    override func viewDidAppear(animated: Bool) {
        super.viewDidAppear(animated)
        
        fetchPrices { error in
            if error == nil {
                self.updatePriceLabel()
                self.updatePriceChangeLabel()
                self.updatePriceHistoryLineChart()
            }
        }
    }
    
同时要通过一个代理方法，给这个视图控制器设置这个组件的 Edge Insets。

    func widgetMarginInsetsForProposedMarginInsets(defaultMarginInsets: UIEdgeInsets) -> UIEdgeInsets {
        return UIEdgeInsetsZero
    }

运行 Extension 。 等待读取数据后会发现如下所示：

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120921.png)

可以注意到，这里还没有底部的 Line Chart .

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120922.png)

在点击按钮的 Action 中我们添加显示 Line Chart 的代码。

    @IBAction func toggleLineChart(sender: AnyObject) {
        if lineChartIsVisible {
            lineChartHeightConstraint.constant = 0
            let transform = CGAffineTransformMakeRotation(0)
            toggleLineChartButton.transform = transform
            lineChartIsVisible = false
        } else {
            lineChartHeightConstraint.constant = 98
            let transform = CGAffineTransformMakeRotation(CGFloat(M_PI))
            toggleLineChartButton.transform = transform
            lineChartIsVisible = true
        }
    }

同时在控制器每次 layout subviews 之后更新 Line Chart 上的数据。

    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        updatePriceHistoryLineChart()
    }

然后运行程序，点击按钮的前后对比：

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120923.png)

这时我们发现这个线还是白色（默认），可以通过一个代理方法，设置这个 Line 的颜色。

    override func lineChartView(lineChartView: JBLineChartView!, colorForLineAtLineIndex lineIndex: UInt) -> UIColor! {
        return UIColor(red: 0.17, green: 0.49, blue: 0.82, alpha: 1.0)
    }

更改颜色后样子为：

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014120924.png)


基本上我们需要的功能就全部完成。
不过我们会注意到，在 Xcode 为我们生成的模板代码中，有这么一段：

	func widgetPerformUpdateWithCompletionHandler(completionHandler: ((NCUpdateResult) -> Void)!) {  
	    // Perform any setup necessary in order to update the view.
	
	    // If an error is encoutered, use NCUpdateResult.Failed
	    // If there's no update required, use NCUpdateResult.NoData
	    // If there's an update, use NCUpdateResult.NewData
	
	    completionHandler(NCUpdateResult.NewData)
	}

它的作用是什么呢？

>>对于通知中心扩展，即使你的扩展现在不可见 (也就是用户没有拉开通知中心)，系统也会时不时地调用实现了 NCWidgetProviding 的扩展的这个方法，来要求扩展刷新界面。这个机制和 iOS 7 引入的后台机制是很相似的。在这个方法中我们一般可以做一些像 API 请求之类的事情，在获取到了数据并更新了界面，或者是失败后都使用提供的 completionHandler 来向系统进行报告。


所以我们也要在这里面添加刷新页面的代码 :

    func widgetPerformUpdateWithCompletionHandler(completionHandler: ((NCUpdateResult) -> Void)!) {
        fetchPrices { error in
            if error == nil {
                self.updatePriceLabel()
                self.updatePriceChangeLabel()
                self.updatePriceHistoryLineChart()
                completionHandler(.NewData)
            } else {
                completionHandler(.NoData)
            }
        }
    }
    
最后，我们经常需要在点击我们的 Today Extension 的某些组件的时候，能够跳入指定的程序，并传入一些参数跳到指定的页面。这需要我们为我们的 Container App 添加 URLSchema 然后让这个 Extension 触发某些行为时去 *OpenURL:* 即可。不过在 Extension 的 Controller 中， **UIApplication.sharedApplication()** 是 Unavailable 的。而苹果为扩展增加了一个 extensionContext 的引用，我们使用它来完成这个操作。

可以发现 Class NSExtensionContext 中有这样一个 method :

    // Asks the host to open an URL on the extension's behalf
    func openURL(URL: NSURL, completionHandler: ((Bool) -> Void)?)
    
实现这个功能时，先注册 URLSchema :

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014121101.png)

之后我们在事件中直接 *OpenURL：* 即可。

    @IBAction func toggleOpenApp(sender: AnyObject) {
        let url = NSURL(string: "rannieTest://test")
        extensionContext?.openURL(url!, completionHandler: nil)
    }
    
至此，大部分的 Today Extension 功能都已实现。关于通知与 Container App 之间共享数据或者代码，可参考 [WWDC 2014 Session笔记 - iOS 通知中心扩展制作入门][4] 。

以上为本篇博客全部内容,欢迎提出建议,个人联系方式详见[关于][5]。


[1]:https://bitcoin.org/en/
[2]:https://github.com/Rannie/AppExtensions/tree/master/Today%20Extension
[3]:http://rannie.github.io/ios/2014/11/26/app-extension-introducing.html
[4]:http://onevcat.com/2014/08/notification-today-widget/
[5]:http://rannie.github.io/about