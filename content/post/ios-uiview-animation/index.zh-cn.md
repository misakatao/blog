---
title: "UIView动画事务"
description: "UIView 动画与事务的使用详解"
date: 2016-10-23T14:24:07+08:00
slug: ios-uiview-animation
image: "cover.svg"
categories:
    - iOS
tags:
    - UI
    - iOS开发
    - 动画
---

UIView 动画事务是 iOS 开发中最常用的动画方式之一，通过简洁的接口即可实现位移、缩放、旋转、颜色渐变、透明度变化等多种动画效果。本文介绍 UIView 动画事务的基本要素、配置方式、Block 实现、UIImageView 动画以及动画过程中的事件处理。

<!--more-->

## 基本要素

UIView 动画事务支持对以下属性进行隐式动画：

- `frame` / `bounds`：视图大小及位置
- `center`：视图中心点位置
- `transform`：仿射变换（缩放、旋转）
- `alpha`：透明度

动画的核心配置包括：

- 持续时间（Duration）：动画执行的总时长
- 线性规律（Curve）：动画的缓动方式
- 动画类型：通过改变视图属性产生动画效果
- 回调方法：监视动画执行状态，在完成时执行后续操作

## 初始化与配置

### 一般方法

通过 `beginAnimations:context:` 和 `commitAnimations` 包裹动画配置：

```objective-c
[UIView beginAnimations:@"alpha" context:nil];

// 在此配置持续时间、线性规律、动画类型等
[UIView setAnimationDuration:1.0];
[UIView setAnimationCurve:UIViewAnimationCurveLinear];

// 修改视图属性触发动画
_animationsView.alpha = 0.5;

[UIView commitAnimations];
```

### Block 方法

使用 Block 语法更加简洁，也是目前推荐的方式：

```objective-c
[UIView animateWithDuration:1.0 animations:^{
    _animationsView.alpha = 0.5;
}];
```

带完成回调和配置项的完整形式：

```objective-c
[UIView animateWithDuration:1.0
                      delay:0.0
                    options:UIViewAnimationOptionCurveLinear
                 animations:^{
    _animationsView.center = self.view.center;
} completion:^(BOOL finished) {
    // 动画完成后的操作
}];
```

## 转场动画与委托回调

### 转场动画类型

通过 `setAnimationTransition:forView:cache:` 设置转场效果：

- `UIViewAnimationTransitionNone`：无转场效果
- `UIViewAnimationTransitionFlipFromLeft`：从左向右翻转
- `UIViewAnimationTransitionFlipFromRight`：从右向左翻转
- `UIViewAnimationTransitionCurlUp`：向上翻页
- `UIViewAnimationTransitionCurlDown`：向下翻页

### 委托回调

一般方法通过设置委托和选择器来监听动画状态：

```objective-c
[UIView setAnimationDelegate:self];
[UIView setAnimationWillStartSelector:@selector(animationWillStart:context:)];
[UIView setAnimationDidStopSelector:@selector(animationDidStop:finished:context:)];
```

回调方法签名：

```objective-c
- (void)animationDidStop:(NSString *)animationID
                finished:(NSNumber *)finished
                 context:(void *)context;
```

Block 方法则直接在 `completion` 中处理回调，无需声明额外方法。实现组合动画时，Block 方式可以在 `completion` 中直接触发下一个动画，比委托方式更直观。

## 完整示例：一般方法实现组合动画

以下示例通过委托回调串联多个动画，依次执行位移、缩放、旋转、变色、透明度变化：

