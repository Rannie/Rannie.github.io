---
layout: post
title:  "OS X 和 Swift 初始教程（第一部分）"
date:   2015-01-26 16:50:00
categories: iOS

---

本篇博客翻译自[Getting Started With OS X and Swift Tutorial][1]

这里我们要使用 Swift 语言来完成 Mac 应用开发的初步学习。

##Getting Start

使用 Xcode 创建一个 OS X Application 起名为 "ScaryBugsMac"

![xcode](http://cdn5.raywenderlich.com/wp-content/uploads/2014/11/smats-newproject-700x407.png)

![xcode](http://cdn2.raywenderlich.com/wp-content/uploads/2014/11/smats-newproject2-700x404.png)

创建完成后运行程序将会显示：

![snapshot](http://cdn2.raywenderlich.com/wp-content/uploads/2014/11/smats-emptywindow.png)

我们可以看到 Mac 应用与 iOS 还是有比较大的区别：

* 窗口 (Window) 不与手机端一样有一些固定的尺寸，这个是完全可自定义调整的。
* Mac 应用可以有多个窗口，随意你如何安排。

接下来我们来创建一个视图控制器

![xcode](http://cdn1.raywenderlich.com/wp-content/uploads/2014/11/smats-newmasterviewcontroller1-700x406.png)

这个视图控制器继承于 *NSViewController*

![xcode](http://cdn4.raywenderlich.com/wp-content/uploads/2014/11/smats-newmasterviewcontroller2-700x406.png)

然后我们开始通过在 IB 上拖控件来对新建的 xib 文件进行布局。

要做的第一件事是展示一个 Bugs 的列表，在 OS X 上这个控件被称为 NSTableView (类似 iOS 中的 UITableView) 。

如果你熟悉 iOS 编程的话，会发现这里的一个模式就是 UIKit 中许多用户界面类来源于 OS X 中的 AppKit 。它们中许多只是把 NS 前缀改成了 UI 前缀而已。所以作为一个经验，你可以通过更改前缀的方式尝试去发现是不是 iOS 上的控件在 Mac 上都有对应。你会惊讶的发现有 NSScrollView , NSLabel , NSButton 甚至更多。不过注意的是用法在两个平台上可能会有不同，不过苹果正在使得 OS X 开发的 API 越来越像 iOS 上使用的。

下面我们加入一个 NSTableView :

![xcode](http://cdn1.raywenderlich.com/wp-content/uploads/2014/11/smats-addbugstableview-700x419.png)

暂时先不关注 TableView 的尺寸 :

![xcode](http://cdn1.raywenderlich.com/wp-content/uploads/2014/11/smats-addbugstableview2-700x411.png)

下面在 AppDelegate 中将 MasterViewController 的视图添加进去：


	var masterViewController: MasterViewController!

	func applicationDidFinishLaunching(aNotification: NSNotification) {
	masterViewController = MasterViewController(nibName: "MasterViewController", bundle: nil)
	window.contentView.addSubview(masterViewController.view)
	masterViewController.view.frame = (window.contentView as NSView).bounds
	}



我们初始化了一个 MasterViewController 实例然后把它加到了 window 上并且设置了 frame 。

在 OS X 中的 window 总会创建一个名为 contentView 的视图，这个视图会自动的跟随 window 调整大小。如果你想添加自定义的视图，需要给 contentView 添加 subview 。

在设置大小的时候与 iOS 有些不同，这里不通过设置 rootViewController 来匹配大小。


##A Scary Data Model: Organization

接下来我们要为应用来创建数据模型。

首先在程序中添加一个 Model 的文件夹：

![xcode](http://cdn5.raywenderlich.com/wp-content/uploads/2014/11/smats-projectnavigator4-Model.png)

然后我们要规划两个模型:

* ScaryBugData: 包括名称和排行
* ScaryBugDoc: 包括各个尺寸的图片以及数据

##A Scary Data Model: Implementation

创建 ScaryBugData 类

![xcode](http://cdn4.raywenderlich.com/wp-content/uploads/2014/11/smats-createscarybugdata-700x406.png)

类的初始属性及构造器代码


	class ScaryBugData: NSObject {
	  var title: String
	  var rating: Double
	  
	  override init() {
	    self.title = String()
	    self.rating = 0.0
	  }
	  
	  init(title: String, rating: Double) {
	    self.title = title
	    self.rating = rating
	  }
	}


创建 ScaryBugDoc 类


	class ScaryBugDoc: NSObject {
	  var data: ScaryBugData
	  var thumbImage: NSImage?
	  var fullImage: NSImage?
	  
	  override init() {
	    self.data = ScaryBugData()
	  }
	  
	  init(title: String, rating: Double, thumbImage: NSImage?, fullImage: NSImage?) {
	    self.data = ScaryBugData(title: title, rating: rating)
	    self.thumbImage = thumbImage
	    self.fullImage = fullImage
	  }
	}


现在我们有了数据模型，接下来我们需要创建一些数据并保存到一个数组中。

在 MasterViewController 中添加一个属性

	var bugs = [ScaryBugDoc]()


##Scary bug pictures and sample data

现在我们还没有数据，可以先[下载图片][2]。

有了图片之后我们可以创建一些示例数据：

	func setupSampleBugs() {
	    let bug1 = ScaryBugDoc(title: "Potato Bug", rating: 4.0,
	      thumbImage:NSImage(named: "potatoBugThumb"), fullImage: NSImage(named: "potatoBug"))
	    let bug2 = ScaryBugDoc(title: "House Centipede", rating: 3.0,
	      thumbImage:NSImage(named: "centipedeThumb"), fullImage: NSImage(named: "centipede"))
	    let bug3 = ScaryBugDoc(title: "Wolf Spider", rating: 5.0,
	      thumbImage:NSImage(named: "wolfSpiderThumb"), fullImage: NSImage(named: "wolfSpider"))
	    let bug4 = ScaryBugDoc(title: "Lady Bug", rating: 1.0,
	      thumbImage:NSImage(named: "ladybugThumb"), fullImage: NSImage(named: "ladybug"))
	    
	    bugs = [bug1, bug2, bug3, bug4]
	}

然后我们可以在 *appdelegate* 中 contentView addSubview 之前加载这些示例数据。

##A Different Kind of Bug List

下面就是要把我们的示例数据显示到 tableview 中。

在 OS X 中， NSTableView 与 iOS 中的 UITableView 最主要的区别就是 NSTableView 可以显示多列。

打开 MasterViewController.xib 确保 tableview 的 Content Mode 是 View Based, 配置不显示头部，勾选 Alternating Rows 来保证行交替颜色变化，

![xcode](http://cdn5.raywenderlich.com/wp-content/uploads/2014/11/smats-tableviewproperties-700x403.png)

下一步是配置 tableview 的 cell 。我们需要一个图片和一个文本区域来显示信息。 NSTableCellView 已经存在这种类型的 cell 我们可以直接使用。打开 IB 对象面板寻找 *Image & Text Table Cell View* 然后拖到 TableView 中。

然后我们调整尺寸，设置 Cell 的 Identifier :

![xcode](http://cdn3.raywenderlich.com/wp-content/uploads/2014/11/smats-tableviewcellproperties-700x470.png)

![xcode](http://cdn2.raywenderlich.com/wp-content/uploads/2014/11/smats-bugcolumn-700x470.png)

下一步我们要给 TableView 设置数据源和代理

![xcode](http://cdn1.raywenderlich.com/wp-content/uploads/2014/11/smats-connectdelegate-700x448.png)

最后就是写 TableView 的方法了

	extension MasterViewController: NSTableViewDataSource {
	  func numberOfRowsInTableView(tableView: NSTableView) -> Int {
	    return self.bugs.count
	  }
	  
	  func tableView(tableView: NSTableView, viewForTableColumn tableColumn: NSTableColumn?, row: Int) -> NSView? {
	    var cellView: NSTableCellView = tableView.makeViewWithIdentifier(tableColumn?.identifier, owner: self) as NSView!
	    
	    if tableColumn?.identifier == "BugColumn" {
	      let bugDoc = bugs[row] as ScaryBugDoc
	      cellView.imageView!.image = bugDoc.thumbImage
	      cellView.textField!.stringValue = bugDoc.data.title
	    }
	    
	    return cellView
	  }
	}

两个数据源方法，跟 UITableView 的用法极其相似，唯一不同在于 Cell 的数据源方法稍有不同，由于有多列的情况可能存在，所以多了一个列的参数。其余的都很相近。

然后运行程序

![xcode](http://cdn3.raywenderlich.com/wp-content/uploads/2014/11/smats-part1app.png)

这样我们的第一个使用 Swift 编写的 OS X 应用就初步完成了。

-- EOF --

欢迎提出建议,个人联系方式详见[关于][3]。

[1]:http://www.raywenderlich.com/87002/getting-started-with-os-x-and-swift-tutorial-part-1
[2]:http://d1xzuxjlafny7l.cloudfront.net/downloads/ScaryBugs.zip
[3]:http://rannie.github.io/about