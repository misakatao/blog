---
title: "Category 底层原理研究"
description: "深入研究 Objective-C Category 的底层实现原理"
date: 2018-12-03T14:24:07+08:00
slug: ios-category-internals
image: "cover.svg"
categories:
    - iOS
tags:
    - Objective-C
    - Runtime
---

本文结合 objc4 源码，梳理 Objective-C 中 Category 的加载、附加与方法调用机制，重点分析其在 Runtime 初始化阶段的处理流程，以及 `+load`、`+initialize` 两个关键方法的执行时机与差异。

<!--more-->

## Category 的加载处理过程

在这篇博客 [iOS 程序 main 函数之前发生了什么 ](https://blog.sunnyxx.com/2014/08/30/objc-pre-main/) 中提到，`_objc_init` 是 runtime 系统的初始化函数。因此，我们可以直接从 `_objc_init` 开始分析。Category 加载过程中的函数调用顺序如下：

```objective-c
void _objc_init(void);
└── void map_images(...);
    └── void map_images_nolock(...);
        └── void _read_images(...);
            └── static void addUnattachedCategoryForClass(...);
                └── static void remethodizeClass(Class cls);
                    └── static void attachCategories(Class cls, category_list *cats, bool flush_caches);
```

| 文件名              | 方法                               |
| ------------------- | ---------------------------------- |
| objc-os.mm          | `_objc_init`                       |
| objc-os.mm          | `map_images`                       |
| objc-os.mm          | `map_images_nolock`                |
| objc-runtime-new.mm | `_read_images`                     |
| objc-runtime-new.mm | `addUnattachedCategoryForClass`    |
| objc-runtime-new.mm | `remethodizeClass`                 |
| objc-runtime-new.mm | `attachCategories`                 |
| objc-runtime-new.mm | `attachLists`                      |

`_read_images` 函数会处理当前镜像文件的头部信息，主要步骤如下：

1. 获取镜像文件中的类列表，遍历列表并读取类信息（调用 `readClass`）
2. 遍历并注册所有 `selector` 名字（调用 `__sel_registerName`）
3. 遍历读取协议列表（调用 `_getObjc2ProtocolList`、`readProtocol`）
4. 遍历读取分类列表（调用 `_getObjc2CategoryList`、`addUnattachedCategoryForClass`、`remethodizeClass`）
5. 遍历并实例化运行时类结构（调用 `realizeClass`）

其中，与分类相关的代码如下：

```c++
// Discover categories.
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }
```

在上面的代码中，首先通过 `_getObjc2CategoryList` 获取 `category` 列表 `catlist`，随后遍历该列表，取出 category 对应的 Class。接着，根据 Class（类对象和元类对象）上的实例方法、协议、属性等信息，判断是否调用 `addUnattachedCategoryForClass`，并进一步决定是否执行 `remethodizeClass`。

```c++
static void addUnattachedCategoryForClass(category_t *cat, Class cls, 
                                          header_info *catHeader)
{
    runtimeLock.assertWriting();

    // DO NOT use cat->cls! cls may be cat->cls->isa instead
    NXMapTable *cats = unattachedCategories();
    category_list *list;

    list = (category_list *)NXMapGet(cats, cls);
    if (!list) {
        list = (category_list *)
            calloc(sizeof(*list) + sizeof(list->list[0]), 1);
    } else {
        list = (category_list *)
            realloc(list, sizeof(*list) + sizeof(list->list[0]) * (list->count + 1));
    }
    list->list[list->count++] = (locstamped_category_t){cat, catHeader};
    NXMapInsert(cats, cls, list);
}
static NXMapTable *unattachedCategories(void)
{
    runtimeLock.assertWriting();
	
    static NXMapTable *category_map = nil;
    if (category_map) return category_map;
    // fixme initial map size
    category_map = NXCreateMapTable(NXPtrValueMapPrototype, 16);
    return category_map;
}
```

在 `addUnattachedCategoryForClass` 中，会先通过 `unattachedCategories()` 创建一个单例 `MapTable` 对象 `cats`。随后从该对象中获取当前 category 对应的 `list` 指针，判断其是否为空，并据此分配或扩容内存，最后将 category 数据插入到这个单例 `MapTable` 中。

在 `remethodizeClass` 中，会进一步调用 `attachCategories`，将分类信息附加到对应的类上。`attachCategories` 会分别把分类中的方法列表、属性列表和协议列表合并到类中，并且默认类别列表的加载顺序与类别文件的加载顺序一致。

```c++
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // 重新分配内存
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

其中，真正负责加载的函数是 `attachLists`，其关键实现如下：

```c
array()->count = newCount;
memmove(array()->lists + addedCount, 
        array()->lists, oldCount * sizeof(array()->lists[0]));
memcpy(array()->lists, 
       addedLists, addedCount * sizeof(array()->lists[0]));
```

在 `attachLists` 中，主要关注两个变量：`array()->lists` 和 `addedLists`。

- **array()->lists**：类对象原有的方法列表、属性列表、协议列表
- **addedLists**：传入的所有分类的方法列表、属性列表、协议列表

这段代码的作用是：先通过 `memmove` 将原有类的方法、属性、协议列表整体后移，再通过 `memcpy` 将传入的分类方法、属性、协议列表填充到起始位置。

这里可以简单总结一下整个过程：

> 1、通过 Runtime 加载某个类的所有 Category 数据
>
> 2、把所有 Category 的方法、属性、协议数据合并到一个大数组中，后参与编译的 Category 数据会排在数组前面
>
> 3、将合并后的分类数据（方法、属性、协议）插入到类原有数据的前面

## 拓展

### load 源码分析

通过 [objc4](https://opensource.apple.com/tarballs/objc4/) 源码进行分析，`load` 的调用过程中的函数顺序如下：

```text
void _objc_init(void);
└── void load_images(...);
    └── void call_load_methods(...);
        └── void call_class_loads(...);
```

在 `load_images` 函数中，核心逻辑是调用 `prepare_load_methods` 与 `call_load_methods`。

`prepare_load_methods` 的作用是提前准备好满足 `+load` 方法调用条件的类和分类，以供后续执行。在这个过程中会调用 `schedule_class_load(Class cls)`，并且在处理入参时递归调用父类，以确保父类优先。

`call_load_methods` 会循环调用所有类的 `+load` 方法。需要注意的是，这里无论是类还是分类的 `+load`，都是通过函数地址 `(*load_method)(cls, SEL_load);` 直接调用，而不是通过消息发送 `objc_msgSend`。

- 分析 `call_load_methods` 源码

```c++
void call_load_methods(void)
{
    static BOOL loading = NO;
    BOOL more_categories;

    recursive_mutex_assert_locked(&loadMethodLock);

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

从中可以看出：

1. 通过 `do-while` 循环不断加载类的 `+load` 方法，`call_class_loads` 是执行 `+load` 的核心函数
2. 会先调用完所有类的 `+load` 方法，再调用分类的 `+load` 方法

- 分析 `call_class_loads` 源码

```c++
static void call_class_loads(void)
{
    int i;

    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;

    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        // 找到类中的 load 方法，并初始化一个指针指向它
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue;

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        // 通过 load 方法的内存地址直接调用
        (*load_method)(cls, SEL_load);
    }

    // Destroy the detached list.
    if (classes) _free_internal(classes);
}
```

这个函数的作用是真正执行类的 `+load` 方法。它会先从全局变量 `loadable_classes` 中取出所有可调用的类，并将相关计数清零。其中，`loadable_classes` 指向保存类信息的内存首地址，`loadable_classes_allocated` 表示已分配的内存大小，`loadable_classes_used` 表示已使用的内存大小。随后再循环调用所有类的 `+load` 方法。需要注意的是，这里与分类 `+load` 一样，都是通过函数地址 `(*load_method)(cls, SEL_load)` 直接调用，而不是通过消息发送 `objc_msgSend`。

**`+ (void)load` 小总结**

1. `+load` 方法会在 runtime 加载类或分类时调用
2. 每个类和分类的 `+load` 在程序运行过程中只会调用一次
3. 调用顺序为：父类 -> 子类 -> 父类的 category -> 子类的 category

```text
2018-12-03 15:33:36.915480+0800 CALayerDemo[20979:460542] Foo +[Foo load]
2018-12-03 15:33:36.916050+0800 CALayerDemo[20979:460542] SubFoo +[SubFoo load]
2018-12-03 15:33:36.916125+0800 CALayerDemo[20979:460542] Foo(Test) +[Foo(Test) load]
2018-12-03 15:33:36.916196+0800 CALayerDemo[20979:460542] SubFoo(Test) +[SubFoo(Test) load]
```

### initialize 源码分析

通过 [objc4](https://opensource.apple.com/tarballs/objc4/) 源码进行分析，`initialize` 的调用过程中的函数顺序如下：

```text
Method class_getInstanceMethod(Class cls, SEL sel);
└── IMP lookUpImpOrNil(Class cls, SEL sel, id inst, bool initialize, bool cache, bool resolver);
    └── IMP lookUpImpOrForward(Class cls, SEL sel, id inst, bool initialize, bool cache, bool resolver);
        └── void _class_initialize(Class cls);
        	  └── void callInitialize(Class cls);
```

**`+ (void)initialize` 小总结**

1. `+initialize` 方法会在类第一次接收到消息时调用
2. 先调用父类的 `+initialize`，再调用子类的 `+initialize`
3. 先初始化父类，再初始化子类，每个类只会初始化 1 次

```text
2018-12-03 15:33:36.916315+0800 CALayerDemo[20979:460542] Foo(Test) +[Foo(Test) initialize]
2018-12-03 15:33:36.916390+0800 CALayerDemo[20979:460542] SubFoo(Test) +[SubFoo(Test) initialize]
```

### load 与 initialize 对比

| 条件                     | +load                               | +initialize                         |
| ------------------------ | ----------------------------------- | ----------------------------------- |
| 关键方法                 | `(*load_method)(cls, SEL_load)`     | `objc_msgSend`                      |
| 调用时机                 | 被添加到 runtime 时                 | 收到第一条消息前，可能永远不调用    |
| 调用顺序                 | 父类 -> 子类 -> 父类分类 -> 子类分类 | 父类 -> 子类或(父类分类 -> 子类分类) |
| 调用次数                 | 1 次                                | 多次                                |
| 是否需要显式调用父类实现 | 否                                  | 否                                  |
| 是否沿用父类的实现       | 否                                  | 是                                  |
| 分类中的实现             | 类和分类都执行                      | 覆盖类中的方法，只执行分类的实现    |

## 参考

[iOS 程序 main 函数之前发生了什么 ](https://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

[Objc runtime 初始化过程分析](https://www.jianshu.com/p/b3b34d40e068)
