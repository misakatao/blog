---
title: "iOS开发：排序算法在OC中的实现"
description: "常见排序算法在 Objective-C 中的实现"
date: 2016-11-23T13:41:07+08:00
slug: ios-sorting-algorithms
image: "cover.svg"
categories:
    - iOS
tags:
    - iOS开发
    - 算法
---

排序算法是计算机科学中最基础也最重要的算法之一。本文整理了选择排序、冒泡排序、插入排序、归并排序、快速排序及其变体、堆排序共七种常见排序算法，并给出了基于 Objective-C 的实现。所有排序方法均以 `NSMutableArray` 分类的形式封装，便于直接集成到 iOS 项目中使用。

<!--more-->

## 公共接口定义

以下是 `NSMutableArray` 排序分类的接口声明，所有排序方法均依赖 `comparator` 比较器和 `mb_exchangeWithIndexA:indexB:` 交换方法：

```objective-c
typedef NSComparisonResult (^MBSortComparator)(id obj1, id obj2);

@interface NSMutableArray (MBSort)

@property (nonatomic, copy) MBSortComparator comparator;

- (void)mb_exchangeWithIndexA:(NSInteger)indexA indexB:(NSInteger)indexB;

- (void)mb_selectionSort;
- (void)mb_bubbleSort;
- (void)mb_insertionSort;
- (void)mb_mergeSort;
- (void)mb_quickSort;
- (void)mb_identicalQuickSort;
- (void)mb_quick3WaysSort;
- (void)mb_heapSort;

@end
```

## 选择排序

选择排序是一种简单直观的排序算法。其核心思想是每次从未排序区间中选出最小（或最大）元素，放到已排序区间的末尾，直到全部有序。无论输入数据如何分布，时间复杂度始终为 O(n²)，适用于数据规模较小的场景。

### 复杂度

- 时间复杂度：O(n²)
- 空间复杂度：O(1)
- 不稳定排序

### 实现

```objective-c
- (void)mb_selectionSort {
    for (NSInteger i = 0; i < self.count; i++) {
        for (NSInteger j = i + 1; j < self.count; j++) {
            if (self.comparator(self[i], self[j]) == NSOrderedDescending) {
                [self mb_exchangeWithIndexA:i indexB:j];
            }
        }
    }
}
```

## 冒泡排序

冒泡排序通过反复遍历待排序数列，依次比较相邻元素并在顺序错误时交换它们。每轮遍历会将当前未排序部分的最大元素"浮"到末尾。当某轮遍历未发生任何交换时，说明数列已有序，可提前终止。

### 复杂度

- 时间复杂度：O(n²)，最优 O(n)（已有序时）
- 空间复杂度：O(1)
- 稳定排序

### 实现

```objective-c
- (void)mb_bubbleSort {
    BOOL swapped;
    do {
        swapped = NO;
        for (NSInteger i = 1; i < self.count; i++) {
            if (self.comparator(self[i - 1], self[i]) == NSOrderedDescending) {
                swapped = YES;
                [self mb_exchangeWithIndexA:i indexB:i - 1];
            }
        }
    } while (swapped);
}
```

## 插入排序

插入排序的工作原理类似于整理扑克牌：将第一个元素视为已排序序列，然后依次取出后续元素，在已排序序列中从后向前扫描，找到合适的位置插入。对于近乎有序的数据，插入排序的效率非常高。

### 复杂度

- 时间复杂度：O(n²)，最优 O(n)（已有序时）
- 空间复杂度：O(1)
- 稳定排序

### 实现

```objective-c
- (void)mb_insertionSort {
    for (NSInteger i = 1; i < self.count; i++) {
        id e = self[i];
        NSInteger j = i;
        for (; j > 0 && self.comparator(self[j - 1], e) == NSOrderedDescending; j--) {
            self[j] = self[j - 1];
        }
        self[j] = e;
    }
}
```

## 归并排序

归并排序是基于分治法（Divide and Conquer）的经典排序算法。其核心思想是将数组递归地拆分为两半，分别排序后再合并。本文采用自顶向下的递归实现方式。

### 复杂度

- 时间复杂度：O(n log n)
- 空间复杂度：O(n)
- 稳定排序

### 实现

