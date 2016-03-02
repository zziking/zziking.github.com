---
layout: post
title: Objective-C Runtime(二)OC的消息机制
fullview: true
category: iOS
tags: Runtime
keywords: Runtime,Messaging,Forwarding
---



在上一篇文章中，我们探讨了Objective-C的对象模型，在本文中我们来了解OC的一个强大的特性：消息机制，了解消息机制，可以让我们知道OC中调用一个方法会经历哪些过程。



# 消息传递(Messaging)

在C等语言中，调用一个方法就是执行内存中的一段代码，这在编译的时候就决定好了，所以没有动态的特性，而在Objective-C中，方法的调用会被编译成消息的发送，调用某个对象的方法，其实是在运行时给对象发送了一条消息， 例如，下面的代码是等价的：

```objc

[array insertObject:obj atIndex:0];



objc_msgSend(array, @selector(insertObject:atIndex:), foo, 0);

```

那么，runtime又是如何处理`objc_msgSend`发送的消息呢？从发送消息到最终的方法调用，这之间经历了一个怎样的过程？

`objc_msgSend`的源码是使用汇编写的，本人汇编渣，无法从源码角度去分析，不过从详尽的注释和相关的文档，我们可以一窥其面目，`objc_msgSend`会经过以下步骤：



1. 通过对象的`isa`指针找到对象所属的class

2. 在class的方法缓存`(objc_cache)`中查找方法，如果没有找到，则继续3、4步骤

3. 在class的`method_list`中查找方法

4. 如果class中没有找到方法，则继续往它的`super class`中查找, 直到查找到根类

5. 一旦找到方法，就去执行对应的方法实现(IMP)，并把方法添加到方法缓存中



