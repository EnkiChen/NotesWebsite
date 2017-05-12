title: Protocol Buffer 简介与使用
date: 2017-02-17 14:58:27
categories: 知识整理/总结
tags: [Protobuf]
---

Protocol Buffer(简称Protobuf或PB)是由 Google 开发的，原本用来解决索引服务器的请求、响应协议，后面才对外使用和开源的，它是一种数据交换格式，与 XML 和 JSON 相比 ，它是一种二进制格式，避免了各种文本格式转换的问题，并且更体积更小、速度更快使用更简单方便，还自带部分数据压缩功能，在网络传输时可以减少数据流量。同时它也是与平台和语言无关的，尤其在网络数据交换方面，使得它越来越流行。

Protobuf 是一个开源项目，项目托管在 GitHub 上，链接为:[https://github.com/google/protobuf](https://github.com/google/protobuf)，源码包含两部分内容：

* **PB基础库**：用来完成对象模型与数据模型之前的转换
* **PB编译器**：源码生成器，用来将 *.proto* 文件转换成对应语言的对象模型的源码

Protobuf 截止目前最新版本为 `3.2.0-alpha-1` 版本，在 `3.0.0` 版本之前也就是 `2.6.1` 版本时官方只支持 `C++/Java/Python` 三种语言，`3.0.0` 版本之后才逐步支持其他语言。在这之前如果其他语言需要用到 Protobuf 都是通过第三方扩展来实现的，目前官方以及支持以下编程语言：

<!--more-->

| Language            | Source             |
|---------------------|--------------------|
| C++ (include C++ runtime and protoc) | [src](https://github.com/google/protobuf/blob/master/src)   |
| Java                                 | [java](https://github.com/google/protobuf/blob/master/java)   |
| Python                               | [python](https://github.com/google/protobuf/blob/master/python)  |
| Objective-C                          | [objectivec](https://github.com/google/protobuf/blob/master/objectivec)  |
| C#                                   | [csharp](https://github.com/google/protobuf/blob/master/csharp)  |
| JavaNano                             | [javanano](https://github.com/google/protobuf/blob/master/javanano)  |
| JavaScript                           | [js](https://github.com/google/protobuf/blob/master/js)  |
| Ruby                                 | [ruby](https://github.com/google/protobuf/blob/master/ruby) |
| Go                                   | [golang/protobuf](https://github.com/golang/protobuf) |
| PHP                                  | [php](https://github.com/google/protobuf/blob/master/php)  |

> 备注：后续将使用最新版本`3.2.0-alpha-1 `以及 Objective-C 语言作为示例

### Protobuf 编译器

Protobuf 编译器在于将 *.proto* 文件转成对应语言的对象模型的源代码，可以从 GitHub 上下载源代码在本地进行编译，然后生成 Protobuf 编译器，也可以从 GitHub 上下载已经编译好的编译器。

下载链接为：[https://github.com/google/protobuf/releases](https://github.com/google/protobuf/releases)

如果是源码的话需要确保一下工具已经安装：

```
autoconf automake libtool curl make g++ unzip
```

执行以下几个步骤进行编译和安装：

```
# 生成 configure 文件
$ ./autogen.sh

# 配置 makefile
$ ./configure

# 编译
$ make
$ make check

# 安装编译后的二进制文件
$ sudo make install

```

> 如果出现 `/usr/local/Cellar/../Library/ENV/4.3/sed: No such file or directory` 错误可以尝试使用 `brew reinstall libtool` 重新安装 libtool

如果编译成功，就可以使用 `proto` 命令了，如果是直接下载的 `proto` 文件，就不需要编译可直接运行。我们可以创建一个 *.proto* 文件来测试一下环境是否搭建好，样例文件如下：

```
message User
{
  required int32 uid = 1;        // 用户唯一标识
  required string nickname = 2;  // 用户昵称
  required int32 age = 3;        // 用户年龄
}
```

如果需要生成 Objective-C 源文件的话，执行以下命令即可：

```
protoc --objc_out=./ ./test.proto
```

如果环境正常的话，可以在当前目录下生成 *Test.pbobjc.h* 和 *Test.pbobjc.m* 两个文件，由于 Protobuf 编译器生成的 Objective-C 源文件是基于 MRC 的，所以如果在 ARC 工程成需要添加 `-fno-objc-arc` 标签，否则无法通过编译。

### Protobuf 基础库

Protobuf 基础库是用来做对象的序列化以及反序列化用的库，也可以从以下地址下载基础库的载源码：

下载链接为：[https://github.com/google/protobuf/releases](https://github.com/google/protobuf/releases)

将下载好的源码拖到对应的工程中即可，同样需要注意的是基础库的源码也是基于 MRC 的，所以也得添加 `-fno-objc-arc` 标签。同时必须保证基础库和编译器的版本一致，否则也会导致工程无法正常编译。

基础库在 iOS 项目中也可以使用 **cocoapods** 方式引入，只要在 `Podfile` 中添加：

```
# 当前工程支持的 SDK 版本
platform :ios, '7.1'
pod 'Protobuf', '~> 3.1.0'

```

### 在项目中使用 Protobuf

将编译器生成的对象的源文件以及基础库导入到工程后，我们就可以使用它们了，示例如下：

```
// 创建一个 User 对象
User *user = [[User alloc] init];
user.uid = 10086;
user.nickname = @"EnkiChen";
user.age = 28;
    
// 序列化为 Data
NSData *data = [user data];
    
// 反序列化为对象
User *us = [User parseFromData:data error:NULL];
NSLog(@"uid:%d nickname:%@ age:%d", us.uid, us.nickname, us.age);

```

### Protobuf 的优缺点

#### 优点

* **性能好/效率高**： 相比于 XML 和 JSON 而言，Protobuf 序列化反序列化速度很快，生成的数据体积更小，适合网络传输
* **有代码生成机制**：可以由编译器自动生成对应语言的类对象文件，无需自己编写解析代码
* **支持向后/向前兼容**：所谓的“向后兼容”（backward compatible），就是说，当模块 B 升级了之后，它能够正确识别模块 A 发出的老版本的协议。 所谓的“向前兼容”（forward compatible），就是说，当模块A升级了之后，模块 B 能够正常识别模块 A 发出的新版本的协议。
* **支持多种语言跨平台**：如上文所提到，官方已经支持多种主流语言，并且跨平台

#### 缺点

* **可读性差**：相对于 XML 和 JSON 而言，对象序列化后生成的数据可读性差（二进制格式）
* **缺乏自描述**：XML 和 JSON 是自描述的，而 Protobuf 格式则不是。一段 Protobuf 格式的二进制数据内容，不配合 *.proto* 文件结构体是看不出来什么作用的。

对于 *.proto* 文件的编写规则及语法等内容就不介绍了，后续有时间再补上。