```objective-c
- (void)mb_mergeSort {
    [self mb_mergeSortWithLeftIndex:0 rightIndex:(NSInteger)self.count - 1];
}

- (void)mb_mergeSortWithLeftIndex:(NSInteger)l rightIndex:(NSInteger)r {
    if (l >= r) return;
    NSInteger mid = l + (r - l) / 2;
    [self mb_mergeSortWithLeftIndex:l rightIndex:mid];
    [self mb_mergeSortWithLeftIndex:mid + 1 rightIndex:r];
    if (self.comparator(self[mid], self[mid + 1]) == NSOrderedDescending) {
        [self mb_mergeWithLeftIndex:l midIndex:mid rightIndex:r];
    }
}

- (void)mb_mergeWithLeftIndex:(NSInteger)l midIndex:(NSInteger)mid rightIndex:(NSInteger)r {
    NSMutableArray *aux = [NSMutableArray arrayWithCapacity:r - l + 1];
    for (NSInteger i = l; i <= r; i++) {
        [aux addObject:self[i]];
    }
    NSInteger i = l, j = mid + 1;
    for (NSInteger k = l; k <= r; k++) {
        if (i > mid) {
            self[k] = aux[j - l];
            j++;
        } else if (j > r) {
            self[k] = aux[i - l];
            i++;
        } else if (self.comparator(aux[i - l], aux[j - l]) != NSOrderedDescending) {
            self[k] = aux[i - l];
            i++;
        } else {
            self[k] = aux[j - l];
            j++;
        }
    }
}
```

## 快速排序

快速排序由 Tony Hoare 发明，是实践中最常用的排序算法之一。其核心思想是选取一个基准元素（pivot），将数组分为小于基准和大于基准的两部分，然后递归地对两部分排序。在大多数情况下，快速排序比其他 O(n log n) 算法更快，因为其内部循环可以被高效实现。

### 复杂度

- 时间复杂度：平均 O(n log n)，最坏 O(n²)
- 空间复杂度：O(log n)（递归栈）
- 不稳定排序

### 实现

```objective-c
- (void)mb_quickSort {
    [self mb_quickSortWithLeftIndex:0 rightIndex:(NSInteger)self.count - 1];
}

- (void)mb_quickSortWithLeftIndex:(NSInteger)l rightIndex:(NSInteger)r {
    if (l >= r) return;
    NSInteger p = [self mb_partitionWithLeftIndex:l rightIndex:r];
    [self mb_quickSortWithLeftIndex:l rightIndex:p - 1];
    [self mb_quickSortWithLeftIndex:p + 1 rightIndex:r];
}

/// 对 self[l...r] 进行 partition 操作，返回 p 使得 self[l...p-1] < self[p] < self[p+1...r]
- (NSInteger)mb_partitionWithLeftIndex:(NSInteger)l rightIndex:(NSInteger)r {
    // 随机选取基准元素，避免最坏情况
    NSInteger randomIndex = l + arc4random_uniform((uint32_t)(r - l + 1));
    [self mb_exchangeWithIndexA:l indexB:randomIndex];

    id pivot = self[l];
    NSInteger j = l;
    for (NSInteger i = l + 1; i <= r; i++) {
        if (self.comparator(self[i], pivot) == NSOrderedAscending) {
            j++;
            [self mb_exchangeWithIndexA:j indexB:i];
        }
    }
    [self mb_exchangeWithIndexA:l indexB:j];
    return j;
}
```

## 双路快速排序

当数组中存在大量重复元素时，基础快速排序会退化为 O(n²)。双路快速排序通过从数组两端同时向中间扫描，将等于基准的元素均匀分配到两侧，有效避免了这一问题。

### 复杂度

- 时间复杂度：平均 O(n log n)
- 空间复杂度：O(log n)（递归栈）
- 不稳定排序

### 实现

```objective-c
- (void)mb_identicalQuickSort {
    [self mb_identicalQuickSortWithLeftIndex:0 rightIndex:(NSInteger)self.count - 1];
}

- (void)mb_identicalQuickSortWithLeftIndex:(NSInteger)l rightIndex:(NSInteger)r {
    if (l >= r) return;
    NSInteger p = [self mb_partition2WithLeftIndex:l rightIndex:r];
    [self mb_identicalQuickSortWithLeftIndex:l rightIndex:p - 1];
    [self mb_identicalQuickSortWithLeftIndex:p + 1 rightIndex:r];
}

- (NSInteger)mb_partition2WithLeftIndex:(NSInteger)l rightIndex:(NSInteger)r {
    NSInteger randomIndex = l + arc4random_uniform((uint32_t)(r - l + 1));
    [self mb_exchangeWithIndexA:l indexB:randomIndex];

    id pivot = self[l];
    // self[l+1...i) <= pivot, self(j...r] >= pivot
    NSInteger i = l + 1, j = r;
    while (YES) {
        while (i <= r && self.comparator(self[i], pivot) == NSOrderedAscending) {
            i++;
        }
        while (j >= l + 1 && self.comparator(self[j], pivot) == NSOrderedDescending) {
            j--;
        }
        if (i > j) break;
        [self mb_exchangeWithIndexA:i indexB:j];
        i++;
        j--;
    }
    [self mb_exchangeWithIndexA:l indexB:j];
    return j;
}
```

