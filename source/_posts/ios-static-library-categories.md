---
title: iOS 静态库中的 Category 运行时错误
date: 2017-09-13 11:49:24
categories: [开发笔记]
tags: [Category]
---

最近在弄 iOS 下的音视频 SDK 的移植和适配，该 SDK 是基于 WebRTC 项目但并未使用官方的 `ninja` 编译脚本，而是使用的 `cmake` 作为编译工具。在 WebRTC 的音频模块中引用了一个 `UIDevice` 的 `Category` 来做设备类型的判断，编译和链接都没有出现问题，但在运行时出现了 `selector not recognized` 的异常。该异常可以从我之前的  [《iOS 运行时之消息转发机制》](http://www.enkichen.com/2017/04/21/ios-message-forwarding/) 文章中了解到，由于对象接收到了一个无法处理该 `selector` ，经过消息转发后还未得到处理，会在 `doesNotRecognizeSelector` 方法中抛出的异常。

`Category` 以及一些其他的工具类被编译在一个基础的静态库中，在音频模块中引用该静态库，除了 `Category` 代码其他所有的代码都能正常编译、链接以及运行，但唯独 `Category` 在运行时出现了错误，由于 `Category` 的 Objective-C 语言的特性，最开始我以为需要为编译器添加针对 `Category` 的参数，找了很久也没找到针对 `Category` 特性的编译参数，无奈之下只好求助 `stackoverflow` ，最终找到了根本原因和解决方法。
<!--more--> 
### 编译/链接过程

编译器在编译的过程中首先会将 `.c/.cc/.cpp/.m/.mm` 的源文件编译成后缀为 `.o` 的对象文件 ，源文件与对象文件是一一对应的，对象文件中包含了符号、代码以及数据，对象文件是不能被系统加载并使用的。当在编译生成静态库时，这些所有的对象文件，都会被封装到 `.a(archive)` 文件，可以理解为一个归档文件，也就是我们平常使用的静态库。

当需要生产二进制文件或是动态库时，编译器会对静态库 `.a(archive)` 文件进行处理；编译器将获取静态库中所以的符号表，并检查哪些符号被引用，只有被引用的对象文件才会被链接器真正的加载并处理。例如在静态库中有 10 个对象文件，但被引用到只有 2 个，则链接器只加载被引用的 2 个对象文件，未被使用到的 8 个对象文件则会被忽略。

在 C/C++ 语言中，这种机制可以很好的工作，因为 C/C++ 语言会尽可能的在编译期去做这些事。在 Objective-C 语言中非常依赖运行时特性，`Category` 就是基于运行时实现，但它并不会像类或者函数一样被创建链接符号，编译器在检查符号表时便不能检查到 `Category` 对应的符号表，从而 `Category` 的对象文件就不能被正常的加载，在运行时便无法找到对应的 `selector` 。

### 解决方案

基于上述的原因，网上存在 5 种解决方法：

1. 在编译选项 `Other Linker Flags` 中添加 `-all_load`，用于会告诉编译器 **对于所有静态库中的所有对象文件，不管里面的符号有没有被用到，全部都载入**，这种方法可以解决问题，但是会产生比较大的二进制文件。
2. 添加编译选项 `-force_load` 并指定路径，`-force_load $(BUILT_PRODUCTS_DIR)/<library_name.a>` 这种方法和 `-all_load` 类似，不同的是它只载入指定的静态库。
3. 在编译选项 `Other Linker Flags` 中添加 `-ObjC`，这个标识告诉编译器 **如果在静态库的对象文件中发现了 Objective-C 代码，就把它载入**，`Category`  中肯定会存在 Objective-C 代码。该方法与前两张类似，只是将加载的范围减少了。
4. 另一种解决方法是新版本 Xcode 里 `build setting` 中的 `Perform Single-Object PreLink`，如果启用这个选项，所有的对象文件都会被合并成一个单文件（这不是真正的链接，所以叫做预链接），这个对象文件（有时被称做主对象文件 `master object file`）被添加到静态库中。现在如果主对象文件中的任何符号被认为是在使用，整个主对象文件都会被认为在使用，这样它里面的 Objective-C  部分就会被载入了，当然也包括 `Category` 对应的对象文件。
5. 最后一种解决方法是在 `Category` 的源文件里添加 `Fake symbol`，并确保以某种方法在编译时引用了该 `Fake symbol`，这会使得 `Fake symbol` 对象文件被加载时它里面 `Category`  代码也会被载入。该方法可以控制哪些 `Category` 可以被正常加载，同时也不需要添加编译参数做特殊处理。

> 在 64 位的 Mac 系统或者 iOS 系统下，链接器有一个 bug，会导致只包含有 Category 的静态库无法使用 -ObjC 标志来加载  Objective-C 对象文件。

### 问题的处理

我这里用的是最后一种方法来解决该问题，原因在于前 4 种都会增加二进制文件的体积，在第三方使用当前 SDK 时需要手动设置编译参数，会给第三方带来不好的使用体验。为了使用方便可定义一下宏：

```
#define FIX_CATEGORY_BUG_H(name) \
@interface FIX_CATEGORY_BUG_##name : NSObject \
+(void)print; \
@end

#define FIX_CATEGORY_BUG_M(name) \
@implementation FIX_CATEGORY_BUG_##name \
+ (void)print {} \
@end

#define ENABLE_CATEGORY(name) [FIX_CATEGORY_BUG_##name print]
```

在 `Category` 的头文件中使用 `FIX_CATEGORY_BUG_H()` 宏来声明一个 `Fake symbol` ，在 `Category` 的实现文件中使用 `FIX_CATEGORY_BUG_M()` 宏来实现该 `Fake symbol`。最后在找一处运行 `ENABLE_CATEGORY()` 宏，可以是初始化方法中，也可以是其他任何地方，只要确保它能被正常调用，目的在于该 `Fake symbol` 确保编译器能正常加载它。

### 总结

在 iOS/Mac 平台下，包含 `Category` 的静态库无法被正常加载，原因在于 `Category` 是 Objective-C 语言的特性，编译器并不会为它生成链接符号，在链接过程中便无法找到该对象文件的引用关系，链接器将会直接忽略掉 `Category`  对应的对象文件，从而在运行时无法找到相应的 `selector`。解决该问题的目标就是让链接器加载 `Category` 对应的对象文件，一种方法是添加编译参数让编译器加载所有的对象文件或是加载指定的对象文件；另一种方法是在 `Category`  的对象文件中添加 `Fake symbol` ，当 `Fake symbol` 被加载时 `Category` 的对象文件便一同被加载。

### 参考资料

* [Objective-C categories in static library](https://stackoverflow.com/questions/2567498/objective-c-categories-in-static-library)
* [创建含有category的静态库,selector not recognized的解决方案](http://www.dreamingwish.com/frontui/article/default/the-create-the-static-the-library-containing-the-category.html)
