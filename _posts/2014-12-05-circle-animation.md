---
layout: post
title:  "如何做出像 Ping 一样的 View Controller 转场动画"
date:   2014-12-05 09:08:00
categories: iOS

---

匿名社交应用 Secret 最近发布了一个新的应用 Ping ，它允许用户接收他们感兴趣的话题的通知。 Ping 的主界面到菜单间的圆形切换动画很出色，如下图所示。

![pingimg](http://cdn4.raywenderlich.com/wp-content/uploads/2014/12/ping.gif)

>> Every time I see a really neat animation, I do what everybody does: I think “Now how would I implement that on iOS…” — wait, normal people don’t think that?! :]

在这个教程中，我们将使用 Swift 来实现这个 Cool Animation 。 在这个过程中，将涉及到 *shape layers* , *masking* , *UIViewControllerAnimatedTransitioning* 协议，类 *UIPercentDrivenInteractiveTransition* 以及更多。

###Overall Strategy

在 Ping 中，动画发生在视图控制器的切换过程中。

在iOS中，你可以给两个由 UINaviegationController 的视图控制器间切换加上自定义的动画。实现则是通过 iOS7 以后出现的 *UIViewControllerAnimatedTransitioning* 协议来定义动画。

细节下面会涉及到，但是需要知道这个协议能做什么:

 * 指定动画的持续时间
 * 创建一个容器视图，包含对两个视图控制器的引用
 * 自由的设计动画
 
###Implementation Strategy

如果我用言语来描述这个动画时，可能会这么形容：

 * 有一个圆圈源自右上角的按钮，它作为一个视图展示的视窗。
 * 换句话说, mask 作为遮盖，显示所有内部的东西，并隐藏所有范围外的事物。
 
你可以实现这个效果通过在 CALayer 上使用遮盖。它的 alpha 通道来决定哪部分显示哪部分不显示。

![pingimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/mask-diagram-700x381.png)

既然我们知道通过 mask 来实现，下一步就要决定选择什么样的 mask 。 由于它是圆形的，所以我们要选择 CAShapeLayer 。 这个动画就是给它增加半径并显示的过程。

###Getting Started

现在开始动手去做，选择 **Single View Application** 模板。

![xcodeimg](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/step-1-700x411.png)

选择 **Swift** 作为开发语言。

![xcodeimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/step-2-700x414.png)

打开 Main.storyboard 发现有一个单一的视图控制器，但是我们需要至少两个。我们选中视图控制器，然后通过 *Editor\Embed In\Navigation Controller* 来将其由导航视图控制器管理。
然后选中导航视图控制器，将其设为入口，并隐藏导航栏。

![xcodeimg](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-25-03.25.11-337x320.png)

现在我们再添加一个视图控制器到画布中。

![xcodeimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/12/ViewControllers.png)

然后我们将新的视图控制器的 Custom Class 设为 ViewController 。

![xcodeimg](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-03.20.25-273x320.png)

接下来为两个视图控制器右上角添加按钮。

![xcodeimg](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-12.16.11.png)

背景颜色设为黑色后，来为他们添加约束。
边距为10，宽高44，然后通过 *Update Frames* 来更新位置。

![xcodeimg](http://cdn1.raywenderlich.com/wp-content/uploads/2014/12/PinConstraints2.png)

最后是来配置按钮的形状，我们不需要通过编码来设置，只需要在 IB 中设置一下 attributes 就可以。 IB 中并不会把按钮显示为圆形，但是你可以运行程序来观看效果。

现在我们需要为两个视图控制器添加一些内容，为何不先设置背景颜色呢？

![xcodeimg](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-13.42.48-700x412.png)

然后添加两个图像视图，添加约束（视图居中显示，宽高300点）。完成后如下图。

![xcodeimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-25-14.36.26-700x475.png)

添加两个图片。

![xcodeimg](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/2014-10-25_15-40-43-700x444.png)

###Wire It Up

开始连线添加行为。将首个视图控制器的按钮 action 指定为 show 第二个视图控制器。

![xcodeimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-14.33.40-495x500.png)

为了将来在代码中访问到这个 Segue (场景) ，我们需要指定一个 Segue ID 为 PushSegue 。

![xcodeimg](http://cdn1.raywenderlich.com/wp-content/uploads/2014/11/Screen-Shot-2014-11-04-at-11.13.41-PM.png)

接下来我们要添加 pop 的代码以及 button 的引用在 ViewController.swift 中。

	@IBOutlet weak var button: UIButton!
    @IBAction func circleTapped(sender: UIButton) {
        self.navigationController?.popViewControllerAnimated(true)
    }

然后链接到 IB 中。

![xcodeimg](http://cdn5.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-15.04.58-590x500.png)
![xcodeimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-21.01.32-587x500.png)

然后运行程序，则可以 Push 和 Pop 。

![xcodeimg](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/normal-push.gif)


###Custom Animation

编写一个自定义的 Push 或者 Pop 动画，你需要实现 **UINavigationControllerDelegate** 协议的 **animationControllerForOperation** 方法。 创建一个新的 *iOS\Source\Cocoa Touch Class* 模板文件。

	class NavigationControllerDelegate: NSObject, UINavigationControllerDelegate {
 
	}
	
然后打开 Main.storyboard 来设置一个 UINavigationController 的代理的实例。首先需要在 Navigation Controller Scene 中添加一个 Object 。

![xcodeimg](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-15.39.30-700x403.png)

将其 Custom Class 设置为刚才添加的 NavigationControllerDelegate 

![xcodeimg](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-15.40.25.png)

然后将 NavigationController 的 Delegate 指向这个 Object

![xcodeimg](http://cdn5.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-15.41.18-406x500.png)

回到 NavigationControllerDelegate 添加占位方法

    func navigationController(navigationController: UINavigationController, animationControllerForOperation operation: UINavigationControllerOperation, fromViewController fromVC: UIViewController, toViewController toVC: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return nil;
    }
   
这个方法会得到导航视图控制器切换的两个视图控制器，然后它的工作就是负责实现一个 **UIViewControllerAnimatedTransitioning** 。

之后我们创建一个新的类来实现动画。 CircleTransitionAnimator :

![xcodeimg](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-23-15.52.17-700x410.png)

然后我们需要让这个类遵守协议，并添加 required 方法。

	weak var transitionContext: UIViewControllerContextTransitioning?
    
    func transitionDuration(transitionContext: UIViewControllerContextTransitioning) -> NSTimeInterval {
        return 0.5
    }
    
    func animateTransition(transitionContext: UIViewControllerContextTransitioning) {
        
    }
    



###An Interactive Gesture Animation










