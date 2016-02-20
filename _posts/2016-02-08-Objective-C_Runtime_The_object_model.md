---
layout: post
title: Objective-C Runtime(一)对象模型及类与元类
fullview: true
category: iOS
tags: Runtime
keywords: Runtime,对象模型，metaclass
description: Objective-C 是一个动态语言，这意味着它不仅需要一个编译器，也需要一个运行时系统来动态得创建类和对象、进行消息传递和转发。理解 Objective-C 的 Runtime 机制可以帮我们更好的了解这个语言，适当的时候还能对语言进行扩展，从系统层面解决项目中的一些设计或技术问题
---




#一、什么是Objective-C Runtime

下面是摘自[官网的一段原文](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)

The Objective-C language defers as many decisions as it can from compile time and link time to runtime. Whenever possible, it does things dynamically. This means that the language requires not just a compiler, but also a runtime system to execute the compiled code. The runtime system acts as a kind of operating system for the Objective-C language; it’s what makes the language work.

翻译一下:

Objective-C语言会将编译和链接时要做的事推迟到运行时进行，这就意味着Objective-C语言不仅需要一个编译器，还需要一个运行时系统来执行编译好的代码。这个运行时系统就好比Objective-C语言的操作系统， Objective-C基于Objective-C Runtime来工作。

简单的来说，就是Objective-C扩展了 C 语言，并加入了面向对象特性和消息传递机制。而这个扩展的核心是一个用C和汇编语言写的 Runtime 库。它是 Objective-C 面向对象和动态机制的基石。



#二、Objective-C 对象模型

##对象

Objective-C是一门面向对象的语言，我们无时无刻地在和对象打交道，对象本质上是一个`objc_object`结构体，他包含一个唯一的私有成员变量`isa`,可以在`objc.h`中看到对象的定义：

{% highlight objective-c linenos %}

/// A pointer to an instance of a class.


typedef struct objc_object *id;

/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
{% endhighlight %}

对象的`isa`指针(注:在 64 位 CPU 下，`isa` 已经不再是一个简单的指针,具体详情可参考[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html))指向该对象的类，这个类其实是指向`objc_class`的结构体，在`runtime.h` 中可以看到这个结构体的定义：

{% highlight objective-c linenos %}
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY; //

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;//父类
    const char *name                                         OBJC2_UNAVAILABLE;//类名
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;//实例大小
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;//成员变量列表
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;//方法列表
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;//方法缓存
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;//协议列表
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */

{% endhighlight %}

从结构体的定义可以看到类描述了实例对象的特点，如对象占用的内存大小、成员变量列表、成员函数列表、方法缓存、父类的信息等，关于类的方法列表及方法缓存等，将在下一篇文章中进行介绍。  Class也有一个isa指针，指向一个Class，这个isa指向的Class其实就是类的元类(meta class)，同时你可能会有疑问，每一个`objc_class`都有一个指向`objc_class`的isa指针，那总有个`isa`的终点吧？那最终的这个根类的`isa`指针指向什么呢？



#三、类与元类(Meta Class)



上面说到，Class的`isa`指针指向的是这个类的元类，那什么是元类呢？它的作用又是什么？元类的`isa`指向的终点是什么？



带着这个问题，我们先来写一个demo：建立两个类`Son`和`Father`,类`Son`继承自`Father`:

{% highlight objective-c linenos %}

@interface Father : NSObject

@end


@interface Son : Father

@end

{% endhighlight %}


然后，执行下面这段代码：

{% highlight objective-c linenos %}

Son *son = [Son new];

Class class = object_getClass(son);

do {
    
    Class isaClass = object_getClass(class);

    id metaClass = objc_getMetaClass(object_getClassName(class));
    Class superClass = class_getSuperclass(class);
    
    NSMutableString *string = [NSMutableString new];
    
    [string appendFormat:@"Class %@->isa=%@(%p); superClass=%@(%p); metaclass=%@(%p)\n\n", class, isaClass, isaClass, superClass ?: @"nil", superClass, metaClass, metaClass];
    
    Class metaClassSuper = class_getSuperclass(metaClass);
    id metaClassIsa = object_getClass(metaClass);
    
    [string appendFormat:@"metaclass %@(%p)->isa=%@(%p); superClass=%@(%p)\n", metaClass, metaClass, metaClassIsa, metaClassIsa, metaClassSuper, metaClassSuper];
    
    NSLog(@"%@-------------------------------------\n", string);
    
    class = superClass;
    
}while (class);

{% endhighlight %}



最终的运行结果是：

{% highlight objective-c linenos %}

2016-02-08 16:11:54.114 ObjcRuntimeDemo[4289:107651] Class Son->isa=Son(0x105d90eb8); superClass=Father(0x105d90f30); metaclass=Son(0x105d90eb8)

metaclass Son(0x105d90eb8)->isa=NSObject(0x1065ec198); superClass=Father(0x105d90f08)
 -------------------------------------
2016-02-08 16:11:54.115 ObjcRuntimeDemo[4289:107651] Class Father->isa=Father(0x105d90f08); superClass=NSObject(0x1065ec170); metaclass=Father(0x105d90f08)

metaclass Father(0x105d90f08)->isa=NSObject(0x1065ec198); superClass=NSObject(0x1065ec198)
 -------------------------------------
2016-02-08 16:11:54.115 ObjcRuntimeDemo[4289:107651] Class NSObject->isa=NSObject(0x1065ec198); superClass=nil(0x0); metaclass=NSObject(0x1065ec198)

metaclass NSObject(0x1065ec198)->isa=NSObject(0x1065ec198); superClass=NSObject(0x1065ec170)
 -------------------------------------
{% endhighlight %}

直接看结果可能不太直观，将上述结果绘制成对象模型图：

![](/assets/posts/objective-c_class_model_0.jpg)



从图中可以看出：一个实例对象的`isa`指向对象所属的类，这个类的`isa`指向这个类的元类，而这个元类的`isa`又指向NSObject的元类，NSObject的元类的`isa`指向其本身，最终形成形成一个闭环。

在OC中，每一个对象都是类的一个实例，对象的isa指针指向他所属的类，而类本身其实也是一个对象,继承自`objc_object`，这一点从`objc-runtime-new.h`中可以看到：



{% highlight objective-c linenos %}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
......

{% endhighlight %}



当你调用一个类的类方法，如`[NSObject alloc]`其实是给类对象发送了一个消息, 因为类是一个对象，所以他必须是另外一个类的实例，这个类就是元类，每一个类都有一个isa指针指向这个类所属的类，类的isa指针指向的类就是这个类的元类， 类方法其实是类的元类的实例方法，元类为类对象描述了类方法，就像类为类的实例对象描述实例方法一样， 这一点从runtime的源码`objc-class.mm`文件中可以看到：



{% highlight objective-c linenos %}

Method class_getClassMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;
    //类的类方法其实是类的元类的实例方法
    return class_getInstanceMethod(cls->getMeta(), sel);
}

{% endhighlight %}



#四、总结

1. Objective-C中的对象是对C的结构体的封装

2. 所有的对象都有一个`isa`指针，指向对象所属的类，类也是一个对象，类对象的`isa`指针指向类的元类

3. 类描述了对象的特点，如成员变量、方法列表等，元类描述了类的特点，如类的类方法等



参考链接

- [Apple ObjCRuntimeGuide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)
- [http://opensource.apple.com/tarballs/objc4/]()
- [http://blog.devtang.com/2013/10/15/objective-c-object-model/]()
- [http://blog.leichunfeng.com/blog/2015/04/25/objective-c-object-model/]()