```objective-c
#import "UIViewAnimationsViewController.h"

static NSTimeInterval const animationDuration = 1.0;

static NSString *const kAnimationPosition = @"position";
static NSString *const kAnimationScale    = @"scale";
static NSString *const kAnimationRotate   = @"rotate";
static NSString *const kAnimationColor    = @"color";
static NSString *const kAnimationAlpha    = @"alpha";
static NSString *const kAnimationEnd      = @"end";

@interface UIViewAnimationsViewController ()
@property (nonatomic, strong) UIView *animationsView;
@end

@implementation UIViewAnimationsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"ViewAnimations";
    self.view.backgroundColor = [UIColor whiteColor];
    [self.view addSubview:self.animationsView];

    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleTap:)];
    [self.animationsView addGestureRecognizer:tap];
}

- (void)handleTap:(UITapGestureRecognizer *)gesture {
    [self positionAnimation];
}

- (void)positionAnimation {
    [UIView beginAnimations:kAnimationPosition context:nil];
    [UIView setAnimationDuration:animationDuration];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    [UIView setAnimationDelegate:self];
    [UIView setAnimationDidStopSelector:@selector(animationDidStop:finished:context:)];
    _animationsView.center = self.view.center;
    [UIView commitAnimations];
}

- (void)scaleAnimation {
    [UIView beginAnimations:kAnimationScale context:nil];
    [UIView setAnimationDuration:animationDuration];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    [UIView setAnimationDelegate:self];
    [UIView setAnimationDidStopSelector:@selector(animationDidStop:finished:context:)];
    _animationsView.transform = CGAffineTransformMakeScale(1.5, 1.5);
    [UIView commitAnimations];
}

- (void)rotateAnimation {
    [UIView beginAnimations:kAnimationRotate context:nil];
    [UIView setAnimationDuration:animationDuration];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    [UIView setAnimationDelegate:self];
    [UIView setAnimationDidStopSelector:@selector(animationDidStop:finished:context:)];
    _animationsView.transform = CGAffineTransformRotate(_animationsView.transform, M_PI_2);
    [UIView commitAnimations];
}

- (void)colorAnimation {
    [UIView beginAnimations:kAnimationColor context:nil];
    [UIView setAnimationDuration:animationDuration];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    [UIView setAnimationDelegate:self];
    [UIView setAnimationDidStopSelector:@selector(animationDidStop:finished:context:)];
    _animationsView.backgroundColor = [UIColor redColor];
    [UIView commitAnimations];
}

- (void)alphaAnimation {
    [UIView beginAnimations:kAnimationAlpha context:nil];
    [UIView setAnimationDuration:animationDuration];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    [UIView setAnimationDelegate:self];
    [UIView setAnimationDidStopSelector:@selector(animationDidStop:finished:context:)];
    _animationsView.alpha = 0.5;
    [UIView commitAnimations];
}

- (void)endAnimations {
    [UIView beginAnimations:kAnimationEnd context:nil];
    [UIView setAnimationDuration:animationDuration];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    _animationsView.backgroundColor = [UIColor blackColor];
    _animationsView.alpha = 1;
    _animationsView.center = CGPointMake(CGRectGetMidX(_animationsView.bounds),
                                         CGRectGetMidY(_animationsView.bounds) + 64);
    _animationsView.transform = CGAffineTransformIdentity;
    [UIView setAnimationTransition:UIViewAnimationTransitionCurlUp forView:_animationsView cache:NO];
    [UIView commitAnimations];
}

- (void)animationDidStop:(NSString *)animationID finished:(NSNumber *)finished context:(void *)context {
    if ([animationID isEqualToString:kAnimationPosition]) {
        [self scaleAnimation];
    } else if ([animationID isEqualToString:kAnimationScale]) {
        [self rotateAnimation];
    } else if ([animationID isEqualToString:kAnimationRotate]) {
        [self colorAnimation];
    } else if ([animationID isEqualToString:kAnimationColor]) {
        [self alphaAnimation];
    } else if ([animationID isEqualToString:kAnimationAlpha]) {
        [self endAnimations];
    }
}

- (UIView *)animationsView {
    if (!_animationsView) {
        _animationsView = [[UIView alloc] init];
        _animationsView.bounds = CGRectMake(0, 0, 160, 160);
        _animationsView.center = CGPointMake(CGRectGetMidX(_animationsView.bounds),
                                              CGRectGetMidY(_animationsView.bounds) + 64);
        _animationsView.backgroundColor = [UIColor blackColor];
    }
    return _animationsView;
}

@end
```

## 完整示例：Block 方法实现组合动画

同样的组合动画效果，使用 Block 方式实现更加简洁：

