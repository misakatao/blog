---
title: "UI 触摸与手势"
description: "iOS 开发中触摸事件与手势识别的使用"
date: 2016-08-13T14:24:07+08:00
slug: ios-touch-gesture
image: "cover.svg"
categories:
    - iOS
tags:
    - UI
    - iOS开发
---
在 iOS 交互开发中，触摸事件与手势识别是构建用户输入体验的基础能力。理解 UITouch 的事件分发机制，以及 UIGestureRecognizer 的使用方式，有助于在复杂界面中更清晰地组织交互逻辑，并提升代码的可维护性。

<!--more-->

## 触摸事件（UITouch）

### UITouch 常用事件方法

UIView 及其子类可以通过以下方法直接响应用户的触摸事件：

```objective-c
// 触摸开始
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;

// 触摸移动
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;

// 触摸结束
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;

// 触摸取消
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event;
```

### 通过 touchesMoved: 实现视图跟随手指移动

在某些交互场景中，需要让视图跟随用户手指轨迹移动。这类需求可以通过 `touchesMoved:withEvent:` 方法实现。下面的示例演示了如何根据触摸点的位移更新视图位置，并补充了必要的判空与条件判断。

```objective-c
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    [super touchesMoved:touches withEvent:event];

    UITouch *touch = [touches anyObject];
    if (!touch || touch.view != self.showView) {
        return;
    }

    CGPoint previousLocation = [touch previousLocationInView:self.view];
    CGPoint currentLocation = [touch locationInView:self.view];
    CGPoint offset = CGPointMake(currentLocation.x - previousLocation.x,
                                 currentLocation.y - previousLocation.y);

    self.showView.center = CGPointMake(self.showView.center.x + offset.x,
                                       self.showView.center.y + offset.y);
}
```

## 手势识别（UIGestureRecognizer）

### 基本概念

Cocoa Touch 提供了统一的手势处理基类 `UIGestureRecognizer`，用于将底层触摸事件封装为更高层次的交互语义。常见的手势识别器包括：

- 点击手势：`UITapGestureRecognizer`
- 缩放手势：`UIPinchGestureRecognizer`
- 旋转手势：`UIRotationGestureRecognizer`
- 滑动手势：`UISwipeGestureRecognizer`
- 拖动手势：`UIPanGestureRecognizer`
- 长按手势：`UILongPressGestureRecognizer`

手势识别器的常见初始化方法如下：

```objective-c
- (instancetype)initWithTarget:(id)target action:(SEL)action;
```

为视图添加或移除手势识别器时，可使用以下方法：

```objective-c
// 添加手势
- (void)addGestureRecognizer:(UIGestureRecognizer *)gestureRecognizer;

// 移除手势
- (void)removeGestureRecognizer:(UIGestureRecognizer *)gestureRecognizer;
```

当同一视图上存在多个可能冲突的手势时，可以通过如下方法指定优先失败关系，以减少识别冲突：

```objective-c
- (void)requireGestureRecognizerToFail:(UIGestureRecognizer *)otherGestureRecognizer;
```

### 手势识别器示例

下面的示例集中演示了点击、长按、滑动、拖拽、捏合和旋转等常见手势识别器的初始化与响应处理方式。

