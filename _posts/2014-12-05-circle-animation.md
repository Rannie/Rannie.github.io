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
    
接下来开始编写其中的动画代码

    func animateTransition(transitionContext: UIViewControllerContextTransitioning) {
        //1
        self.transitionContext = transitionContext
        
        //2
        var containerView = transitionContext.containerView()
        var fromViewController = transitionContext.viewControllerForKey(UITransitionContextFromViewControllerKey) as ViewController
        var toViewController = transitionContext.viewControllerForKey(UITransitionContextToViewControllerKey) as ViewController
        var button = fromViewController.button
        
        //3
        containerView.addSubview(toViewController.view)
        
        //4
        var circleMaskPathInitial = UIBezierPath(ovalInRect: button.frame)
        var extremePoint = CGPoint(x: button.center.x - 0, y: button.center.y - CGRectGetMaxY(toViewController.view.bounds))
        var radius = sqrt((extremePoint.x * extremePoint.x) + (extremePoint.y * extremePoint.y))
        var circleMaskPathFinal = UIBezierPath(ovalInRect: CGRectInset(button.frame, -radius, -radius))
        
        //5
        var maskLayer = CAShapeLayer()
        maskLayer.path = circleMaskPathFinal.CGPath
        toViewController.view.layer.mask = maskLayer
        
        //6
        var maskLayerAnimation = CABasicAnimation(keyPath: "path")
        maskLayerAnimation.fromValue = circleMaskPathInitial.CGPath
        maskLayerAnimation.toValue = circleMaskPathFinal.CGPath
        maskLayerAnimation.duration = self.transitionDuration(transitionContext)
        maskLayerAnimation.delegate = self;
        maskLayer.addAnimation(maskLayerAnimation, forKey: "path")
    }
    

分步介绍： <br/>
1. 创建一个指向 transitionContext 的引用 <br/>
2. 获得一些引用，包括 containnerView , fromViewController , toViewController , button 。 <br/>containerView 是动画发生的视图。在动画时， fromViewController 和 toViewController 是相同的部分。
3. 将 toViewController 作为子视图添加到 containerView 上。 <br/>
4. 创建两个圆形的 UIBezierPath 实例；一个是 button 的 size ，另外一个则拥有足够覆盖屏幕的半径。最终的动画则是在这两个贝塞尔路径之间进行的。 <br/>
5. 创建一个 CAShapeLayer 来负责展示圆形遮盖。将它的 path 指定为最终的 path 来避免在动画完成后会回弹。 <br/>
6. 创建一个关于 path 的 CABasicAnimation 动画来从 circleMaskPathInitial.CGPath 到 circleMaskPathFinal.CGPath 。我们也指定了它的 delegate 来在完成动画时做一些清除工作。

接下来，我们实现 *animationDidStop()* 方法来做一点的清除。

    override func animationDidStop(anim: CAAnimation!, finished flag: Bool) {
		self.transitionContext?.completeTransition(!self.transitionContext!.transitionWasCancelled())
        self.transitionContext?.viewControllerForKey(UITransitionContextFromViewControllerKey)?.view.layer.mask = nil
    }
 
第一行告诉 iOS 这个 transition 完成。因为完成后我们才可以清除 mask 。

最后一步我们要使用这个 *CircleTransitionAnimator* 。
回到 NavigationControllerDelegate.swift 然后我们来修改下面方法。

    func navigationController(navigationController: UINavigationController, animationControllerForOperation operation: UINavigationControllerOperation, fromViewController fromVC: UIViewController, toViewController toVC: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return CircleTransitionAnimator()
    }
    
运行程序，你将会看到下面的酷炫效果！

![xcodeimg](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/circle.gif)

###An Interactive Gesture Animation

一旦你的动画运行完成，你可能会把你的注意力放到另外一个 ViewController 切换的特性上: 一个允许交互的"返回"手势。由于只是点击已经过时，请确保你实现这个深度定制的 UI .

