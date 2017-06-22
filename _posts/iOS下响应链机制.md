---
layout:     post
title:      IOS下的图片模糊化模糊化
category: project
description: IOS下的图片模糊化模糊化
---


关于iOS的响应链
测试妹子给我挂了一个Bug，原因是那个隐秘按钮热点区域太小，点击不够顺畅。我仔细定位了一下，发现确实是妹子的手指太大，不够秀巧，刚想驳回，转眼一想有一个增加热点的办法，故进行了尝试。

其实，这是个老生常谈的话题，iOS下的事件传递机制，事件响应是根据响应者链条进行传递的，（https://developer.apple.com/documentation/uikit/understanding_event_handling_responders_and_the_responder_chain）根据官方文档描述，说干就干。

这个隐藏密码的眼睛，即便调大了也并不能增加多少热点区域，所以我打算把这个按钮往右边偏移一部分，成为这样：
但是这样多出来的并不能点击，这还要涉及响应者链条问题。

首先，要知道的是，要让按钮可以相应，就必须让按钮成为hit-test-view,就是说，只有hit-test-view才能相应事件。根据文档，当你点击了屏幕上的某个view，这个动作由硬件层传导到操作系统，然后又从底层封装成一个事件（Event）顺着view的层级往上传导，一直要找到含有这个点击点且层级最高（文档说是最低，我理解是逻辑上最靠近手指）的view来响应事件，这个view就是hit-test-view。

决定谁hit-test view是通过不断递归调用view中的 - (UIView *)hitTest: withEvent: 方法和 -(BOOL)pointInside: withEvent: 方法来实现的。- (UIView *)hitTest: withEvent:方法是用于传递事件，而 -(BOOL)pointInside: withEvent:用于判断触摸点是否在自身范围内。

对于传递的顺序以及调用过程，可以参考demo https://github.com/slemon/HitTestViewUsage ，找到hit-test view后，它会有最高的优先权去响应逐级传递上来的Event，如它不能响应就会传递给它的superview，依此类推，一直传递到UIApplication都无响应者，这个Event就会被系统丢弃了。

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
所以我在按钮的父类加上方法判断是否在当前点击，这样就能找到对应的hit-test-view了，但是还有一个问题，那就是层级问题，hit-test-view是通过这个Controller的view的subviews按照从后往前的顺序来寻找hit-test-view的，所以，[self bringSubviewToFront:self.passwordInputField];这样就是将层级提高到最前面，就是离手指最近，也就是subviews的最后面。

这样隐藏密码的按钮就可以响应事件了。现在来看层级结构图，我们之前认为最上面的就是可以点击的，事实上最上面的就是subvuews的最后一个啊，而hit-test就是从后往前找，这样来看，是不是就清楚多了。



[工程代码](https://github.com/xiaobaiso/BlurryImage)