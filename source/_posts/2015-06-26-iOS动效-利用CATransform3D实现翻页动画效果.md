---
title: 利用CATransform3D实现翻页动画效果
date: 2015-06-26 13:53:00
categories:
- 表-iOS界面开发应用
tags:
- iOS
- UI
---

> 从事 iOS 开发已经有一段时间了，之前一直忙于工作，几乎很少有时间写一些东西来对自己掌握的技术进行一下总结，现在想想，有些后悔，因为之前在遇见问题的时候或者学习新技术的时候都是在翻看他人的博客或者查看苹果的官方文档，一直是一个在行业内的“价值消耗者”.
> 对此，我也做过深刻的反思，现在下定决心，以自己微薄的力量来贡献一些东西，也许会对他人有所帮助，希望自己不再是一名“价值消耗者”或是“观望者”，转变为一名“价值贡献者”或是“价值传递者”，在此向那些不断向互联网贡献的人们致以崇高敬意！

### iOS 动效

![图为CityGuideApp,来自UI中国，特此声明](http://upload-images.jianshu.io/upload_images/24274-02cd10da01878d69.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/q/100)

_在 App 爆发式增长的时代，移动 App 的竞争已从单纯的量与技术稳定性的竞争，上升到用户体验与服务等更高层面上的竞争，越来越多的公司开始更加富有“人文关怀”地关注 App 上的用户体验，合理有效的动效是构建良好用户体验不可缺失的一环，恰当好处的动效可以使用户在 App 中得到明确的交互反馈、愉悦的使用感受以及在复杂场景下不易迷失方向等诸多好处，这也是为什么我们要谈动效的原因。_

从技术层面上来说，Apple 在 iOS 平台上提供了供开发者使用的诸多计算机图形技术，这里包括了 UIKit 框架内的 UIView 动画技术、Graphics & Animation 框架内的 2D Drawing、3D Drawing 技术、CoreAnimation 技术等。这些丰富的图形与动画技术框架是 iOS 平台上实现各种华丽动效的基石，第三方也有诸如 facebook 的 Pop 动画引擎，甚至有些 App 会使用 OpenGL、SpriteKit、Metal 等游戏引擎来创作复杂的动画效果，Apple 也在 WWDC 2015 中展示了 OX 10.11 与 iOS9 中，使用 Metal 来重写系统级别的动效，从而提升用户体验，我们有理由相信在未来，越来越多的 App 开发会大量使用合理的动效技术来完善 App 的用户体验。

废话了这么多，下面开始讲点实在的技术东西，这次来实现以下利用 CATransform3D 来做一个类似 FilpBoard 折纸翻页。

效果如下：
雨滴的 iCon 图片是用 sketch 做的，有点丑，大家包涵！

![演示git](https://upload-images.jianshu.io/upload_images/24274-39315d973aeeb536.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/q/100)
![viewDebugging的元素构成](https://upload-images.jianshu.io/upload_images/24274-b263cbd5bb75cfe8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/q/100)

思路梳理

1.  图片本身在 UIView 上，UIView 上布局两个 UIImageView，分为左右两边，每个 UIImageView 进行了圆角 mask 遮罩，UIImageView 的 image 属性为切割的一半 image。

2.  UIView 增加 Pan 手势，手势使得右边的 UIImageView 做 transform 属性动画，transform 属性为 CATransform3D，做沿着 Y 轴的 CATransform3DRotation 仿射变换。

3.  在不断改变 transform 的属性的同时，判断右边 UIImageVIew 是否旋转程度，使得左右两边的图片上显示渐变阴影。

4.  旋转超过一半的时候，将右边的 UIImageVIew 的 image 图片，替换成原来图片加上阴影左右翻转的图片，实现翻转后背景图是模糊的背景。

以上就是简单的思路，这里面比较核心的就是右边的 UIImageVIew 如何做 Y 轴的 rotation 旋转，并且如何实现带有左边小右边大的视察效果。

使用 transform 属性来做仿射变换，我们需要清楚 transform 的本质是什么，iOS 中分为 2D 与 3D 的仿射变换矩阵。

![3D仿射变换矩阵](https://upload-images.jianshu.io/upload_images/24274-13f57a0c18aae04f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/q/100)

在 iOS 中，layer 层的位置变化是通过如图所示的仿射变换矩阵计算来表示的，计算机图形学中，三维坐标系可以描述三维空间的位置，iOS 平台也不例外，在 CoreAnimation 框架中，layer 层的 tranform 属性可以是 CATransform3D 类型的四维仿射变换矩阵，并且提供预置好的进行旋转、变形的仿射变换矩阵，这时候，我们只需要通过获取我们想要的动画效果完成之后需要的 CATransform3D 的 transform，再加上 CAAnimation 的帮助，就可以做出优雅的补间动画。

对于上面的栗子，我们的核心部分就是，右边的 UIImageView 根据 pan 手势而不断改变的 transform 属性，核心代码如下:

```Objective-C
-(CATransform3D)getTransForm3DWithAngle:(CGFloat)angle{

CATransform3D transform =CATransform3DIdentity;//获取一个标准默认的CATransform3D仿射变换矩阵

transform.m34=4.5/-2000;//透视效果

transform=CATransform3DRotate(transform,angle,0,1,0);//获取旋转angle角度后的rotation矩阵。

return transform;

}
```

上面最重要的是 m34 这个属性，CATransform3DRotate 获取的旋转如果之前联合的 transform 不支持透视，那在 x、y 轴上做旋转是只有 frame 放大缩小的变化，我们需要的是在旋转的时候要使得离视角近的地方放大，离视角远的地方缩小，就是所谓的视差来形成 3D 的效果。

```Objective-C
struct CATransform3D

{

CGFloat m11, m12, m13, m14;

CGFloat m21, m22, m23, m24;

CGFloat m31, m32, m33, m34;

CGFloat m41, m42, m43, m44;

};

//这是CATransform3D基本的结构体
```

![矩阵乘法计算](https://upload-images.jianshu.io/upload_images/24274-10e08261df25b74a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/q/100)

我们可以看到 m34 实际上影响了 z 轴方向的 translation，m34= -1/D, 默认值是 0，我们需要尽可能的让 m34 这个值尽可能小，但又必须有明显的远小近大的效果。

以上我们做了根据角度（angle）获得 transform 的 API，那么我们就可以将在 UIView 上的 pan 手势转化为 angle，获取实时的 transform 属性，来做属性动画，在我的 DEMO 里只是单纯的设置新的 transform 属性，如果希望动画更细腻，可以使用 CAAnimation 或者是 CADisplayLink 进行每秒 60 帧的刷新，使得形变更加细腻。

```
CGPoint location = [pan locationInView:self];//获取在View中的location

if(pan.state==UIGestureRecognizerStateBegan) {

    self.initialLocation= location.x;

}//在手势开始的时候，使用属性记录手势在View上的X坐标

if([[self.rightImageView.layervalueForKeyPath:@"transform.rotation.y"]floatValue] > -M_PI_2&&([[self.rightImageView.layervalueForKeyPath:@"transform.rotation.x"]floatValue] !=0)) {

    self.rightBackView.alpha=1;//此时使得模糊翻转的背景图片显示出来
    self.rightShadowLayer.opacity=0;//使得右边的UIImageView的阴影遮罩隐藏
    CGFloat opacity = (location.x-self.initialLocation)/(CGRectGetWidth(self.bounds)-self.initialLocation);//根据初始化距离和此时location的大小除以相对（宽度-初始x坐标）等于一个可正负的手势滑动程度，可直接用作调节阴影的透明度
    self.leftShadowLayer.opacity=fabs(opacity)*0.5;

}
//在右边UIImageView的rotation.y>-M_PI_2(说明在做顺时针旋转)和rotation.x!=0(说明相对于原来位置，x轴做了旋转，此时在右边翻转到左边),至于为何是这样的参数判断，需要了解layer层的坐标系与标准空间坐标系的差别。

else if(([[self.rightImageView.layervalueForKeyPath:@"transform.rotation.y"]floatValue] > -M_PI_2)&&(
    [[self.rightImageView.layervalueForKeyPath:@"transform.rotation.y"]floatValue]<0)&&([[self.rightImageView.layervalueForKeyPath:@"transform.rotation.x"]floatValue] ==0))
{
    self.rightBackView.alpha=0;//此时右边的UIImageView已经旋转但是还没有旋转到左边，使得背景模糊左右颠倒的图片的透明度为0
    CGFloatopacity = (location.x-self.initialLocation)/(CGRectGetWidth(self.bounds)-self.initialLocation);
    //self.rightShadowLayer.opacity = 0 ;
    self.rightShadowLayer.opacity=fabs(opacity)*0.5;
    self.leftShadowLayer.opacity=fabs(opacity)*0.5;
}
if([selfisLocation:locationinView:self]) {
    CGFloat conversioFactor =M_PI/(CGRectGetWidth(self.bounds)-self.initialLocation);
    self.rightImageView.layer.transform= [selfgetTransForm3DWithAngle:(location.x-self.initialLocation)*conversioFactor];//在手势还在view的范围内，通过location获得需要旋转的角度的百分比，然后调用写好的API来获取transform
}
else{
    pan.enabled=NO;
    pan.enabled=YES;
}
```

在完成 pan 手势的 handle 之后，我们需要解决一下，右边 UIImageView 的沿 X=0 的 Y 轴进行旋转的问题。这时候我们需要修改右边 UIImageView 的 Layer 的 Anchor Points。

![anchoriPoint说明](https://upload-images.jianshu.io/upload_images/24274-74e2d73b70a32fee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240/q/100)

```Objective-C
self.rightImageView.layer.anchorPoint=CGPointMake(0,0.5);//需要沿X=0的y轴旋转，需要使得anchorPoint=(0,0.5)
self.rightImageView.frame=CGRectMake(CGRectGetMidX(self.bounds),0,CGRectGetWidth(self.bounds)/2,CGRectGetHeight(self.bounds)); //修改完anchorPoint之后会引起layer的position的变化，所以需要重新修改frame使得layer在正确的位置上。注意修改的先后顺序，frame实际是没有值的，是根据position和anchorPoint算出来的，这一点大家需要周知
```

### 总结

1.  需要正确设置仿射变换的 transform，这里牵扯到旋转变换的坐标轴是怎样设置。

2.  需要注意 pan 手势 handle 中的换算问题，如何将手势滑动程度转换为旋转角度，这个没什么特别需要注意，只能通过经验积累。

3.  需要注意旋转的视差设置，需要了解 3D 仿射变换的矩阵运算，抓住本质问题。

4.  需要调节诸如左右图片的圆角 mask 设置，在 DEMO 源码中，大家可以找到使用 UIBezierPath 做的 mask 遮罩 layer，能够设置圆角的个数和位置，还有阴影的调节，旋转到一定角度后的模糊背景，从而模仿真实的物理场景，关注这些细节才能完善用户体验。

总的来说，要做出这样的交互类型的动效，需要关注 iOS 平台下关于动画的基本知识和运作原理。苹果官方提供强大的文档 Library 供大家学习，在此，我也仅仅是希望大家不要害怕英文文档，阅读官方文档是一种进阶的快速方法。

下次，我将展示一个使用 pop 动画引擎库实现的卡片式设计的动画，这方面一是希望能够和更多的 UE 设计人员沟通，因为 pop 动画简化了开发成本，但对设计要求更高，需要设置出符合动效美学的动画参数，而且配合 facebook 为苹果的 Quartz Composer 功能提供的 origami 插件，pop 动画实现的 userinterface 会更加具有“情怀”。同时，个人觉得，App 的开发，无论对于开发者或是产品或是 UE 等等相关人员来讲，都是一个越加专业化的事情，而这些优秀的工具是用来提高我们的工作效率，但也仅仅是工具，代码也是，而真正有价值的是我们共同对美好（审美）、用户体验、满足合理诉求的一致的追求。

非常感谢大家！希望大家共同进步！

相关资料:
[Apple Developer Library ：Graphics&Animation](https://developer.apple.com/library/ios/navigation/#section=Topics&topic=Graphics%20%26amp%3B%20Animation)
[DEMO](https://github.com/Neojoke/flippingPage)
喜欢的点个赞 👍
