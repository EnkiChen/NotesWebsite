---
title: Android 对 C++ Runtime 的支持
date: 2017-12-20 20:50:24
categories: [开发笔记]
tags:
---

> 近期将项目中的 webrtc 代码与官方最新的 m63 分支进行了同步，由于之前的代码比较老，这次升级后，webrtc 的代码结构相对来说有了很多的调整和变化，再加上对 webrtc 代码的修改和代码的添加，索性依照 m63 分支的 gn 脚本使用 cmake 重写了 webrtc 的编译脚本。在编译 android 平台时遇到一些 c++ 标准库中的函数无法进行链接的问题，于是了解了一下 android 对 c++ 标准库的支持。

Android 对 C++ 基础库的实现有多个版本，具体的版本可以在 NDK 目录下的 `sources/cxx-stl` 子目录中查看，我当前使用的版本是 `android-ndk-r12b` 子目录内容如下：

<!--more-->

```
drwxr-xr-x  10 Enki  staff   340 12  7 18:05 gabi++
drwxr-xr-x   5 Enki  staff   170 12  7 17:29 gnu-libstdc++
drwxr-xr-x  16 Enki  staff   544 12  7 17:37 llvm-libc++
drwxr-xr-x   8 Enki  staff   272 12  7 17:12 llvm-libc++abi
drwxr-xr-x  16 Enki  staff   544 12  7 18:37 stlport
drwxr-xr-x   9 Enki  staff   306 12  7 20:47 system
```

* gabi++：该库不支持 C++ 标准库，但加入了对异常和 RTTI 的支持。
* gnu-libstdc++：GNU 标准 C++ 运行时库（libstdc++-v3），同样支持异常和 RTTI。
* llvm-libc++：该版本是 [LLVM libc++](http://libcxx.llvm.org/) 的一个移植版本，支持 C++ 的所有特性。
* stlport：该库也提供 C++ 所有的特性支持，它是开源项目 [STLPort](http://www.stlport.org) 的一个 android 移植版本
* system：默认最小的 C++ 运行库，这样生成的应用体积小，内存占用小，但部分功能将无法支持

在 NDK 目录中都有预编译好的静态库和动态库，在实际项目中，只要链接对应的库即可。在使用 `cmake` 脚本的项目中可以使用 `ANDROID_STL` 宏来指定需要链接的库，在使用 `MK` 项目中可以使用 `APP_STL` 宏来指定链接的库例如：
 
```
# cmake
-DANDROID_STL=c++_static

# Application.mk
APP_STL:= c++_static
```

链接库可选值有如下几种：

* stlport_static：stlport 作为静态库
* stlport_shared：stlport 作为动态库，这个可能产生兼容性和部分低版本的 Android 固件
* c++_static：stdlib 作为静态库链接
* c++_shared：stdlib 作为动态库链接
* gnustl_static：使用 GNU libstdc++ 作为静态库
* system：使用默认的最小的 C++ 运行库

在 android 平台下编译 m63 分支时，原来使用的是 `gnustl_static` 版本的标准库，在新的 m63 分支中使用了一些诸如 `std::strtoll()` 的方法（还使用了一些其他方法也 `gnustl_static` 中没有实现的），而在 `gnustl_static` 版本的标准库中，并没有提供该函数的实现，从而导致 android 平台下无法链接 webrtc 库。这里使用了 `c++_static` 替换了 `gnustl_static` 来解决链接的问题。

NDK 中提供了不同版本，不同的版本有不同的特性，有些体积大提供更丰富的特性和标准函数的支持，一些体积小占用空间少，可根据具体的项目需求来选择，随着不 NDK 版本的升级，特性也会有改动。