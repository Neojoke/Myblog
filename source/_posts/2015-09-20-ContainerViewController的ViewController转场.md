---
title: ContainerViewController的ViewController 转场
date: 2015-09-20 01:28:00
categories:
- 表-iOS界面开发应用
tags:
- iOS
- UI
---

又过去十天了，更新的速度有点慢，很不好意思，自己并不是一个高产的程序员，以后一定要加油了。
上篇文章 [Implementing a Container View Controller 翻译+自我实践](http://www.jianshu.com/p/68a589e244e4) 中解释了如何实现一个简单的容器类视图控制器，这篇文章讲讲自定义容器类视图控制器如何实现子视图控制器的转场切换的。

按照惯例，还是说一些基础知识，iOS 中的转场，指的是视图控制器的转场。
官方文档对 Transition（转场）有专门的章节进行介绍，希望初学的开发者仔细阅读，理解一些基本的概念和原理：
[The Presentation and Transition](https://developer.apple.com/library/ios/featuredarticles/ViewControllerPGforiPhoneOS/PresentingaViewController.html#//apple_ref/doc/uid/TP40007457-CH14-SW1)

转场其实就是使得 ViewController 显示在屏幕上，一共有两种方式：

1.  present 一个视图控制器，在日常开发中，主要体现在使用一个 ViewController 模态跳转到另外一个 ViewController
2.  在 ContainerViewController 中显示一个视图控制器，这种体现在 navigationController 和

tabbarController 等容器类视图控制器中切换视图控制器，例如导航的 push 与 tabbarController 的 select 一个视图控制器，当然我们自定义的 ContainerController 中提供的转场也是属于此类。

一般的开发者有时候只会简单的使用第一种和第二种中系统提供的容器类视图控制器的转场，对转场的原来不是很理解，我们要实现一个自定义容器类视图控制器的转场，必须要把这些都讲明白。其实按照我个人的理解，视图控制器的转场跟我们自己实现一两个 View 之间动画切换差不多，无非就是 ViewController 的 View 在切换。当然，几个 View 的切换要在同一个父视图上进行，那 ViewController 的 View 的切换则要在上篇文章中提到的 ContainerView 上进行，并且需要在转场前和转场后处理好子视图控制器与容器类视图控制器的父子关系的创建与管理，也就是调用`addChildViewController:` 、`didMoveToParentViewController:` 、`willMoveToParentViewController:` 、`removeFromParentViewController` 使得视图控制器的生命周期函数在转场的过程中正确的调用，并且在转场过程中，完成视图控制器的 View 在 ContainerView 上的布局。在上面第一种提到的转场和第二种中系统提供的容器类视图控制器（navigationController 和 tabbarController 等）的转场，系统完成了一些操作，使得视图控制器的生命周期函数得以正确调用，并且在 containerView 上正确的呈现。

在上篇文章中的例子里，我们在左右切换的方法中，调用了创建视图控制器的父子关系的一些函数，并且调用了`transitionFromViewController(fromViewController: UIViewController, toViewController: UIViewController, duration: NSTimeInterval, options: UIViewAnimationOptions, animations: (() -> Void)?, completion: ((Bool) -> Void)?)`来完成视图控制器的 View 在 ContainerView 上的布局和动画切换，该函数是 iOS5 中提供的，在当时用于自定义视图控制器转场的时候来完成动画的转场，例子里的核心代码如下：

````Swift
fromViewController.willMoveToParentViewController(nil);
self.addChildViewController(toViewController);
toViewController.view.bounds = container.bounds;
let endCenter:CGPoint;
fromViewController.transitioningDelegate = self;
if orientation == Orientation.Left{
    toViewController.view.center = CGPointMake(self.view.bounds.size.width + toViewController.view.bounds.size.width/2, self.view.bounds.size.height/2)
    endCenter = CGPointMake(-self.view.bounds.size.width - toViewController.view.bounds.size.width/2, self.view.bounds.size.height/2)
}
else
    {
        toViewController.view.center = CGPointMake(-self.view.bounds.size.width - toViewController.view.bounds.size.width/2, self.view.bounds.size.height/2)
        endCenter = CGPointMake(self.view.bounds.size.width + toViewController.view.bounds.size.width/2, self.view.bounds.size.height/2)
    }
self.transitionFromViewController(fromViewController, toViewController: toViewController, duration: 0.25, options: UIViewAnimationOptions.CurveEaseInOut, animations: { () -> Void in
toViewController.view.frame = fromViewController.view.frame
fromViewController.view.center = endCenter
}) { (completion) -> Void in
    toViewController.didMoveToParentViewController(self)
    fromViewController.removeFromParentViewController()
}
`
其实我们也可以不调用`transitionFromViewController： `这个方法，自己书写视图控制器View在ContainerView上的添加、删除和布局，但一定要注意做这些事情的顺序，以保证视图控制器的生命周期方法正常调用，虽然这是复杂的，但是确实可行的。

等到iOS7之后，苹果为了满足开发者在这方面的需求，将转场的过程进行了协议化，提出了如下几个概念：

1. 动画控制器 (Animation Controllers) 遵从 UIViewControllerAnimatedTransitioning 协议，用来专门处理转场中的动画切换
2. 交互控制器 (Interaction Controllers) 通过遵从 UIViewControllerInteractiveTransitioning 协议，用来处理可交互的转场逻辑
3. 转场代理 (Transitioning Delegates)  遵守UIViewControllerTransitioningDelegate协议，在代理方法中，针对不同类型（模态跳转还是导航push，是可交互的还是不可交互）转场，为转场提供对应的动画控制器或者是交互控制器。
4. 转场上下文 (Transitioning Contexts) 定义了转场时需要的元数据，比如在转场过程中所参与的视图控制器和视图的相关属性。 转场上下文对象遵从 UIViewControllerContextTransitioning 协议，并且这是由系统负责生成和提供的。
转场协调器(Transition Coordinators) 可以在运行转场动画时，并行的运行其他动画。 转场协调器遵从 UIViewControllerTransitionCoordinator 协议。

以上几个概念，都是使用定义在UIViewController 文件里的几组协议，利用这几种概念，我们通过协议能更好的将转场的逻辑合理的进行分离，使得耦合度降低并且使得每一块都专注于本身要处理的逻辑之中，并且能够随意复用，比方说，相同的转场上下文中，我可以在转场代理中提供不同的动画控制器或交互控制器实现不同的转场。同样，我们可以使用如上概念，来自定义模态跳转或者是navigationcontroller和tabbarcontroller的转场，只需要设置好转场代理，在代理中提供不同的动画控制器和交互控制器，而转场上下文，在系统控件的转场中由系统生成，提供给动画或者交互控制器来使用。
以上的流程我们可以总结为：
1. 无论是在任何类型下的转场将要发生的时候，我们在此之前，需要设置转场的代理，例如在模态跳转之前我们需要使用发起模态跳转的视图控制器设置transitioningDelegate来提供转场代理。
2. 在UIViewControllerTransitioningDelegate 代理方法中提供
```Swift
//视图控制器消失的时候，需要提供的动画控制器
func animationControllerForDismissedController(dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
    return nil
}
//视图控制器显示的时候，需要提供的动画控制器
func animationControllerForPresentedController(presented: UIViewController, presentingController presenting: UIViewController, sourceController source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
    return nil
}
//可交互的转场中，视图控制器消失的时候，需要提供的动画控制器
func interactionControllerForDismissal(animator: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
    return nil
}
//可交互的转场中，视图控制器显现的时候，需要提供的动画控制器
func interactionControllerForPresentation(animator: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
    return nil
}
````

以上适用于模态跳转的流程，navigationController 的自定义转场，需要设置的导航视图控制器的 delegate（遵守 UINavigationControllerDelegate），在代理方法中如上第二部提供不同的动画控制器和交互控制器，在此就不一一赘述了。

那么我们在回过头来，看看我们自定义的容器类视图控制器如何实现转场，前面已经提到，我们可以自己调用构建父子关系的函数并且操作 ContainerView 来做动画转场，或者借助 transitionFromViewController 来完成，但这次我们使用动画控制器来实现，好处很明显，在上面已经介绍过了。我将上篇文章中的 Demo 在 github 上做了一个分支，来说明这个做法，我注释掉左右切换处理的函数中老的实现方法，来自定义动画控制器实现切换动画，新 Demo 的地址如下：
[ContainerViewController](https://github.com/Neojoke/ContainerViewController/tree/CustomContext%26TransitionAnimator)

我注释掉之前的逻辑，新的逻辑如下：

```Swift
func swipeFromViewController(fromViewController:UIViewController,ToViewController toViewController:UIViewController, WithOrientation orientation:Orientation)
//第一步，在转场之前，构建fromViewController toViewController和containerViewController的父子关系
    fromViewController.willMoveToParentViewController(nil)
    self.addChildViewController(toViewController)
//第二步，因为我们是自定义的容器控制器，我们需要提供转场上下文
    let context = CustomTransitionContext(containerView: container, toViewController: toViewController, fromViewController: fromViewController)
//第三步，设置转场完成之后，调用一个回调的闭包，来处理父子关系的重新构建
    context.completeHandle = {
        (isComplete : Bool) -> Void in
        toViewController.didMoveToParentViewController(self)
            fromViewController.removeFromParentViewController()
        }
        let animator = CustomTransitionAnimtor(context: context)
        animator.orientaion = orientation
        animator.animateTransition(context)
```

接下来我们看一下我们自定义的转场上下文，一个遵守 UIViewControllerContextTransitioning 协议的 NSObject 对象：

```Swift
class CustomTransitionContext: NSObject,UIViewControllerContextTransitioning
    {
        weak var customContainerView : UIView?
        private weak var toViewController:UIViewController?
        private weak var fromViewController:UIViewController?
        internal var completeHandle : ((isComplete : Bool)->Void)?;
        var animating : Bool = true
        func isAnimated() -> Bool {
            return animating
        }
        func isInteractive() -> Bool {
            return false
        }
        func transitionWasCancelled() -> Bool {
            return false
        }
        func presentationStyle() -> UIModalPresentationStyle {
            return UIModalPresentationStyle.Custom
        }
//完成转场之后要做的操作
        func completeTransition(didComplete: Bool) {
            if let handler = completeHandle {
                animating = false
                handler(isComplete: didComplete)
            }
        }
        func updateInteractiveTransition(percentComplete: CGFloat) {
        }
        func finishInteractiveTransition() {}
        func cancelInteractiveTransition() {}
//转场上下文提供的UITransitionContextFromViewController与           UITransitionContextToViewController
        func viewControllerForKey(key: String) -> UIViewController? {
            switch key{
                case UITransitionContextFromViewControllerKey:
                return fromViewController
                case UITransitionContextToViewControllerKey:
                return toViewController
                default:
                return nil
            }
        }
        @available(iOS 8.0,*)
        func viewForKey(key: String) -> UIView? {
            switch key{
                case UITransitionContextFromViewKey:
                return fromViewController?.view
                case UITransitionContextToViewKey:
                return toViewController?.view
            default:
                return nil
        }
    }
        func finalFrameForViewController(vc: UIViewController) -> CGRect {
            return CGRectZero
        }
        func initialFrameForViewController(vc: UIViewController) -> CGRect {
            return CGRectZero
            }
        func containerView() -> UIView? {
            return self.customContainerView
            }

//提供一个便利构造方法，能够获取转场相关的视图控制器与ContainerView
        convenience init(containerView : UIView? ,toViewController : UIViewController , fromViewController : UIViewController){
            self.init()
            self.toViewController = toViewController
            self.fromViewController = fromViewController
            self.customContainerView = containerView
        }
        func targetTransform() -> CGAffineTransform {
            return CGAffineTransformIdentity
        }
}
```

以上就是转场上下文的构建，带有注释的方法是比较核心的几个方法，而其他的都是为了满足协议而补全的方法。核心就是使得我们自定义的转场上下文拥有 ContainerView 和要切换的视图控制器，提供给动画控制器使用。
下面是重点，动画控制器的实现：

```Swift
class CustomTransitionAnimtor: NSObject ,  UIViewControllerAnimatedTransitioning {
    internal var orientaion : ContainerViewController.Orientation = ContainerViewController.Orientation.Left
    var toViewController : UIViewController?
    var fromViewController : UIViewController?
    var privateContext : CustomTransitionContext;
    //该方法提供转场动画需要的时间
    func transitionDuration(transitionContext: UIViewControllerContextTransitioning?) -> NSTimeInterval {
        return 0.25
    }
    //自定义的构造函数用来获取自定义的转场上下文
    init(context : CustomTransitionContext){
        privateContext = context
        super.init()
    }
    //该方法是用来处理动画逻辑的
    func animateTransition(transitionContext: UIViewControllerContextTransitioning) {
        //获取containerView的引用
        let containerView = privateContext.containerView()
        //获取要转场的相关视图控制器，通过自定义的转场上下文来获取
        let toViewController = privateContext.viewControllerForKey(UITransitionContextToViewControllerKey)
        let fromViewController = privateContext.viewControllerForKey(UITransitionContextFromViewControllerKey)
        //配置相关控制器的View的信息，大小与在Container的添加和删除
        toViewController?.view.bounds = (fromViewController?.view.bounds)!
        var endCenter : CGPoint;
        if orientaion == ContainerViewController.Orientation.Left{
            endCenter = CGPointMake(-(fromViewController?.view.bounds.size.width)!, (containerView?.bounds.size.height)!/2)
            toViewController?.view.center = CGPointMake((containerView?.bounds.size.width)! + (toViewController?.view.bounds.width)!/2, (containerView?.bounds.height)!/2)
        }
        else
        {
            endCenter = CGPointMake((containerView?.bounds.size.width)! + (fromViewController?.view.bounds.size.width)!/2, (containerView?.bounds.size.height)!/2)
            toViewController?.view.center = CGPointMake(-(containerView?.bounds.size.width)! - (toViewController?.view.bounds.width)!/2, (containerView?.bounds.height)!/2)
        }
        containerView?.addSubview((toViewController?.view)!)

                //按照预定的来实现动画切换UIView.animateWithDuration(self.transitionDuration(privateContext), animations: { () -> Void in
            toViewController?.view.center = (fromViewController?.view.center)!
            fromViewController?.view.center = endCenter
            }) { (isComplection) -> Void in
                fromViewController?.view.removeFromSuperview()
                self.animationEnded(true)
        }
    }
        //在完成转场动画的时候，调用自定义转场上下文的completeTransition方法，来告诉上下文转场已经完成
    func animationEnded(transitionCompleted: Bool) {
        self.privateContext.completeTransition(true)
    }
}
```

以上我们清晰的看到，动画控制器有三个方法，依次是提供动画时间，动画过程和动画完成后要做的一些逻辑操作，非常简单，动画控制器就是专一完成切换的动画的逻辑的，代码注释已经相当清楚。
以上的步骤很清晰，但有两个问题，那就是，为什么我们没有使用转场代理，在转场代理的协议方法中提动画控制器？第二是为什么我们需要自己构建转场上下文？其实这个也是困扰我的问题，其实这两个问题是一体的，如果设置了模态跳转的转场代理，我们在动画控制器里获取的是系统构建的转场上下文，但经过我的实验，很明显系统提供的转场上下文，通过 viewControllerForKey 提取出来的相关控制器不是正确的，也就是说，我们在容器类视图控制器中使用模态跳转的方式来自定义模态跳转的转场动画是不可行的，因为模态跳转构建的转场上下文，fromViewController 一直为容器视图控制器而不是真正的一个子视图控制器。所以，我们要自定义转场上下文，来通过各种协议完成整个转场过程。

Ok，以上就是一个使用动画控制器和自定义转场上下文来实现的自定义容器类视图控制器里子视图控制器的简单切换，下个目标，是实现更为复杂的可交互的容器控制器的转场切换，在此感谢大家的阅读！