```objective-c
#import "UIViewAnimationsViewController.h"

static NSTimeInterval const animationDuration = 1.0;

@interface UIViewAnimationsViewController ()
@property (nonatomic, strong) UIView *animationsView;
@end

@implementation UIViewAnimationsViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"ViewAnimations";
    self.view.backgroundColor = [UIColor whiteColor];
    [self.view addSubview:self.animationsView];

    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleTap:)];
    [self.animationsView addGestureRecognizer:tap];
}

- (void)handleTap:(UITapGestureRecognizer *)gesture {
    [self positionAnimationBlock];
}

- (void)positionAnimationBlock {
    [UIView animateWithDuration:animationDuration delay:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        self.animationsView.center = self.view.center;
    } completion:^(BOOL finished) {
        [self scaleAnimationBlock];
    }];
}

- (void)scaleAnimationBlock {
    [UIView animateWithDuration:animationDuration delay:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        self.animationsView.transform = CGAffineTransformMakeScale(1.5, 1.5);
    } completion:^(BOOL finished) {
        [self rotateAnimationBlock];
    }];
}

- (void)rotateAnimationBlock {
    [UIView animateWithDuration:animationDuration delay:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        self.animationsView.transform = CGAffineTransformRotate(self.animationsView.transform, M_PI);
    } completion:^(BOOL finished) {
        [self colorAnimationBlock];
    }];
}

- (void)colorAnimationBlock {
    [UIView animateWithDuration:animationDuration delay:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        self.animationsView.backgroundColor = [UIColor redColor];
    } completion:^(BOOL finished) {
        [self alphaAnimationBlock];
    }];
}

- (void)alphaAnimationBlock {
    [UIView animateWithDuration:animationDuration delay:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        self.animationsView.alpha = 0.5;
    } completion:^(BOOL finished) {
        [self endAnimationsBlock];
    }];
}

- (void)endAnimationsBlock {
    [UIView animateWithDuration:animationDuration delay:0.0 options:UIViewAnimationOptionCurveLinear animations:^{
        self.animationsView.center = CGPointMake(CGRectGetMidX(self.animationsView.bounds),
                                                  CGRectGetMidY(self.animationsView.bounds) + 64);
        self.animationsView.backgroundColor = [UIColor blackColor];
        self.animationsView.alpha = 1;
        self.animationsView.transform = CGAffineTransformIdentity;
    } completion:nil];
}

- (UIView *)animationsView {
    if (!_animationsView) {
        _animationsView = [[UIView alloc] init];
        _animationsView.bounds = CGRectMake(0, 0, 160, 160);
        _animationsView.center = CGPointMake(CGRectGetMidX(_animationsView.bounds),
                                              CGRectGetMidY(_animationsView.bounds) + 64);
        _animationsView.backgroundColor = [UIColor blackColor];
    }
    return _animationsView;
}

@end
```

## UIImageView 动画

UIImageView 除了展示静态图片，还可以通过逐帧播放实现动画效果。

相关属性：

- `animationImages`：动画图片数组（`NSArray<UIImage *> *`）
- `animationDuration`：动画持续时长
- `animationRepeatCount`：重复次数，默认 0 表示无限循环

```objective-c
- (void)setupImageAnimation {
    _imageAnimationsView.animationDuration = 5.0;
    _imageAnimationsView.animationImages = _images;
    _imageAnimationsView.animationRepeatCount = 0;
    [_imageAnimationsView startAnimating];
}
```

通过触摸事件可以实现拖动 UIImageView 动画视图：

```objective-c
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    if (touch.view == _imageAnimationsView) {
        CGPoint previousLocation = [touch previousLocationInView:_imageAnimationsView];
        CGPoint currentLocation = [touch locationInView:_imageAnimationsView];
        CGPoint offset = CGPointMake(currentLocation.x - previousLocation.x,
                                     currentLocation.y - previousLocation.y);
        _imageAnimationsView.center = CGPointMake(_imageAnimationsView.center.x + offset.x,
                                                   _imageAnimationsView.center.y + offset.y);
    }
}
```

## 动画过程中的事件处理

UIView 动画事务执行的是隐式动画，视图的实际位置在动画开始时就已经更新到目标值，因此通过手势识别器无法正确响应动画过程中的点击。需要通过 `presentationLayer` 获取视图的当前显示位置来判断：

```objective-c
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event {
    UITouch *touch = [touches anyObject];
    CGPoint location = [touch locationInView:self.view];
    if ([self.animatingView.layer.presentationLayer hitTest:location]) {
        NSLog(@"点击了正在动画的视图");
    }
}
```
