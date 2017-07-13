---
layout:     post
title:      iOS下关于屏幕旋转的一二件小事
category: project
description: iOS下关于屏幕旋转的一二件小事
---


## dd

一般的应用，只会支持竖屏正方向一个方向，支持多个屏幕方向的应用还是比较少的。最近在我的项目中，涉及到视频的旋转，跟这个屏幕方向接触比较多，根据网上的几个测试例子，结合项目做了这个总结。

## 怎么控制屏幕方向
在 iOS 的应用中，有多种方式可以控制界面的屏幕方向，有全局的，有针对 UIWindow 中界面的控制，也有针对单个界面。

### 单个界面控制
单个界面控制主要是指某个ViewController的方向控制。这里要注意，要在ViewController的跟控制器加上方法，一般来说，如果是push上来的，就是NavigationController上面加：

```
- (BOOL)shouldAutorotate
{
    return [self.topViewController shouldAutorotate];
}

- (UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    return [self.topViewController supportedInterfaceOrientations];
}
```

这里要注意几个问题，在设备屏幕旋转时，系统会调用 - shouldAutorotate 方法检查当前界面是否支持旋转，只有 - shouldAutorotate 返回 YES 的时候，- supportedInterfaceOrientations 方法才会被调用，以确定具体方向。

需要注意的是，单个ViewController是不可以控制全局的，就是说状态栏，调整音量都可能是以竖屏作为基准的。公司的项目做横屏的时候，居然是将一个view旋转90度再添加到父控件上面去。这样做没有多大意义。要想真的能控制系统层面的横屏，就要用Window这一层切换。

### Window层的旋转控制
第一，可以直接调用转屏的函数，这个函数据说能够躲避审核风险。

```
//为了防止可能会被拒绝上架,使用这种办法来躲避苹果审核
- (void)interfaceOrientation:(UIInterfaceOrientation)orientation{
    
    if ([[UIDevice currentDevice] respondsToSelector:@selector(setOrientation:)]) {
        SEL selector             = NSSelectorFromString(@"setOrientation:");
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:[UIDevice instanceMethodSignatureForSelector:selector]];
        [invocation setSelector:selector];
        [invocation setTarget:[UIDevice currentDevice]];
        int val                  = orientation;
        // 从2开始是因为0 1 两个参数已经被selector和target占用
        [invocation setArgument:&val atIndex:2];
        [invocation invoke];
    }
}

```
这个函数一调用就会立刻旋转。然后再监听通知，通知中修改一下横屏的布局。

```
    //转动屏幕的通知
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(statusBarOrientationChange:) name:UIApplicationDidChangeStatusBarOrientationNotification object:nil];
```

最后在APPDelagate加上从后台切换到前台的一段代码，用于表示切换到前台请继续保持横屏模式

```
- (UIInterfaceOrientationMask)application:(UIApplication *)application supportedInterfaceOrientationsForWindow:(UIWindow *)window {
    
    UIDeviceOrientation orientation = [UIDevice currentDevice].orientation;
    if (orientation == UIDeviceOrientationLandscapeLeft || orientation == UIDeviceOrientationLandscapeRight) {
        return UIInterfaceOrientationMaskLandscape;
    }else { // 横屏后旋转屏幕变为竖屏
        return UIInterfaceOrientationMaskPortrait;
    }
}
```

这样就完成了视频Controller中的横竖屏切换以及前后台切换的问题了