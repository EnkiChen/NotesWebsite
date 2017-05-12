---
title: iOS 运行时之消息转发机制
date: 2017-04-21 14:49:54
categories: [知识整理/总结]
tags: [运行时, 消息转发]
---

今天写一篇老生常谈的话题 —— Objective-C 的消息转发机制。Objective-C 下所有的方法调用都可以理解为，给一个对象发送一个消息。一个对象接收到消息后，会从当前类的方法列表或者父类的方法列表查找到对应的方法实现（IMP）来处理该消息。大致流程如下：

1. 通过 `NSObject` 的 *isa* 指针找到对应的 Class
2. 在 Class 的方法列表中找到对应的 *selector*
3. 如果在当前 Class 中未能找到 *selector* 则往父类的方法列表中继续查找
4. 如果能找到对应的 *selector* 则去执行对象的方法实现（IMP）

在上述流程中如果不能找对对应的 *selector* 时，这时候就会进入消息转发机制。消息转发机制可分为两个阶段，在这两个阶段中，有 3 次机会来处理之前未能处理 *selector*，越往后所花费的代价将越大，处理的灵活程度也就越高。如下图所示：

![消息转发流程](/uploads/forwardflow.png)
<!--more--> 
### 第一阶段

第一阶段也可称之为 **动态方法解析** 阶段，在该阶段中，可以动态的为类添加一个方法，从而让动态添加的方法来处理之前未能处理的消息。可重写类以下方法： 

```
+ (BOOL)resolveInstanceMethod:(SEL)sel
```

如果是类的静态方法，可重写以下方法：

```
+ (BOOL)resolveClassMethod:(SEL)sel
```

`SEL` 就是未能处理的 *selector*，返回值为 `BOOL` 表示是否增加了新的方法来处理该 *selector*。在当前阶段处理未知 *selector* 的前提是，你已经准备好了新的方法来处理该 *selector*，等着运行时将方法动态添加到类中即可，该阶段一般用来实现 `@dynamic` 属性。

### 第二阶段

如果在第一该阶段中为能处理未知的 *selector*，运行时将进入第二阶段消息的转发，在该阶段中我们可以将未知的 *selector* 转发给其他对象来处理。运行时提供两次机会，来做消息的转发，第一次是重写以下方法：

```
- (id)forwardingTargetForSelector:(SEL)aSelector
```

该方法的 `SEL` 就是未能处理的 *selector*，返回值类型为 `id` 用来指定 *selector* 处理的对象，运行时将会把未能处理的 `SEL` 转发给该对象。该阶段我们可以将 *selector* 转发到类中的其他对象来处理，从而实现 `代理模式`。如果不重写该方法，运行时将把方法调用的所有细节封装到 `NSInvocation` 对象中，进入完整的消息转发机制中，运行时将继续调用一下方法来进行消息的派发：

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
- (void)forwardInvocation:(NSInvocation *)anInvocation
```

方法中的 `NSInvocation` 参数，包含了所有方法调用的细节，包括 `selector/target/参数` 等，重写该方法后我们可以将 `anInvocation` 转发给多个对象来处理该消息。在该阶段我们可以用来实现 "多重继承" 或者多重代理等。

如果在两个阶段都不做任何处理的话，运行时将会把 *selector* 交由 *doesNotRecognizeSelector* 方法来处理，从而抛出异常导致 Crash ，异常信息一般如下：

```
-[*** ***]:unrecognized selector sent to instance 0x*****
```

### 重写 respondsToSelector 方法

Objective-C 中 *respondsToSelector* 方法可以用检查类对象是否能够处理对应的 *selector*，当我们通过消息转发机制来处理 *selector* 时， *respondsToSelector* 并不能按原意正常工作了，这时候需要重写类的 *respondsToSelector* 方法，用来告诉方法调用者对应的 *selector* 是能够被处理的。如果是在 **动态方法解析** 阶段使用 *class_addMethod* 来为类动态添加方法，则不需要重写 *respondsToSelector* 。

### 消息转发特性的应用场景

#### 为 @dynamic 实现方法

使用 `@synthesize` 可以为 `@property` 自动生成 `getter` 和 `setter` 方法（现 Xcode 版本中，会自动生成），而 `@dynamic` 则是告诉编译器，不用生成 `getter` 和 `setter` 方法。当使用 `@dynamic` 时，我们可以使用消息转发机制，来动态添加 `getter` 和 `setter` 方法。当然你也用其他的方法来实现。

#### 代理模式实现

看完 Objective-C 的消息转发机制，相信很多朋友都能想到 **代理模式**。对 **代理模式** 不熟悉或者不明白的应用场景的同学，可以自行去学习一下 **代理模式**。同时 Objective-C 提供了 `NSProxy` 类可以用来做动态代理。

#### 多重继承

学过 C++ 的同学都支持，C++ 是支持多继承的，子类可以从多个类继承，从而获得多个类所有的特性。在消息转发的最后一次处理机会中，运行时会产生一个 `NSInvocation` 对象，进入到完整的消息转发机制中，在该流程中我们将 *selector* 转发到不同的对象中处理，便可以达到 “多重继承” 的特性。做法是在类的构造方法中，构造一个或者多个需要继承的对象，将未能处理的 *selector* 转发到对应的对象中处理即可。

> [样例代码](https://github.com/EnkiChen/MsgForwardingSample.git) 在这里

### 小结

* 当接收无法处理的 *selector* 时，则进入消息转发流程
* 消息转发流程可分为两阶段，一共有 3 次机会来处理未知的 *selector*
* 第一阶段为 **动态方法解析** 阶段，用来为类动态添加方法，第二阶段才是正在的消息转发阶段，该阶段可以将未知的 *selector* 转发到一个或者多个对象中来处理
* 消息转发流程完成后，都不做任何处理的话，这进入 *doesNotRecognizeSelector* 方法从而抛出异常
* 如果将消息转发到其他对象来处理，则需要重写 *respondsToSelector* 方法来保证该方法正常工作
* `NSProxy` 类是基于消息转发机制来实现的动态代理模式
* 消息转发机制可用来实现 `@dynamic` 属性、代理模式、多重继承等