```objective-c
#import "UIGestureRecognizerViewController.h"

#define vc_title @"Gesture"

@interface UIGestureRecognizerViewController ()

@property (nonatomic, strong) UILabel *stateLabel;
@property (nonatomic, strong) UIView *gestureView;

// 初始化用户界面
- (void)initializeUserInterface;
// 初始化手势
- (void)initializeGestureRecognizer;

@end

@implementation UIGestureRecognizerViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    [self initializeUserInterface];
    [self initializeGestureRecognizer];
}

- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];

    [self.view addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|-0-[_stateLabel]-0-|"
                                                                      options:0
                                                                      metrics:nil
                                                                        views:NSDictionaryOfVariableBindings(_stateLabel)]];
    [self.view addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"V:|-100-[_stateLabel(==30)]"
                                                                      options:0
                                                                      metrics:nil
                                                                        views:NSDictionaryOfVariableBindings(_stateLabel)]];
}

#pragma mark - Init

- (void)initializeUserInterface
{
    self.view.backgroundColor = [UIColor whiteColor];
    self.title = vc_title;

    [self.view addSubview:self.stateLabel];
    [self.view addSubview:self.gestureView];
}

- (void)initializeGestureRecognizer
{
    // 1. 点击手势识别器
    UITapGestureRecognizer *tapGesture = [[UITapGestureRecognizer alloc] initWithTarget:self
                                                                                 action:@selector(respondsToGestureRecognizer:)];
    tapGesture.numberOfTapsRequired = 1;
    tapGesture.numberOfTouchesRequired = 1;
    [self.gestureView addGestureRecognizer:tapGesture];

    // 2. 长按手势识别器
    UILongPressGestureRecognizer *longPressGesture = [[UILongPressGestureRecognizer alloc] initWithTarget:self
                                                                                                   action:@selector(respondsToGestureRecognizer:)];
    [self.gestureView addGestureRecognizer:longPressGesture];

    // 3. 滑动手势识别器
    UISwipeGestureRecognizer *swipeGesture = [[UISwipeGestureRecognizer alloc] initWithTarget:self
                                                                                        action:@selector(respondsToGestureRecognizer:)];
    swipeGesture.direction = UISwipeGestureRecognizerDirectionRight;
    [self.gestureView addGestureRecognizer:swipeGesture];

    // 4. 拖拽手势识别器
    UIPanGestureRecognizer *panGesture = [[UIPanGestureRecognizer alloc] initWithTarget:self
                                                                                 action:@selector(respondsToGestureRecognizer:)];
    [self.gestureView addGestureRecognizer:panGesture];

    // 5. 捏合手势识别器
    UIPinchGestureRecognizer *pinchGesture = [[UIPinchGestureRecognizer alloc] initWithTarget:self
                                                                                        action:@selector(respondsToGestureRecognizer:)];
    [self.gestureView addGestureRecognizer:pinchGesture];

    // 6. 旋转手势识别器
    UIRotationGestureRecognizer *rotationGesture = [[UIRotationGestureRecognizer alloc] initWithTarget:self
                                                                                                 action:@selector(respondsToGestureRecognizer:)];
    [self.gestureView addGestureRecognizer:rotationGesture];

    // 手势冲突处理：要求滑动手势在拖拽手势失败后再识别
    [swipeGesture requireGestureRecognizerToFail:panGesture];
}

#pragma mark - Gesture Event

- (void)respondsToGestureRecognizer:(UIGestureRecognizer *)gesture
{
    if ([gesture isKindOfClass:[UITapGestureRecognizer class]]) {
        UITapGestureRecognizer *tap = (UITapGestureRecognizer *)gesture;

        if (tap.state == UIGestureRecognizerStateEnded) {
            [self.stateLabel setText:@"用户点击结束"];
        }
    } else if ([gesture isKindOfClass:[UILongPressGestureRecognizer class]]) {
        UILongPressGestureRecognizer *longPress = (UILongPressGestureRecognizer *)gesture;

        if (longPress.state == UIGestureRecognizerStateBegan) {
            [self.stateLabel setText:@"长按手势开始"];
        } else if (longPress.state == UIGestureRecognizerStateEnded ||
                   longPress.state == UIGestureRecognizerStateCancelled) {
            [self.stateLabel setText:@"长按手势结束"];
        }
    } else if ([gesture isKindOfClass:[UISwipeGestureRecognizer class]]) {
        UISwipeGestureRecognizer *swipe = (UISwipeGestureRecognizer *)gesture;

        if (swipe.state == UIGestureRecognizerStateEnded) {
            [self.stateLabel setText:@"滑动手势结束"];
        }
    } else if ([gesture isKindOfClass:[UIPanGestureRecognizer class]]) {
        UIPanGestureRecognizer *pan = (UIPanGestureRecognizer *)gesture;
        static CGPoint firstCenter;

        if (pan.state == UIGestureRecognizerStateBegan) {
            firstCenter = self.gestureView.center;
            [self.stateLabel setText:@"拖拽手势开始"];
        } else if (pan.state == UIGestureRecognizerStateChanged) {
            CGPoint translation = [pan translationInView:self.view];
            self.gestureView.center = CGPointMake(firstCenter.x + translation.x,
                                                  firstCenter.y + translation.y);
            [self.stateLabel setText:@"拖拽手势拖动中"];
        } else if (pan.state == UIGestureRecognizerStateEnded ||
                   pan.state == UIGestureRecognizerStateCancelled) {
            firstCenter = self.gestureView.center;
            [self.stateLabel setText:@"拖拽手势结束"];
        }
    } else if ([gesture isKindOfClass:[UIPinchGestureRecognizer class]]) {
        UIPinchGestureRecognizer *pinch = (UIPinchGestureRecognizer *)gesture;
        static CGFloat lastScale;

        if (pinch.state == UIGestureRecognizerStateBegan) {
            lastScale = pinch.scale;
            [self.stateLabel setText:@"捏合手势开始"];
        } else if (pinch.state == UIGestureRecognizerStateChanged) {
            CGFloat changeScale = (pinch.scale - lastScale) / 2.0 + 1.0;
            CGFloat targetWidth = CGRectGetWidth(self.gestureView.bounds) * changeScale;

            if (targetWidth < 100.0) {
                return;
            }

            self.gestureView.transform = CGAffineTransformScale(self.gestureView.transform,
                                                                changeScale,
                                                                changeScale);
            lastScale = pinch.scale;
            [self.stateLabel setText:@"捏合手势变化中"];
        } else if (pinch.state == UIGestureRecognizerStateEnded ||
                   pinch.state == UIGestureRecognizerStateCancelled) {
            lastScale = 1.0;
            [self.stateLabel setText:@"捏合手势结束"];
        }
    } else if ([gesture isKindOfClass:[UIRotationGestureRecognizer class]]) {
        UIRotationGestureRecognizer *rotation = (UIRotationGestureRecognizer *)gesture;
        static CGFloat lastRotate;

        if (rotation.state == UIGestureRecognizerStateBegan) {
            lastRotate = rotation.rotation;
            [self.stateLabel setText:@"旋转手势开始"];
        } else if (rotation.state == UIGestureRecognizerStateChanged) {
            CGFloat changeRotate = rotation.rotation - lastRotate;
            self.gestureView.transform = CGAffineTransformRotate(self.gestureView.transform,
                                                                 changeRotate);
            lastRotate = rotation.rotation;
            [self.stateLabel setText:@"旋转手势变化中"];
        } else if (rotation.state == UIGestureRecognizerStateEnded ||
                   rotation.state == UIGestureRecognizerStateCancelled) {
            lastRotate = 0.0;
            [self.stateLabel setText:@"旋转手势结束"];
        }
    }
}

#pragma mark - Getter

- (UILabel *)stateLabel
{
    if (!_stateLabel) {
        _stateLabel = [[UILabel alloc] init];
        _stateLabel.translatesAutoresizingMaskIntoConstraints = NO;
        _stateLabel.textAlignment = NSTextAlignmentCenter;
        _stateLabel.font = [UIFont systemFontOfSize:25.0];
    }

    return _stateLabel;
}

- (UIView *)gestureView
{
    if (!_gestureView) {
        _gestureView = [[UIView alloc] init];
        _gestureView.bounds = CGRectMake(0, 0, 200, 200);
        _gestureView.center = self.view.center;
        _gestureView.backgroundColor = [UIColor redColor];
        _gestureView.userInteractionEnabled = YES;
    }

    return _gestureView;
}

@end
```
