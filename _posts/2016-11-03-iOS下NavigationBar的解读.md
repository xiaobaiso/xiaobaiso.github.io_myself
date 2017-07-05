---
layout:     post
title:      iOS下NavigationBar解析
category: project
description: iOS下NavigationBar解析
---


## iOS下NavigationBar解析

最近在制作基于毛玻璃的一个滑动渐变NavigationBar,对NavigationBar的层级有了一些了解，故做了一个小的总结

### NavigationBar
这是一个基本的层级：

<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/NavigationBar解析2.png" style="zoom:50%" />

可以看到，UINavigationBar有一个控制自身颜色的UIBarBackground，与它同级的有两个按钮，done，title等按钮，我们关注的就是UIBarBackground

### UIBarBackground
UIImageView其实就是shadowImage，就是NavigationBar下面的那个灰色的线，而UIVisualEffectView这个就是苹果附赠的毛玻璃的控件。要注意EffectFilterView是在最前面的，就是说，这一层将直接决定NavigationBar的颜色

### setBarTintColor和setTintColor
>
 The behavior of tintColor for bars has changed on iOS 7.0. It no longer affects the bar's background
 and behaves as described for the tintColor property added to UIView.
 To tint the bar's background, please use -barTintColor.
 
setTintColor是在iOS7之前控制颜色的方法，之后的话用的是setBarTintColor，他的层级是：

<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/NavigationBar解析4.png" style="zoom:50%" />

注意看，这个层级关系中，UIBarBackground的子控件最上层添加了一个EffectFilterView，这就是调用setBarTintColor导致的结果。也就是说，setBarTintColor是一个最能影响NavigationBar颜色的属性(前提是UIBarBackground层级还在，从下文可知，UIBarBackground可能丢失)


### 乱入的层级关系
一般来说，如果不做太多设置，这个层级关系就是上文所言，但是如果进行了如下设置：

```
[self.navigationController.navigationBar setBackgroundImage:[UIImage new] forBarMetrics:UIBarMetricsDefault];
```
或：

```
self.navigationController.navigationBar.translucent = NO;
```
问题就变得复杂起来，这个时候的层级是这样的：

<img src="https://github.com/xiaobaiso/xiaobaiso.github.io/raw/master/image/NavigationBar解析3.png" style="zoom:50%" />

这个时候，UIBarBackground这一层都消失了，背景颜色直接由backgroundImage来决定。所以如果要保证要有毛玻璃效果，就不要设置背景图片。

### 最后的任务-滑动渐变
滑动渐变颜色主要希望这样，就是先让整个Navigationbar透明下来，随着滑动而逐渐不透明，最后变成一个毛玻璃的效果。

这里思路很简单，就是既然要保证毛玻璃存在，那么就不能设置背景色，这个时候如果要透明的话，就只能控制UIBarBackground的透明度，而UIBarBackground获取并没有公共API，这个时候可以添加NavigationBar的分类，修改其第一个子控件

```
[self.subviews firstObject] .alpha = alpha;
```


