---
layout: post
title: Objective-C Runtime(三)类的加载与初始化
fullview: true
category: iOS
tags: Runtime
keywords: Runtime,load,initialize
---


我们知道，iOS App的`main()`函数位于`main.m`中，这是我们熟知的程序入口，但是在这之前， 还要先进行加载framework、初始化`runtime`等操作,framework的加载是由dylb调用的，关于dylb，这里不作探讨，感兴趣的同学可以看看这篇文章([dyld: Dynamic Linking On OS X](https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html) )。本文结合`runtime`源码，对类的加载和初始化中涉及到的两个方法`+load`和`+initialize`进行探讨，了解他们的调用时机和调用方式，

所用到的`runtime`源码为[objc4-680.tar.gz](http://opensource.apple.com/tarballs/objc4/objc4-680.tar.gz) 。



# load

`runtime`的初始化函数在`objc-os.mm`中的`_objc_init`中, 这个方法在old-ABI中是由`dylb`在初始化动态链接库的时候调用的，现在是由libSystem在动态链接库初始化之前调用的：

```c

/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Old ABI: called by dyld as a library initializer
* New ABI: called by libSystem BEFORE library initialization time
**********************************************************************/

void _objc_init(void)
{
    ......
    // Register for unmap first, in case some +load unmaps something
    _dyld_register_func_for_remove_image(&unmap_image);
    dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_2_images);
    dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}
```

在这里，`runtime`先会调用`map_2_images`方法, 加载`images`到内存中，`category`中扩展的属性和方法就是在这个时候附加到类上的，这将在后续的文章中进行探讨。然后会调用`load_images`这个方法：

```c

const char *load_images(enum dyld_image_states state, uint32_t infoCount, const struct dyld_image_info infoList[])
{
    bool found;
    // Return without taking locks if there are no +load methods here.
    found = false;
    ......
    // Discover load methods
    {
        rwlock_writer_t lock2(runtimeLock);
        //先load images
        found = load_images_nolock(state, infoCount, infoList);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    if (found) {
               //然后调用类的+load方法
        call_load_methods();
    }
    return nil;
}
```

`load_images`在这个方法里先是调用`load_images_nolock`方法, 在这个方法里会调用`prepare_load_methods`方法去准备好要被调用的`+load`方法，我们先来看下`prepare_load_methods`方法的实现：


```objc

void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertWriting();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }
    
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```

在这里，先是`schedule_class_load(Class cls)`方法去准备好所有满足`＋load`方法调用条件的类，这个方法会对入参的父类进行递归调用，以确保**父类优先**的顺序：

```c

static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```

当`prepare_load_methods`函数执行完之后，所有满足`+load`方法调用条件的类和分类就被分别保存在全局变量`loadable_classes`和`loadable_categories`中了。

准备好类和分类之后，接下来就是对他们的`＋load`方法进行调用了，找到`call_load_methods`方法:

```c

void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

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

在这个方法里，会以**类优先于分类的顺序**调用`+load`方法，这里有两个关键的函数`call_class_loads()`和`call_category_loads`,这两个函数会遍历上一步中准备好的`loadable_classes`和`loadable_categories`的`＋load`方法，需要注意的是他们都是以函数内存地址的方式`((*load_method)(cls, SEL_load))`对`＋load`方法进行调用的，而不是使用发送消息`objc_msgSend`的方式.

> 这样的调用方式就使得 +load 方法拥有了一个特性，那就是子类、父类和分类中的 +load 方法的实现是被区别对待的。也就是说如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的。同理，当一个类和它的分类都实现了 `+load` 方法时，两个方法都会被调用，多个分类实现了`+load`方法时，每个分类的`＋load`方法都会被调用。



--- 

# initialize

 `+initialize`方法是在类或类的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用。也就是说`＋initialize`方法是以懒加载的方式被调用的，如果一直没有给一个类或他的子类发送消息，那么这个类的`＋initialize`方法是永远不会调用的。

当我们向某个类发送消息时，`runtime`会调用`IMP lookUpImpOrForward(...)`这个函数在类中查找相应方法的实现或进行消息转发，打开`objc-runtime-new.h`找到这个函数：

```c


IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    ...
    if (initialize  &&  !cls->isInitialized()) {
            // 类没有初始化时，对类进行初始化
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    ...
}
```

从中可以看到当类没有初始化时，会调用`_class_initialize(Class cls)`对类进行初始化：

```c

void _class_initialize(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    BOOL reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    //递归调用，对父类进行_class_initialize调用，确保父类的initialize方法比子类先调用
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    
    ......
    
    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.
        
        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);
        
        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if 
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: calling +[%s initialize]",
                         cls->nameForLogging());
        }
        //发送调用类方法initialize的消息
        ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);

        ......
}
```

在这里，先是对入参的父类进行递归调用，以确保父类优先于子类初始化，还有一个关键的地方：`runtime`使用了发送消息`objc_msgSend`的方式对`＋initialize`方法进行调用，这样，`＋initialize`方法的调用就与普通方法的调用是一致的，都是走的发送消息的流程，所以，如果子类没有实现`＋initialize`方法，将会沿着继承链去调用父类的`＋initialize`方法，同理，分类中的`＋initialize`方法会覆盖原本类的方法。



虽然对每个类只会调用一次`_class_initialize(Class cls)`方法，但是由于`+initialize`方法的调用走的是消息发送的流程，当某个类有多个子类时，这个类的`＋initialize`方法有可能会被多次调用，这时，可能需要在`＋initialize`方法中判断是否是由子类调用的:

```c

+ (void)initialize{
    if (self == [ClassName class]) {
        ......
    }
}
```



# 总结：

1. `+load`方法在类加载时被调用，`+initialize`方法在类收到第一条消息时调用,可能永远不会调用

2.  两个方法的调用顺序都是先父类后子类，

3.  `+load`方法在`runtime`中是直接以函数地址的方式进行调用，如果有多个分类，所有分类的`＋load`方法也会被调用，并且在类的`＋load`方法之后被调用，多个分类的`＋load`方法调用顺序取决于编译的顺序

4. `＋initialize`方法在`runtime`中是以发送消息的方式调用的，所以子类会覆盖父类的实现，分类会覆盖类的实现，多个分类只会调用一个分类的`+initialize`方法。



## 参考链接

- [https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html]()

- [http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/]() 