## 三路快速排序

三路快速排序将数组分为小于、等于、大于基准值的三个区间，对包含大量重复元素的数组具有显著优势。许多语言的标准库排序（如 Java 的 `Arrays.sort`）都采用了三路快排的思想。

### 复杂度

- 时间复杂度：平均 O(n log n)，大量重复元素时接近 O(n)
- 空间复杂度：O(log n)（递归栈）
- 不稳定排序

### 实现

```objective-c
- (void)mb_quick3WaysSort {
    [self mb_quick3WaysSortWithLeftIndex:0 rightIndex:(NSInteger)self.count - 1];
}

- (void)mb_quick3WaysSortWithLeftIndex:(NSInteger)l rightIndex:(NSInteger)r {
    if (l >= r) return;

    // 随机选取基准元素
    NSInteger randomIndex = l + arc4random_uniform((uint32_t)(r - l + 1));
    [self mb_exchangeWithIndexA:l indexB:randomIndex];

    id pivot = self[l];

    NSInteger lt = l;      // self[l+1...lt] < pivot
    NSInteger gt = r + 1;  // self[gt...r] > pivot
    NSInteger i = l + 1;   // self[lt+1...i) == pivot

    while (i < gt) {
        NSComparisonResult result = self.comparator(self[i], pivot);
        if (result == NSOrderedAscending) {
            lt++;
            [self mb_exchangeWithIndexA:i indexB:lt];
            i++;
        } else if (result == NSOrderedDescending) {
            gt--;
            [self mb_exchangeWithIndexA:i indexB:gt];
        } else {
            i++;
        }
    }
    [self mb_exchangeWithIndexA:l indexB:lt];

    [self mb_quick3WaysSortWithLeftIndex:l rightIndex:lt - 1];
    [self mb_quick3WaysSortWithLeftIndex:gt rightIndex:r];
}
```

## 堆排序

堆排序利用堆这一数据结构完成排序。对于升序排序，通常先构建大顶堆，然后将堆顶元素与末尾元素交换，并对剩余区间继续执行下沉调整。与快速排序相比，堆排序的时间复杂度更加稳定，但通常不具备快速排序那样优秀的缓存局部性。

### 复杂度

- 时间复杂度：O(n log n)
- 空间复杂度：O(1)
- 不稳定排序

### 实现

```objective-c
- (void)mb_heapSort {
    NSInteger count = self.count;
    if (count <= 1) return;

    for (NSInteger i = (count - 2) / 2; i >= 0; i--) {
        [self mb_shiftDownFromIndex:i heapSize:count];
        if (i == 0) break;
    }

    for (NSInteger i = count - 1; i > 0; i--) {
        [self mb_exchangeWithIndexA:0 indexB:i];
        [self mb_shiftDownFromIndex:0 heapSize:i];
    }
}

- (void)mb_shiftDownFromIndex:(NSInteger)index heapSize:(NSInteger)heapSize {
    while (2 * index + 1 < heapSize) {
        NSInteger child = 2 * index + 1;
        if (child + 1 < heapSize && self.comparator(self[child + 1], self[child]) == NSOrderedDescending) {
            child++;
        }
        if (self.comparator(self[index], self[child]) != NSOrderedAscending) {
            break;
        }
        [self mb_exchangeWithIndexA:index indexB:child];
        index = child;
    }
}
```

## 总结

本文将多种经典排序算法统一整理为 `NSMutableArray` 分类实现，便于在 iOS 项目中直接调用。从工程实践角度看，选择排序、冒泡排序和插入排序更适合用来理解基础思想；而归并排序、快速排序及其优化版本、堆排序则更适合处理规模更大的数据集。

在 Objective-C 中实现排序算法时，除了关注算法本身，还应注意接口设计、比较器抽象、内存管理方式以及方法命名的一致性。将比较逻辑通过 block 抽离出来，可以显著提升代码的复用性和可维护性。

## 参考与阅读

- [一本关于排序算法的 GitBook 在线书籍《十大经典排序算法》，使用 JavaScript、Python、Go 实现](https://github.com/hustcc/JS-Sorting-Algorithm)
- [在 JavaScript 中学习数据结构与算法](https://juejin.im/post/594dfe795188250d725a220a)
- [排序动画](https://github.com/JiongXing/JXSort)

