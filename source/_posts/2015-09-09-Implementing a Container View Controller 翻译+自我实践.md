---
title: Implementing a Container View Controller 翻译+自我实践
date: 2015-09-29 16:09:00
categories:
- 表-iOS界面开发应用
tags:
- iOS
- UI
---

大家好，之前做的几个动画 Demo 的文章被许多朋友转载和关注，感谢大家对我鼓励，说实话，以前看别人的文章觉得很容易，自己坚持写文章才真正体会到不容易，加之最近工作中转入 Android 开发，在 iOS 方面的投的精力急剧下降，但好歹，这篇水平不怎地的翻译文章加一个小 Demo 做好了，实际上，我个人觉得，官方文档远远比博客要实在的多（Android 开发者就没那么幸运了），受限于我们的时间和语言上的阻碍，大家往往都会忽略官方文档，导致我们不能理解真正的官方倡导的开发宗旨和意图。所以，各位 iOS 开发者，要多多研读官方文档啊，这是提高水平的最好机会！
本文是 View Controller Programming Guide For iOS 中的一个篇章，WWDC2015 更新了这个 guide，这篇也是最常用的一篇，所以将主要的东西翻译一下，作为记录。

Container View Controller 简称容器类 View Controller，在 UIKit 中，提供诸如 UINavgationController、UITabBarController、UIPageViewController 和 UISplitViewController 等，但有时候我们需要自己定义我们 App 中具有独特交互逻辑的管理其他视图控制器的容器类 View Controller，比方说侧边栏，比方说 ViewPager 似的管理类试图控制器，如何正确的构建容器类的 View Controller，保持被管理的 View Controller 的生命周期调度正确和平衡，是需要精细化设计的事情，而这篇就详细交代了设计这样的容器类 ViewController 的原则以及具体实现起来需要哪些相关的 API，当然，我也会加入实际工作中用到的一些 Tips 与结合 Interface Builder 来讲述一些快速构建容器类 ViewController 的办法。

