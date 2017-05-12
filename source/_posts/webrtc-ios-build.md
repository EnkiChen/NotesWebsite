---
title: WebRTC iOS&OSX 库的编译
date: 2017-05-12 13:04:36
categories: 知识整理/总结
tags: [WebRTC]
---

### 安装 Depot_tools

* git 命令获取 depot_tools：

```
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

* 配置坏境变量

```
$ echo "export PATH=$PWD/depot_tools:$PATH" > $HOME/.bash_profile
$ source $HOME/.bash_profile
```

* 检测配置是否成功

```
$ echo $PATH
```

### 安装 ninja

<!--more-->

**ninja** 是 **WebRTC** 的编译工具，我们需要对其进行编译，步骤如下：

```
$ git clone git://github.com/martine/ninja.git
$ cd ninja/
$ ./bootstrap.py
```

复制到系统目录（也可配置坏境变量）

```
$ sudo cp ninja /usr/local/bin/
$ sudo chmod a+rx /usr/local/bin/ninja
```

### 下载源代码

**WebRTC** 源码托管在 [**Google Source**](https://chromium.googlesource.com/external/webrtc) ，在 [**Release Notes**](https://webrtc.org/release-notes/) 中选择需要的版本，这里选择最新的 **M57** 版本

* 设置要编译的平台到环境变量中：

```
$ export GYP_DEFINES="OS=ios"
```

不同机型的编译参数：

```
# 32位真机
$ export GYP_DEFINES="OS=ios target_arch=arm"
# 64位真机
$ export GYP_DEFINES="OS=ios target_arch=arm64"
# 32位模拟器
$ export GYP_DEFINES="OS=ios target_arch=ia32"
# 64位模拟器
$ export GYP_DEFINES="OS=ios target_arch=x64"
# OSX
$ export GYP_DEFINES="OS=mac target_arch=x64"

# 配置输出路径
$ export GYP_GENERATOR_FLAGS="output_dir=out_xxx"
```

* 创建工作路径并执行下面的语句： 

```
$ fetch --nohooks webrtc_ios
$ gclient sync -r 52b6562a10b495cf771d8388ee51990d56074059 --force
```

> 上面的 commit id 是 **M57** 版本最后一次 commit id，可以从 [**Release Notes**](https://webrtc.org/release-notes/) 中找到，可替换成自己所需的版本的 commit id 或者直接使用最新的 commit id

执行上述命令就会去下载对应的 **WebRTC** 的源码、构建工具链以及依赖的第三方库，由于是国外网站，并且是 **Google**，请自备梯子，我这边翻墙后大概 2 个小时就下完了源码。

### 编译库文件

#### iOS 版本的编译

**ninja** 是 **WebRTC** 的编译平台，iOS 版本我们可以使用自带的编译脚本，这样就不需要自己编译和安装 **ninja**，默认情况行，脚本会编译 3 个平台机型的库文件，以及一个各个平台的集合库，脚本也可以指定编译成 `.a` 的库文件或者 `.framework`，命令如下：

```
$ cd src/webrtc/build/ios/
$ ./build_ios_libs.sh
```

不同的版本，编译脚本的路径会有不一样。编译完成后，如果没有指定输出路径，则会在  *`out_ios_libs`* 目录下生成所需要的 **WebRTC.framework**，子目录中会有对应平台单独的 **WebRTC.framework**，根目录下的则支持所有平台，目录结构如下：

```
drwxr-xr-x   3 Enki  staff   102  5  9 16:49 WebRTC.dSYM/		[符号表文件]
drwxr-xr-x   6 Enki  staff   204  5  9 16:49 WebRTC.framework/	[支持所有平台]
drwx------  14 Enki  staff   476  5  9 16:39 arm64_libs/		[真机 64 位]
drwx------  14 Enki  staff   476  5  9 16:30 arm_libs/			[真机 32 位]
drwx------  14 Enki  staff   476  5  9 16:49 x64_libs/			[模拟器 64 位]
```

#### Mac 版本的编译

下载源码：

```
$ export GYP_DEFINES="OS=mac"
$ fetch --nohooks webrtc_ios
$ gclient sync -r 52b6562a10b495cf771d8388ee51990d56074059 --force
```

Mac 没有现成的编译脚本，我们只能通过 `gn` 来生成对应的 ninja 编译脚本，然后通过 ninja 脚本来编译，用如下命令来生成对应的 ninja 项目文件：

```
$ gn gen out/mac_x64 --args='target_os="mac" target_cpu="x64" is_component_build=false'
```

> 可以添加 --ide=xcode 参数来生成 Xcode 的项目文件，使用 Xcode 来进行编译。

编译源码：

```
$ ninja -C out/mac_x64 rtc_sdk_objc
```

上述命令 `out/mac_x64` 则是 ninja 项目文件的路径，`rtc_sdk_objc` 则是要编译的目标，可以通过以下命令来查看目标列表：

```
$ gn ls out/mac_x64
```

### 其他平台源码下载

**Windows**

```
$ fetch --nohooks webrtc
$ gclient sync
```

**Linux**

```
$ export GYP_DEFINES="OS=linux"
$ fetch --nohooks webrtc_android
$ gclient sync
```
Windows 和 Linux 都使用 `./build/install-build-deps.sh` 脚本进行编译即可。

**Android**

```
$ export GYP_DEFINES="OS=android"
$ fetch --nohooks webrtc_android
$ gclient sync
```
Android 使用 `./build/install-build-deps-android.sh` 脚本进行编译。

### 生成 Example 并运行

执行以下命令，用于生成 ninja 的编译脚本 ：

```
$ gn gen out/mac_x64 --args='target_os="mac" target_cpu="x64" is_component_build=false'
```

用以下命令可用来编译并生成可执行的二进制文件：

```
$ ninja -C out/mac_x64 AppRTCMobile
```
执行之后可在 *`out/mac_x64`* 目录生成可执行的 `AppRTCMobile` 文件，双击即可运行。效果如下图：

![Example](/uploads/AppRTCMobile.png)

通过修改 `--args='target_os="mac" target_cpu="x64"` 参数来生成其他平台的 Example。

### 参考资料

 * [WebRTC 官网](https://webrtc.org)
 * [WebRTC(iOS)下载编译(下载指定版本)](http://www.cnblogs.com/fulianga/p/5868823.html)