---
title: "Swift Concurrency 四种 Continuation 详解"
description: "深入解析 Swift Concurrency 中 withUnsafeContinuation、withUnsafeThrowingContinuation、withCheckedContinuation、withCheckedThrowingContinuation 四种 Continuation 的作用、区别与最佳实践"
date: 2026-05-07T11:30:00+08:00
slug: swift-continuation
image: "cover.svg"
categories:
    - iOS
tags:
    - Swift
    - Swift Concurrency
    - async/await
    - Continuation
---

Swift Concurrency 推出之后，`async/await` 逐渐替代了 callback、delegate、completionHandler、Promise 等传统异步模型。但现实工程里，大量系统 API 仍然是 completionHandler 风格。Swift 为此提供了 Continuation，用于把旧异步 API 桥接到 async/await 世界。本文完整梳理 Swift 中四种 Continuation 的原理、区别、使用方式、生命周期与最佳实践。

<!--more-->

Swift 一共提供了四种 Continuation：

| 方法 | 是否支持 Error | 是否安全检查 |
|------|----------------|--------------|
| `withUnsafeContinuation` | ❌ | ❌ |
| `withUnsafeThrowingContinuation` | ✅ | ❌ |
| `withCheckedContinuation` | ❌ | ✅ |
| `withCheckedThrowingContinuation` | ✅ | ✅ |

## 什么是 Continuation

Continuation 的本质是「暂停 async 函数，并在未来某个时间恢复执行」。比如一个 `async` 函数：

```swift
func fetchData() async -> String
```

内部可能其实是一段 callback 风格的调用：

```swift
networkRequest { result in
    completion(result)
}
```

Continuation 的职责，就是把 callback 转换成可以被 `await` 的表达式。

## 为什么需要 Continuation

因为大量系统 API 仍然是 completionHandler 风格：

```swift
URLSession.shared.dataTask(with: url) { data, response, error in
    // ...
}
```

而不是：

```swift
let data = try await ...
```

Continuation 就是两种模型之间的桥。

## Continuation 的执行原理

以 `withCheckedContinuation` 为例：

```swift
func test() async -> String {
    await withCheckedContinuation { continuation in
        someAsyncWork {
            continuation.resume(returning: "Hello")
        }
    }
}
```

执行流程大致如下：

```text
async 函数执行
    ↓
遇到 Continuation
    ↓
函数挂起（Suspend）
    ↓
保存当前上下文
    ↓
等待 resume
    ↓
resume 后恢复执行
```

Continuation 在概念上等价于「协程恢复点（Coroutine Resume Point）」。

## withUnsafeContinuation

最基础的 Continuation，不支持 throw，没有任何运行时安全检查，性能最高，风险也最大。

```swift
func loadName() async -> String {
    await withUnsafeContinuation { continuation in
        DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
            continuation.resume(returning: "Tesla")
        }
    }
}

Task {
    let value = await loadName()
    print(value) // Tesla
}
```

它的危险性在于：系统不会帮你检查 `resume` 是否正确。

**多次 resume**：

```swift
continuation.resume(returning: 1)
continuation.resume(returning: 2)
```

结果是未定义行为，可能崩溃，也可能内存错误。

**永远不 resume**：

```swift
withUnsafeContinuation { _ in
    // 什么都不做
}
```

结果是 async 永远挂起、Task 泄漏，运行期没有任何警告。

## withUnsafeThrowingContinuation

在 Unsafe 的基础上支持抛出 Error，同样没有安全检查。既可以 `resume(returning:)`，也可以 `resume(throwing:)`：

```swift
enum NetworkError: Error {
    case invalid
}

func fetch() async throws -> String {
    try await withUnsafeThrowingContinuation { continuation in
        let success = true
        if success {
            continuation.resume(returning: "Success")
        } else {
            continuation.resume(throwing: NetworkError.invalid)
        }
    }
}

Task {
    do {
        let value = try await fetch()
        print(value)
    } catch {
        print(error)
    }
}
```

## withCheckedContinuation

实际开发中最推荐使用的 Continuation 之一。不支持 throw，但带运行时安全检查，能在 Debug 阶段及时暴露错误用法。

```swift
func loadUser() async -> String {
    await withCheckedContinuation { continuation in
        DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
            continuation.resume(returning: "Misaka")
        }
    }
}
```

Checked 版本会替你检查：

| 问题 | 是否检测 |
|------|----------|
| 多次 resume | ✅ |
| 不 resume | ✅ |
| 生命周期错误 | ✅ |

当出现多次 resume 时，运行时会直接报错：

```text
Fatal error:
SWIFT TASK CONTINUATION MISUSE
```

这是 Swift Concurrency 最经典的错误之一。

## withCheckedThrowingContinuation

四者中功能最完整的版本：既支持 throw，也带安全检查，是生产环境的首选。