交互手势开始会调用 *navigationController:interactionControllerForAnimationController:* 方法，这是导航控制器代理的一个方法，会返回一个 *UIViewControllerInteractiveTransitioning* 的对象。

iOSSDK 提供给你一个类名字为 **UIPercentDrivenInteractiveTransition** ，它已经实现了上面的协议，而且可以让我们对交互手势做很多处理。

打开 NavigationControllerDelegate.swift ，然后添加一个新属性和一个新方法。

    var interactionController: UIPercentDrivenInteractiveTransition?
    
    func navigationController(navigationController: UINavigationController, interactionControllerForAnimationController animationController: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
        return self.interactionController
    }
    
现在回来继续思考这个交互手势。显而易见的是，你需要一个 gesture recognizer 。

![funimg](http://cdn2.raywenderlich.com/wp-content/uploads/2014/12/i_knew_that-e1415860798288-326x320.png)

你将会在导航视图控制器的代理中对导航控制器添加手势，那么首先我们需要一个导航控制器的引用。
首先在 NavigationControllerDelegate.swift 添加一个属性。

    @IBOutlet weak var navigationController: UINavigationController?
    
然后在 storyboard 中进行连线。

![xcodeimg](http://cdn5.raywenderlich.com/wp-content/uploads/2014/10/Screenshot-2014-10-24-02.18.12-700x449.png)

回到 NavigationControllerDelegate.swift 实现 *awakeFromNib()* 方法。

    override func awakeFromNib() {
        super.awakeFromNib()
        
        var panGesture = UIPanGestureRecognizer(target: self, action: Selector("panned:"))
        self.navigationController!.view.addGestureRecognizer(panGesture)
    }
    
创建了一个 pan gesture 。 然后实现它的 action 。

    @IBAction func panned(gestureRecognizer: UIPanGestureRecognizer) {
        switch gestureRecognizer.state {
        //1
        case .Began:
            self.interactionController = UIPercentDrivenInteractiveTransition()
            if self.navigationController?.viewControllers.count > 1 {
                self.navigationController?.popViewControllerAnimated(true)
            } else {
                self.navigationController?.topViewController.performSegueWithIdentifier("PushSegue", sender: nil)
            }
        
        //2
        case .Changed:
            var transition = gestureRecognizer.translationInView(self.navigationController!.view)
            var completionProgress = transition.x / CGRectGetWidth(self.navigationController!.view.bounds)
            self.interactionController?.updateInteractiveTransition(completionProgress)
            
        //3
        case .Ended:
            if (gestureRecognizer.velocityInView(self.navigationController!.view).x > 0) {
                self.interactionController?.finishInteractiveTransition()
            } else {
                self.interactionController?.cancelInteractiveTransition()
            }
            self.interactionController = nil
        
        //4
        default:
            self.interactionController?.cancelInteractiveTransition()
            self.interactionController = nil
        }
    }

分步介绍: <br/>
1. 开始的时候实例化了一个 **UIPercentDrivenInteractionTransition** 对象赋值给 interactionController 属性。 如果是 pop 比较容易, push 则需要执行 Segue 场景来实现。 另外角度说， push 或者 pop 会触发导航控制器的代理方法，返回 self.interactionController ，所以这个属性这时不会为空。 <br/>
2. 在变化的时候，你只需要根据进度去 update interactionController 即可，不需要做任何更多的复杂工作。 <br/>
3. 在结束时，根据当时的速率方向来确定完成这次交互或者取消。然后做清除工作。 <br/>
4. default只是做了一些清除工作。

然后运行 app . 就可以用手指来控制这个动画了哦！

![xcodeimg](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/circle-interactive.gif)


注:这篇博客翻译自 raywenderlich.com 中的 [How To Make A View Controller Transition Animation Like in the Ping App][1]

以上为本篇博客全部内容,欢迎提出建议,个人联系方式详见[关于][2]。



[1]:http://www.raywenderlich.com/86521/how-to-make-a-view-controller-transition-animation-like-in-the-ping-app
[2]:http://rannie.github.io/about