> ps,如果还不了解`isa`和OC的对象模型的，可以去看看我的上一篇文章:[Objective-C Runtime(一)对象模型及类与元类 ](http://zziking.github.io/ios/2016/02/08/Objective-C_Runtime_1_The_object_model.html)



在这个方法查找过程中，`runtime`引入了缓存机制，这是为了提高方法查找的效率，因为，如果调用的方法在根类中，那么每次方法调用都要沿着继承链去每个类的方法列表中查找一遍，效率无疑是低下的。这个方法缓存的实现，其实就是一个哈希表的存储，以`selector name`的哈希值为索引, 存储方法的实现(IMP)，这样的查找效率较高，看到这里，可能有人会有疑问，既然每个`class`维护着一个方法缓存的哈希表，为什么还要维护一个方法列表`method list`呢？每次直接去哈希表里查找方法不是更快吗？

这其实是因为哈希表是一个无序表，而方法列表是一个有序列表，查找方法会顺着`method list`依次查找，这样就赋予了`category`一个特性：可以覆盖原本类的实现，而如果是使用哈希表，则无法保证顺序。关于`category`的原理和具体实现，将在后续的文章中探讨。



# 动态方法决议与消息转发

在上面的查找方法的过程中，如果最终没有查找到目标方法，会导致crash，但是在crash之前，`runtime`会给我们2次机会去挽救程序：



1. Dynamic Method Resolution

2. Message Forwarding



## Dynamic Method Resolution(动态方法决议)

第一个机会就是动态方法决议， 有时候，我们可能希望能动态地为一个方法提供具体实现，例如，我们使用`@dynamic`声明了一个属性：



```objc

@dynamic propertyName;

```

这会告诉编译器这个属性的`getter`和`setter`方法会动态的提供。



我们可以通过实现`+(BOOL)resolveInstanceMethod:` 和`+(BOOL)resolveClassMethod:`方法来动态地为selector生成方法的实现，这两个方法分别为实例方法和类方法提供了动态添加方法的可能。一个Objective-C的方法其实就是一个最少包含两个参数`（self, _cmd）`的C函数,我们可以使用`class_addMethod`来动态地给类添加方法：

```objc

void dynamicMethodIMP(id self, SEL _cmd) {    

   // implementation ....    

}    



+ (BOOL)resolveInstanceMethod:(SEL)aSEL    

{    

   if (aSEL == @selector(resolveThisMethodDynamically)) {    

         class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");    

         return YES;    

   }    

   return [super resolveInstanceMethod:aSEL];    

}    

```

> 注：上面`class_addMethod`的参数`v@:`详见[Type Encodings](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1) 



在这里， 我们在`+(BOOL)resolveInstanceMethod:`中使用`class_addMethod`添加了对应selector`resolveThisMethodDynamically`函数的实现`void dynamicMethodIMP(id self, SEL _cmd)`，并返回YES，`runtime`就会重启一次消息的发送过程，调用动态添加的方法。



如果这个方法返回NO，或者没有动态地添加方法，`runtime`将进入下一步：**消息转发(Message Forwarding)**



## Message Forwarding(消息转发)

消息转发是`runtime`给我们第二次挽救程序的机会，消息转发分为两步：

1. 首先`runtime`会调用`-(id)forwardingTargetForSelector:(SEL)aSelector`方法，为selector寻找一个转发的目标对象，如果这个方法返回的不是nil或self，`runtime`会将消息发送给返回的目标对象，否则继续下一步

2. `runtime`会调用`-（NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`方法来获得一个方法签名，这个方法签名记录了方法的参数和返回值等信息，如果这个方法返回nil, `runtime`会调用`doesNotRecognizeSelector:`,在这里将抛出`unrecognized selector exception`，程序Crash； 如果返回了一个方法签名，`runtime`会创建一个`NSInvocation`对象，并发送`-forwardInvocation`消息给目标对象。



`NSInvocation`实际上就是对一个消息的描述，包括selector和参数等信息, 我们可以利用这个invocation来实现调用其他对象的目标方法：



```objc

- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector    

{    

   NSMethodSignature* signature = [super methodSignatureForSelector:selector];   

   if (!signature) {    

      signature = [someOtherObject methodSignatureForSelector:selector];    

   }    

   return signature;    

}    



- (void)forwardInvocation:(NSInvocation *)anInvocation    

{    

   if ([someOtherObject respondsToSelector:    

           [anInvocation selector]])    

       [anInvocation invokeWithTarget:someOtherObject];    

   else    

       [super forwardInvocation:anInvocation];    

}    

```



消息转发机制是OC非常强大的一个特性，利用它，我们可以实现很多扩展功能，例如，我在[利用OC的动态方法决议与消息转发机制实现多重代理](http://zziking.github.io/ios/2015/11/01/利用OC的动态方法决议与消息转发机制实现多重代理.html)一文中，实现了代理转发功能。还有，我们可以利用这个特性实现AOP(切面编程)，为某个方法的调用设置拦截器，比如，log信息的记录使用AOP来实现可以降低代码的耦合度。 



动态方法决议和消息转发可以用下图来表示：

![](/assets/posts/objective-c_messaging_0.png)



# 总结



Objective-C中给一个对象发送消息会经过以下几个步骤：

1. 在对象的类中查找selector，如果找到了，执行对应函数的IMP,查找过程会使用缓存，并且沿着类的继承关系链查找

2. 如果没有找到，Runtime 会发送`+resolveInstanceMethod:`或者`+resolveClassMethod:`尝试去 resolve 这个消息

3. 如果 resolve 方法返回 NO，Runtime 就发送`-forwardingTargetForSelector:`允许你把这个消息转发给另一个对象

4. 如果没有新的目标对象返回， Runtime 就会发送`-methodSignatureForSelector:`和`-forwardInvocation:`消息。你可以发送`-invokeWithTarget:`消息来手动转发消息或者发送`-doesNotRecognizeSelector:`抛出异常。



# 参考链接

1. [How Messaging Works](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html#//apple_ref/doc/uid/TP40008048-CH104-SW2) 

2. [Dynamic Method Resolution](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtDynamicResolution.html#//apple_ref/doc/uid/TP40008048-CH102-SW1) 

3. [Message Forwarding](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtForwarding.html#//apple_ref/doc/uid/TP40008048-CH105-SW1) 

4. [ Objective-C Runtime](http://tech.glowing.com/cn/objective-c-runtime/) 