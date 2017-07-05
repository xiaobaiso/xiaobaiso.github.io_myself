---
layout:     post
title:      iOS下NavigationBar解析
category: project
description: iOS下NavigationBar解析
---


##iOS下NavigationBar解析

最近在制作基于毛玻璃的一个滑动渐变NavigationBar,对NavigationBar的层级有了一些了解，故做了一个小的总结

###NavigationBar
这是一个基本的层级：


可以看到，UINavigationBar有一个控制自身颜色的UIBarBackground，与它同级的有两个按钮，done，title等按钮，我们关注的就是UIBarBackground

###UIBarBackground
UIImageView其实就是shadowImage，就是NavigationBar下面的那个灰色的线，而UIVisualEffectView这个就是苹果附赠的毛玻璃的控件。要注意EffectFilterView是在最前面的，就是说，这一层将直接决定NavigationBar的颜色

###setBarTintColor和setTintColor
>
 The behavior of tintColor for bars has changed on iOS 7.0. It no longer affects the bar's background
 and behaves as described for the tintColor property added to UIView.
 To tint the bar's background, please use -barTintColor.
 
setTintColor是在iOS7之前控制颜色的方法，之后的话用的是setBarTintColor，他的层级是：

注意看，这个层级关系中，UIBarBackground的子控件最上层添加了一个EffectFilterView，这就是调用setBarTintColor导致的结果。


###乱入的层级关系
一般来说，如果不做太多设置，这个层级关系就是上文所言，但是如果进行了如下设置：

```
[self.navigationController.navigationBar setBackgroundImage:[UIImage new] forBarMetrics:UIBarMetricsDefault];
```
或：

```
self.navigationController.navigationBar.translucent = NO;
```
问题就变得复杂起来，这个时候的层级是这样的：

这个时候，UIBarBackground这一层都消失了，背景颜色直接由backgroundImage来决定。所以如果要保证要有毛玻璃效果，就不要设置背景图片。

###滑动渐变
滑动渐变颜色主要希望这样，就是先让整个Navigationbar透明下来，随着滑动而逐渐不透明，最后变成一个毛玻璃的效果。

这里思路很简单，就是既然要保证毛玻璃存在，那么就不能设置背景色，这个时候如果要透明的话，就只能控制UIBarBackground的透明度，而UIBarBackground获取并没有公共API，这个时候可以添加NavigationBar的分类，修改其第一个子控件

```
[self.subviews firstObject] .alpha = alpha;
```


<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/关于iOS的响应链4.png" style="zoom:50%" />

说干就干。

这个隐藏密码的眼睛，即便调大了也并不能增加多少热点区域，所以我打算把这个按钮往右边偏移一部分，成为这样：

<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/关于iOS的响应链1.png" style="zoom:50%" />
.

<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/关于iOS的响应链2.png" style="zoom:50%" />
.

但是这样多出来的并不能点击，这还要涉及响应者链条问题。

首先，要知道的是，要让按钮可以相应，就必须让按钮成为hit-test-view,就是说，只有hit-test-view才能相应事件。根据文档，当你点击了屏幕上的某个view，这个动作由硬件层传导到操作系统，然后又从底层封装成一个事件（Event）顺着view的层级往上传导，一直要找到含有这个点击点且层级最高（文档说是最低，我理解是逻辑上最靠近手指）的view来响应事件，这个view就是hit-test-view。

决定谁hit-test view是通过不断递归调用view中的 - (UIView *)hitTest: withEvent: 方法和 -(BOOL)pointInside: withEvent: 方法来实现的。- (UIView *)hitTest: withEvent:方法是用于传递事件，而 -(BOOL)pointInside: withEvent:用于判断触摸点是否在自身范围内。

对于传递的顺序以及调用过程，可以参考[demo](https://github.com/slemon/HitTestViewUsage) ，找到hit-test view后，它会有最高的优先权去响应逐级传递上来的Event，如它不能响应就会传递给它的superview，依此类推，一直传递到UIApplication都无响应者，这个Event就会被系统丢弃了。

```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    
    CGPoint currPoint = [self convertPoint:point toView:self.showPwdBtn];
    
    if ([self.showPwdBtn pointInside:currPoint withEvent:event]) {
        if (![self.showPwdBtn isHidden]) {
            return self.showPwdBtn;
        }
    }
   CGPoint currPoint2 = [self convertPoint:point toView:self.clearBtn];
    
    if ([self.clearBtn pointInside:currPoint2 withEvent:event]) {
        if (![self.clearBtn isHidden]) {
            return self.clearBtn;
        }
    }
    return [super hitTest:point withEvent:event];
    
}
```
所以我在按钮的父类加上方法判断是否在当前点击，这样就能找到对应的hit-test-view了，但是还有一个问题，那就是层级问题，hit-test-view是通过这个Controller的view的subviews按照从后往前的顺序来寻找hit-test-view的，所以，[self bringSubviewToFront:self.passwordInputField];这样就是将层级提高到最前面，就是离手指最近，也就是subviews的最后面。

<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/关于iOS的响应链3.png" style="zoom:50%" />
.

这样隐藏密码的按钮就可以响应事件了。现在来看层级结构图，我们之前认为最上面的就是可以点击的，事实上最上面的就是subvuews的最后一个啊，而hit-test就是从后往前找，这样来看，是不是就清楚多了。
