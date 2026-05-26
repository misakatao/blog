---
title: "UI 界面传值"
description: "iOS 开发中 UI 界面间传值的多种方式"
date: 2016-08-21T13:41:07+08:00
slug: ios-ui-value-passing
image: "cover.svg"
categories:
    - iOS
tags:
    - UI
    - iOS开发
---
## 传值需求

在 iOS 开发中，不同视图控制器之间经常需要传递数据。本文以用户信息 userInfo 为例，说明多种常见的 UI 界面传值方式及其适用场景。

<!--more-->

## 主页传值到详情页

本节模拟从主页视图控制器向详情视图控制器传递用户信息 userInfo。

### 属性传值

属性传值是最常见的方式之一，通常适用于从当前页面向下一个页面传值。

```objective-c
// DetailViewController.h
#import <UIKit/UIKit.h>

@interface DetailViewController : UIViewController

@property (nonatomic, strong) NSDictionary *userInfo; /**< 用户信息 */

@end
```

```objective-c
// HomeViewController.m
#import "DetailViewController.h"

- (void)respondsToButton:(UIButton *)sender {
    // 初始化详情视图控制器
    DetailViewController *detailVc = [[DetailViewController alloc] init];

    // 属性传值
    NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};
    detailVc.userInfo = userInfo;

    // 模态切换（界面跳转）
    [self presentViewController:detailVc animated:YES completion:nil];
}
```

```objective-c
// DetailViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"%@", self.userInfo);
}
```

### init 方法传值

init 方法传值与属性传值类似，同样适用于从当前页面向下一个页面传值。区别在于，数据在对象初始化阶段就已经传入。

```objective-c
// DetailViewController.h
#import <UIKit/UIKit.h>

@interface DetailViewController : UIViewController

@property (nonatomic, strong, readonly) NSDictionary *userInfo;

- (instancetype)initWithUserInfo:(NSDictionary *)userInfo;

@end
```

```objective-c
// DetailViewController.m
#import "DetailViewController.h"

@interface DetailViewController ()

@property (nonatomic, strong, readwrite) NSDictionary *userInfo;

@end

@implementation DetailViewController

- (instancetype)initWithUserInfo:(NSDictionary *)userInfo {
    self = [super init];
    if (self) {
        _userInfo = userInfo;
    }
    return self;
}

@end
```

```objective-c
// HomeViewController.m
#import "DetailViewController.h"

- (void)respondsToButton:(UIButton *)sender {
    NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};

    // 通过 init 方法传值
    DetailViewController *detailVc = [[DetailViewController alloc] initWithUserInfo:userInfo];

    // 模态切换（界面跳转）
    [self presentViewController:detailVc animated:YES completion:nil];
}
```

## 详情页传值到主页

本节模拟从详情视图控制器向主页视图控制器回传用户信息 userInfo。

### 协议传值

协议传值也称代理传值，适用于从后一个页面向前一个页面传值。这种方式结构清晰、可读性较高，在实际开发中使用非常广泛。

```objective-c
// DetailViewController.h
#import <UIKit/UIKit.h>

@class DetailViewController;

@protocol DetailViewControllerDelegate <NSObject>

@optional
- (void)detailViewController:(DetailViewController *)detailViewController
          goBackWithUserInfo:(NSDictionary *)userInfo;

@end

@interface DetailViewController : UIViewController

@property (nonatomic, weak) id<DetailViewControllerDelegate> delegate;

@end
```

```objective-c
// DetailViewController.m
- (void)respondsToButton:(UIButton *)sender {
    // 判断代理是否存在，并且是否实现了协议方法
    if (self.delegate && [self.delegate respondsToSelector:@selector(detailViewController:goBackWithUserInfo:)]) {
        NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};
        [self.delegate detailViewController:self goBackWithUserInfo:userInfo];
    }

    [self dismissViewControllerAnimated:YES completion:nil];
}
```

```objective-c
// HomeViewController.m
#import "DetailViewController.h"

@interface HomeViewController () <DetailViewControllerDelegate>

@end

- (void)respondsToButton:(UIButton *)sender {
    // 初始化详情视图控制器
    DetailViewController *detailVc = [[DetailViewController alloc] init];

    // 设置代理
    detailVc.delegate = self;

    // 模态切换（界面跳转）
    [self presentViewController:detailVc animated:YES completion:nil];
}

#pragma mark - DetailViewControllerDelegate

- (void)detailViewController:(DetailViewController *)detailViewController
          goBackWithUserInfo:(NSDictionary *)userInfo {
    NSLog(@"%@", userInfo);
}
```

### Block 传值

Block 传值本质上是一种回调机制，适用于从后一个页面向前一个页面传值。其优势在于代码集中、调用直观。