原文：
[Implementing a Container View Controller](https://developer.apple.com/library/prerelease/ios/featuredarticles/ViewControllerPGforiPhoneOS/ImplementingaContainerViewController.html#//apple_ref/doc/uid/TP40007457-CH11-SW1)

译文：

和其他内容视图控制器一样，容器视图控制器也是管理一个根视图 View 和一些其他内容视图 View 的视图控制器，但是不同的是，容器类视图控制器它的 View 是呈现被限制的嵌套在它的视图层级中的其他视图控制器的 View，容器类视图控制器决定了嵌入到它视图中的其他视图控制器的 View 的位置和大小，但是原有的 View 也可以管理它上面的内容。

## 设计一个自定义的容器类视图控制器

当你设计容器类视图控制器的时候，总是要理解容器（容器视图控制器）与被包含的视图控制器的关系，这种关系能帮助你清楚被管理的视图控制器的内容如何显示到屏幕上以及本质上是如何管理这些内容的，在设计的流程中，需要考虑如下几个问题：

1.  容器视图控制器在呈现子视图控制器的时候扮演什么角色？
2.  有多少子视图控制器能被同时呈现？（个人觉得在子视图控制器切换的时候，需要着重考虑）
3.  如果有的话，子视图控制器之间是什么关系？（跟呈现逻辑相关）
4.  子视图控制器如何增到容器视图控制器中和如何从容器视图控制器中移除？
5.  子视图控制器的 View 的尺寸和位置是否可以被改变？什么条件下会发生这些改变？
6.  容器视图控制器是否提供具体的组件或者类似于导航的东西来管理这些子视图么？
7.  子视图控制器和容器视图控制器的通讯方式是什么？以及容器视图控制器是否需要传送一些不是 UIViewController 提供的标准时间的特殊事件给子视图控制器？
8.  是否有不同的方式来呈现容器类视图控制器，如果有，怎么呈现？

当你思考完这些问题并且制定出相应的规则之后，实现一个容器类视图控制器将是一件非常简单的事情。UIKit 中唯一需要的条件是构建容器视图控制器和你要管理的视图控制器之间的父子关系。这种父子关系能够保证子视图控制器能够接收到系统发送的消息。（子视图控制器的生命周期函数的正常调用等）除此之外，每一种容器类试图控制器不同的地方就是在布局和管理期间对它包含的 views 的做的操作。你可以增加一些自定义的 Views 到一些定制的组件和导航的视图层级中去。

### 例子：导航控制器

一个 UINavigationController 对象支持通过内容的层级结构来进行导航。导航控制器在一个时间内只能显示一个子视图的界面，上部的导航栏显示当前在子视图控制器当中的哪一个，左边按钮是返回上一个层级，导航到下一层级新的子视图控制器会往左滑入并且可以包含一些按钮或者是可用的表格。
导航在子视图控制器之间是共享的，当一个视图的界面上有一些按钮或者是表格，关联上子视图控制器，子视图控制器可以要求导航控制器 push 到一个新的视图控制器到容器 View 上，子视图控制器更新内容，但是导航控制器管理转场的动画。导航控制器也管理着可以使当前视图控制器消失的导航栏。
图 5-1 显示了一个导航控制器和它上面视图的结构，大部分区域是被子视图控制器给填充的，只有一小部分区域是导航栏。

![](http://upload-images.jianshu.io/upload_images/24274-eacc4654641d8229.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)图 5-1Structure of a navigation interface

无论是宽松还是紧凑的环境下，一个导航控制器仅仅能显示一个子视图控制器。导航控制器可以重新布局子视图控制器以重新适应可用空间。

### 例子：分栏控制器

一个 UISplitViewController 对象可以分为主副区域来显示两个子视图控制器。在这种安排下，一个视图控制器的内容（主）决定另外一个副区域显示哪个子视图控制器。两个视图控制器的显示是可以配置的，同时也取决于两个子视图控制器的的当前环境。在宽松的水平环境下，分栏控制器可以显示两个子视图控制器，在紧凑的模式下，分栏控制器只能在同一时间显示一个试图控制器。
图 5-2 展示了一个分栏控制器在宽松的水平环境下的结构，分栏控制器仅仅包含自己默认的容器 view，在这个例子中，两个子视图控制器显示在两边，子视图控制器的大小是可配置的，从而可以显示主控制器。

![](http://upload-images.jianshu.io/upload_images/24274-648d68beeecd6dfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)图 5-2A split view interface

## 在 interface builder 中配置一个容器类视图控制器

在设计阶段创建一个父子视图容器类控制器，需要在 storyboard 中增加一个 container view 对象，如图 5-3。container view 对象是一个占位符，用来显示子视图控制器的内容。使用视图的大小和位置确定子视图控制器的根视图在容器类视图控制器中的位置和大小。

![](http://upload-images.jianshu.io/upload_images/24274-9de8e765376045a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)5-3Adding a container view in Interface Builder

当你使用一个或者多个 container view 来加载 view controller 的时候，interface builder 也会将子视图控制器的 view 与 container view 连接并加载。子视图控制器在此期间必须完成初始化并且父子关系也会被创建。
如果你不使用 interface builder 创建父子关系，你必须通过代码的形式构建，详情见后面的“增加一个子视图控制器到你的容器里”

## 实现一个自定义的容器类试图控制器

为了实现容易类视图控制器，你必须建立容器类视图控制器与子视图控制器的父子关系。在你即将要管理这些子视图控制器的 view 的之前必须构建好这些父子关系。这样可以让 UIKit 知道你正在管理的子视图控制器的视图的位置和大小，你可以通过 interface builder 或者代码进行构建这些关系。当你使用代码创建这些父子关系的时候，你可以准确的增加或者移除子视图控制器。

## 增加一个子视图控制器到你的容器里

为了能够通过代码将子视图控制器添加到容器类视图控制器中，如下步骤可以创建父子关系：

1.  调用容器视图控制器 addChildViewController: 的方法，这个方法用来告诉 UIKit 你的容器类视图控制器即将要管理一个子视图控制器。
2.  增加子视图控制器的 view 到 容器视图控制的 view（或者 Container View）的视图层级结构中，千万别忘记设置大小的位置。
3.  为管理子视图控制的 view 的大小和位置添加约束条件。
4.  调用子视图控制的 didMoveToParentViewController: 方法。

表 5-1 展示了一个容器类视图控制器怎样嵌入子视图控制器，当创建完父子关系之后，在将子视图加入到容器视图中，设置了子视图的 frame 、size。设置 frame 与 size 是非常重要的也是保证正确的将子视图显示在容器中，在增加完之后，容器类视图控制器调用 didMoveToParentViewController:方法，这是因为要给子视图控制器能够响应变化的机会。（调用相应的生命周期函数布局根视图）

```Objective-C
- (void) displayContentController: (UIViewController*) content {
   [self addChildViewController:content];
   content.view.frame = [self frameForContentController];
   [self.view addSubview:self.currentClientView];
   [content didMoveToParentViewController:self];
}
```

**表 5-1** 增加一个子视图控制器到容器中

在上一个例子中，你会发现你仅仅调用子视图控制器的 didMoveToParentViewController:，这是因为容器类视图控制器调用 addChildViewController:会引发自动调用子视图控制器的 willMoveToParentViewController:，你必须自己调用容器类视图控制器的 didMoveToParentViewController:方法的原因是你必须在完成将子视图控制器的 view 嵌入到容器类视图控制器的视图层级之后才能调用该方法。
当你使用 AutoLayout 的时候，请在将配置约束条件的工作放在子视图控制的 view 已经嵌入到容器类视图控制器的层级结构中。你的约束条件必须仅仅能影响子视图控制器的根视图的位置和大小，不要修改根视图的内容或者是其他子视图控制器视图结构中的视图。

## 移除一个子视图控制器

为了移除一个子视图控制器，移除父子关系必须遵循以下步骤：

1.  调用子视图的 willMoveToParentViewController: 方法 参数为 nil
2.  移除任何你对子视图控制的 view 的约束条件
3.  将子视图控制的 view 从容器类视图控制器的层级结构中移除
4.  调用子视图控制器的 removeFromParentViewController 方法 从而最终结束父子关系

永久性的断绝子视图控制器与容器视图控制器的父子关系，移除一个子视图控制器必须保证你将不再需要引用这个子视图控制器。举个例子，一个导航控制器在 push 进来一个新的子视图控制器到导航栈中并不需要移除当前的子视图控制器，仅仅在该子视图控制器从导航栈中弹出才进行移除子视图控制器。
表 5-2 展示了一个子视图控制器如何从容器中移除.调用 willMoveToParentViewController:的方法，参数为 nil，给子视图控制器一个应对变化的一个机会。removeFromParentViewController 也会调用子视图控制器的 didMoveToParentViewController: 的方法，传递 nil 的参数。最终解除与容器的父子关系。

```Objective-C
- (void) hideContentController: (UIViewController*) content {
   [content willMoveToParentViewController:nil];
   [content.view removeFromSuperview];
   [content removeFromParentViewController];
}
```

**表 5-2** 从容器中移除子视图控制器

## 子视图控制器之间的转场

当你想要使用包含移入移出动画来切换子视图控制器的时候，在这些动画之前，你需要保证两个子视图控制器都是容器视图控制器的内容的一部分，并且你要使当前子视图控制器知道它即将要开始动画。在动画期间，要使得新的子视图控制器进入到预定位置，并且要移出旧的子视图控制器的 view。在完成动画的时候，需要彻底完成移出子视图控制器的工作。

表 5-3 展示了一个使用转场动画完成一个子视图控制器切换到另外一个子视图控制器的例子。在这个例子中，新的视图控制器开启动画移动到当前正在进行移出屏幕的视图控制器的矩形位置上。当动画完成，回调的 block 会从容器视图控制器移出子视图控制器，在这个例子中，transitionFromViewController:toViewController:duration:options:animations:completion: 这个方法自动的更新容器视图控制器中的 view 层级结构，所以你不需要增加或者删除 views。

```- (void)cycleFromViewController: (UIViewController*) oldVC
               toViewController: (UIViewController*) newVC {
   // Prepare the two view controllers for the change.
   [oldVC willMoveToParentViewController:nil];
   [self addChildViewController:newVC];
   // Get the start frame of the new view controller and the end frame
   // for the old view controller. Both rectangles are offscreen.
   newVC.view.frame = [self newViewStartFrame];
   CGRect endFrame = [self oldViewEndFrame];
   // Queue up the transition animation.
   [self transitionFromViewController: oldVC toViewController: newVC
        duration: 0.25 options:0
        animations:^{
            // Animate the views to their final positions.
            newVC.view.frame = oldVC.view.frame;
            oldVC.view.frame = endFrame;
        }
        completion:^(BOOL finished) {
           // Remove the old view controller and send the final
           // notification to the new view controller.
           [oldVC removeFromParentViewController];
           [newVC didMoveToParentViewController:self];
        }];
}
```

## 管理子视图控制器的外观更新

在将子视图控制器增加到容器视图控制器之后，容器类视图控制器自动的将外观相关的消息传送给子视图。这是正常的情况，因为保证了所有的事件都被发送。然而，有些时候，默认的行为在一条指令中发送一些事件，然而并没有使容器察觉。例如，如果多个子视图控制器同时更新它们 view 的状态，你可能会想合并这些变化，因此外观的回调全部发生在同一时间。
为了接管处理这些外观变化的回调，需要在你的容器类视图控制器中重写 shouldAutomaticallyForwardAppearanceMethods 的方法，返回值为 NO，在表 5-4 中的例子，返回 No 使得 UIKit 知道你的容器类视图控制器会通知它的子视图控制器在其界面中的变化。

```Objective-C
- (BOOL) shouldAutomaticallyForwardAppearanceMethods {
    return NO;
}
```

**5-4 表** automatic appearance forwarding

当显示的转场发生的时候，调用子视图的 beginAppearanceTransition:animated:与 endAppearanceTransition 来显现。例如，你的容器有一个单一的子视图控制器，你可以在不同的生命周期函数中调用，确保子视图控制器获取到这些消息。如表 5-5。

```Objective-C
-(void) viewWillAppear:(BOOL)animated {
    [self.child beginAppearanceTransition: YES animated: animated];
}
-(void) viewDidAppear:(BOOL)animated {
    [self.child endAppearanceTransition];
}
-(void) viewWillDisappear:(BOOL)animated {
    [self.child beginAppearanceTransition: NO animated: animated];
}
-(void) viewDidDisappear:(BOOL)animated {
    [self.child endAppearanceTransition];
}
```

**表** 5-5

## 创建一个容器类视图控制器的建议

设计，开发和测试一个新的容器类视图控制器是需要花费时间的，即使独立的行为是简单易懂的，但将视图控制器作为整体仍然是复杂的，当实现一个容器类控制器的类的时候，请考虑以下 tips:
**仅仅有获取子视图控制器根视图的权限.** 容器视图控制器仅仅只需要有每一个子视图控制器根视图的访问权限比如返回子视图的 view 的属性。从不应该获取其他 views 的权限

**子视图控制器应该最小限度的了解它的容器（也就是说子视图控制器不需要做很多的配置才能够被添加到容器中，子视图控制器与容器控制器应该是松耦合的） **.子视图控制器应该把精力放在自己内容的管理上. 如果一个容器类视图控制器想要影响子视图控制器上的内容，不要直接操作，使用代理设计模式去管理这些影响
**Design your container using regular views first.** Using regular views (instead of the views from child view controllers) gives you an opportunity to test layout constraints and animated transitions in a simplified environment. When the regular views work as expected, swap them out for the views of your child view controllers.（这段翻译不知道怎么翻，还是看原文比较舒服）

##委托控制子视图控制器
一个容器试图控制器可以代理自身的 appearance 和一个或者多个子视图控制器，可以采用如下方式代理：

> **让子视图控制器决定状态栏风格**. 将子视图控制器作为修改状态栏的代理, 重写 childViewControllerForStatusBarStyle 和 childViewControllerForStatusBarHidden 在容器视图控制器中的两个方法
> 让子视图控制器提供期许的尺寸大小，容器类控制器有灵活的适应性的布局，能够使用子视图控制器的 preferredContentSize 来帮助定位其尺寸

### 例子（Demo 持续更新）

以下 demo 是做的一个小小的例子，时间仓促，没有过多讲解，后期会不断丰富这个例子，从而实现复杂的容器类视图控制器，并讲解如何自定义容器视图控制器的转场动画（不可交互与可交互）
demo 是使用 XCode7 beat6 开发的，swift2.0，这点大家周知
[2015.9.9 ContainerViewController](https://github.com/Neojoke/ContainerViewController)
