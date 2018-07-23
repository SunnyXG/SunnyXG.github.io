---
layout: post
title: iOS轮播图和PageViewController滑动手势冲突
date: 2018-07-23 19:51 
---

需求场景:

> 在一个UIPageViewController中， 某一个pageview中含有一个轮播图scrollview，需要实现轮播图可以在UIPageViewController中自由滑动点击

遇到的一些问题:
>1.当拖动轮播图时，触发UIPageViewController的翻页动作。
>2.当轮播图滑动时，禁用pageViewController的scrollEnable，当pageviewController在scrollEnable属性被禁用时被触摸，如何区分用的行为是点击和拖动。
>3.使用设置pageviewController.dataSource = nil，用户想要翻页时第一次手势操作无效，需要手动翻页，不能准确判断用户手势的类型。

解决方法：
> 主要思路: 利用UIGestureRecognizer的Delegate，监测接收的手势是什么行为触发的。

主要做法：
1. UIScrollView中， 实现可以多手势识别的代理, 当用户在进行swipe时，scrollview和其他视图同时接收到手势行为，scrollView即使不是第一响应者依旧会执行手势事件。

```code
// 定义手势并指定代理
UISwipeGestureRecognizer *swipeLeftGesture = [[UISwipeGestureRecognizer alloc] initWithTarget:self action:@selector(swipeGestureTrigger:)];
        swipeLeftGesture.direction = UISwipeGestureRecognizerDirectionLeft;
        swipeLeftGesture.delegate = self;
        [self.scrollView addGestureRecognizer:swipeLeftGesture];
        UISwipeGestureRecognizer *swipeRightGesture = [[UISwipeGestureRecognizer alloc] initWithTarget:self action:@selector(swipeGestureTrigger:)];
        swipeRightGesture.direction = UISwipeGestureRecognizerDirectionRight;
        swipeRightGesture.delegate = self;
        [self.scrollView addGestureRecognizer:swipeRightGesture];


#pargma mark - UIGestureRcognizerDelegate
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
    return YES;
}
```

2. 在UIPageViewController中，对pageview子视图中的scrollview添加swipe手势，用于pageViewController的手势行为监控

```
// 新建一个swipe手势添加到pageview的子视图Scrollview上，方向设为左
UISwipeGestureRecognizer *swipe = [UISwipeGestureRecognizer new];
swipe.delegate = self;
swipe.direction = UISwipeGestureRecognizerDirectionLeft;
[pageScrollView addGestureRecognizer:swipe];

#pragma mark - UIGestureRecognizerDelegate

- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer
{
// 当pageViewController翻页时，恢复其scrollEnable为YES，因为pageview不支持多手势识别，所以检测到用户为swipe手势时执行翻页操作。
    if (!self.pageScrollView.scrollEnabled && self.titleTableView.selectedItemIndex == 0) {
        self.pageScrollView.scrollEnabled = YES;
        self.titleTableView.selectedItemIndex = 1;
    }

    return YES;
}

- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
// 用于禁用pageview的scrollEnable从NO改变为YES后，区分第一次用户操作是swipe翻页还是tap点击。设为NO只会识别一种手势行为。
    return NO;
}

- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch
{
// 判断touch点击的位置是否为轮播图（判断条件不唯一），如果是将pageView的scrollview的scrollEnabled设为NO，这样pageview将不再执行翻页操作，但是轮播图可以识别多手势，轮播图依旧滑动。
    if ([touch.view isKindOfClass:[UIImageView class]] && touch.view.tag > 0) {
        self.pageScrollView.scrollEnabled = NO;

        return NO;
    }

    return YES;
}

```