```objective-c
// DetailViewController.h
#import <UIKit/UIKit.h>

typedef void(^CallBackBlock)(NSDictionary *userInfo);

@interface DetailViewController : UIViewController

@property (nonatomic, copy) CallBackBlock callBackBlock;

- (void)getsUserInfoWithBlocks:(CallBackBlock)callBackBlock;

@end
```

```objective-c
// DetailViewController.m
- (void)getsUserInfoWithBlocks:(CallBackBlock)callBackBlock {
    self.callBackBlock = callBackBlock;
}

- (void)respondsToButtonClick:(UIButton *)sender {
    if (self.callBackBlock) {
        NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};
        self.callBackBlock(userInfo);
    }

    [self dismissViewControllerAnimated:YES completion:nil];
}
```

```objective-c
// HomeViewController.m
- (void)respondsToButtonClick:(UIButton *)sender {
    DetailViewController *detailVc = [[DetailViewController alloc] init];

    // 调用 Block，接收回传的数据
    [detailVc getsUserInfoWithBlocks:^(NSDictionary *userInfo) {
        NSLog(@"%@", userInfo);
    }];

    // 模态切换（界面跳转）
    [self presentViewController:detailVc animated:YES completion:nil];
}
```

Block 传值使用时需要注意以下几点：

1. 可通过 typedef 为 Block 定义别名，便于复用与阅读。
2. Block 属性应使用 copy 关键字。
3. 应在适当时机调用 Block，以完成回调传值。
4. Block 方式常用于结构较简单的页面回传场景。

## 多界面传值

当多个控制器之间没有直接依赖关系，或者需要跨层级传值时，可考虑以下方式。

### 通知传值

通知传值适用于任意控制器之间的数据传递，即使两个控制器之间没有直接关联也可以使用。前提是接收方必须先注册通知。

```objective-c
// HomeViewController.m
#import "HomeViewController.h"
#import "DetailViewController.h"

@implementation HomeViewController

- (instancetype)init {
    self = [super init];
    if (self) {
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(respondsToNotification:)
                                                     name:@"notification_name"
                                                   object:nil];
    }
    return self;
}

- (void)respondsToNotification:(NSNotification *)notification {
    NSLog(@"%@", notification.userInfo);
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end
```

```objective-c
// DetailViewController.m
- (void)respondsToButton:(UIButton *)sender {
    NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};

    [[NSNotificationCenter defaultCenter] postNotificationName:@"notification_name"
                                                        object:nil
                                                      userInfo:userInfo];
}
```

通知传值通常包含四个步骤：注册通知、发送通知、处理通知、移除通知。若不在适当时机移除观察者，可能引发运行时问题。

### 单例传值

单例贯穿整个应用程序生命周期，因此也可作为数据共享的中转对象。该方式适用于多个控制器都需要访问同一份数据的场景。

```objective-c
// Singleton.h
#import <Foundation/Foundation.h>

@interface Singleton : NSObject

@property (nonatomic, strong) NSDictionary *userInfo;

+ (instancetype)defaultSingleton;

@end
```

```objective-c
// Singleton.m
#import "Singleton.h"

@implementation Singleton

+ (instancetype)defaultSingleton {
    static Singleton *singleton = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        singleton = [[Singleton alloc] init];
    });
    return singleton;
}

@end
```

```objective-c
// HomeViewController.m
#import "Singleton.h"

- (void)viewDidLoad {
    [super viewDidLoad];

    Singleton *singleton = [Singleton defaultSingleton];
    NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};
    singleton.userInfo = userInfo;
}
```

```objective-c
// DetailViewController.m
#import "Singleton.h"

- (void)viewDidLoad {
    [super viewDidLoad];

    Singleton *singleton = [Singleton defaultSingleton];
    NSLog(@"%@", singleton.userInfo);
}
```

使用单例传值时，必须保证在读取数据之前已经完成赋值，否则获取到的结果为 nil。

### NSUserDefaults 传值

NSUserDefaults 是系统提供的单例对象，其使用方式与自定义单例类似。它适合保存轻量级配置或简单数据，也可用于某些简单场景下的跨界面取值。

```objective-c
// HomeViewController.m
- (void)saveValueInUserDefaults {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

    NSDictionary *userInfo = @{@"name": @"Charles", @"age": @(22)};
    [defaults setObject:userInfo forKey:@"userInfo"];

    [defaults synchronize];
}
```

```objective-c
// DetailViewController.m
- (void)getValueFromUserDefaults {
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

    NSDictionary *userInfo = [defaults objectForKey:@"userInfo"];
    NSLog(@"%@", userInfo);
}
```

使用 NSUserDefaults 传值时，应确保指定 key 对应的数据已经写入成功，否则读取结果可能为 nil。