```swift
enum LoginError: Error {
    case failed
}

func login() async throws -> String {
    try await withCheckedThrowingContinuation { continuation in
        let success = true
        if success {
            continuation.resume(returning: "Token")
        } else {
            continuation.resume(throwing: LoginError.failed)
        }
    }
}
```

## 四种 Continuation 对比

| 方法 | Throw | 安全检查 | 推荐程度 |
|------|-------|----------|----------|
| `withUnsafeContinuation` | ❌ | ❌ | ⭐ |
| `withUnsafeThrowingContinuation` | ✅ | ❌ | ⭐ |
| `withCheckedContinuation` | ❌ | ✅ | ⭐⭐⭐⭐ |
| `withCheckedThrowingContinuation` | ✅ | ✅ | ⭐⭐⭐⭐⭐ |

## 为什么 Unsafe 性能更高

Checked 版本会额外做几件事：resume 状态检测、生命周期检测、调试信息记录、Runtime 校验；Unsafe 则完全跳过。因此 Unsafe 的单次调用开销略低。但对绝大多数业务开发来说，用安全性换这一点性能并不值得——出一次 `SWIFT TASK CONTINUATION MISUSE` 定位的时间就足以抵消所有优化收益。

## 核心规则：一次且仅一次 resume

这是 Continuation 最关键的约束。下面这种写法很容易踩坑：

```swift
if success {
    continuation.resume(returning: value)
}
continuation.resume(returning: fallback)
```

`success` 为 true 时会触发双 resume。有两种常见的防御写法。

**显式 return 提前退出**：

```swift
if success {
    continuation.resume(returning: value)
    return
}
continuation.resume(returning: fallback)
```

**用标记位管理状态**（特别适合 delegate 桥接）：

```swift
var resumed = false

func safeResume() {
    guard !resumed else { return }
    resumed = true
    // ...实际 resume
}
```

## Continuation 与 DispatchSemaphore 的区别

容易产生的误解是把 Continuation 等同于「阻塞线程」，实际上两者完全不同。

`DispatchSemaphore.wait()` 会**阻塞当前线程**，线程资源被占用，不小心就会死锁；`Continuation` 只是**挂起当前 Task**，线程可以继续跑其它任务。前者是线程级阻塞，后者是协程级调度——这正是 Swift Concurrency 能做到高并发的核心之一。

## Continuation 与 Actor

Continuation 恢复后不一定回到原来的线程。`resume` 之后的代码可能跑在主线程、后台线程，或者某个 Actor 的 Executor 上，取决于调度器当时的决定。所以不要依赖线程来保证 UI 安全，而应依赖 `@MainActor` 等 Actor 机制。

## 桥接 Delegate API

这是 Continuation 最经典的落地场景。以 `CLLocationManager` 为例，旧写法是调用 `requestLocation()` 后等待 `didUpdateLocations` 回调。用 Continuation 桥接后可以写成：

```swift
func requestLocation() async throws -> CLLocation {
    try await withCheckedThrowingContinuation { continuation in
        self.continuation = continuation
        locationManager.requestLocation()
    }
}

func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations locations: [CLLocation]
) {
    continuation?.resume(returning: locations.first!)
    continuation = nil
}
```

注意最后要把属性置空，避免下次 delegate 回调再次触发已消费的 continuation。

## 常见崩溃场景

几乎所有与 Continuation 有关的崩溃都会报：

```text
SWIFT TASK CONTINUATION MISUSE
```

常见原因：

| 原因 | 说明 |
|------|------|
| 多次 resume | 最常见 |
| deinit 后 resume | 生命周期错误 |
| 不 resume | Task 永久挂起 |
| 并发 resume | 多线程竞争下的重复调用 |

## 最佳实践

| 规则 | 推荐 |
|------|------|
| 默认使用 Checked 版本 | ✅ |
| 非必要不用 Unsafe | ✅ |
| 一次且仅一次 resume | ✅ |
| 避免跨生命周期持有 continuation | ✅ |
| 注意并发竞争 | ✅ |

生产环境优先选择 `withCheckedThrowingContinuation`，它最安全、支持 Error、Debug 友好、易维护。只有在极限性能场景、Runtime 开销敏感、系统底层库开发这类场合，才考虑使用 Unsafe 版本。

## 小结

Swift Continuation 本质上是一个「协程恢复控制器」，用于让 callback 世界接入 async/await 世界。四种 Continuation 的选择可以简化为一句话：

**默认使用 `withCheckedThrowingContinuation`。**

它背后涉及到的协程调度、Task Suspend、Runtime 恢复机制、Actor Executor，才是 Swift Concurrency 真正强大的地方。

## 延伸阅读

- Swift Structured Concurrency
- Actor 与 `@MainActor`
- Task 与 TaskGroup
- AsyncSequence
- Coroutine 实现原理
