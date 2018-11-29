---
title: Apple 平台下的 VideoToolBox 框架
date: 2018-03-24 13:17:16
categories: [音视频/WebRTC]
tags: [h264, VideoToolBox]
---

`VideoToolBox` 是一个低级的框架，可直接访问硬件的编解码器。它能够为视频提供压缩和解压缩的服务，同时也提供存储在 `CoreVideo` 像素缓冲区的图像进行格式的转换。该框架在 `iOS(8.0)/macOS(10.8)/tvOS(10.2)` 以后才被开放出来，在这之前都是不能使用的。

`VideoToolBox` 框架为我们提供的接口都是以 C 语言函数形式提供的，并且以会话 `Session` 方式来进行管理，这些接口分为以下几个类别：

1. VTCompressionSession：提供数据压缩的会话
2. VTDecompressionSession：提供数据解压缩的会话
3. VTPixelTransferSession：提供数据格式转换的会话
4. VTMultiPassStorage&VTFrameSilo：提供多通道压缩的会话
5. VTSession&Utilities：针对会话的管理以及辅助的函数

这里主要针对 VTCompressionSession 以及 VTDecompressionSession 两个方面的介绍。由于 `VideoToolBox` 可进行多种格式的压缩以及解压缩，下文都是以 H264 编码格式进行介绍和举例。
<!--more-->
### VTCompressionSession 编码器

编码器的使用主要有以下四个流程：

1. 使用 *`VTCompressionSessionCreate`* 函数来创建编码器的 session。
2. 使用 *`VTSessionSetProperty`* 方法来对编码器进行配置。
3. 使用 *`VTCompressionSessionEncodeFrame`* 方法来进行数据的编码，并在回调函数中进行编码后的数据处理。
4. 使用 *`VTCompressionSessionCompleteFrames`* 来完成数据的编码。
5. 最后使用 *` VTCompressionSessionInvalidate`* 以及 *`CFRelease`* 来释放资源和内存。

#### 编码器的创建

```
OSStatus VTCompressionSessionCreate(
	CFAllocatorRef allocator, 
	int32_t width, int32_t height, 
	CMVideoCodecType codecType, 
	CFDictionaryRef encoderSpecification, 
	CFDictionaryRef sourceImageBufferAttributes, 
	CFAllocatorRef compressedDataAllocator, 
	VTCompressionOutputCallback outputCallback, 
	void *outputCallbackRefCon, 
	VTCompressionSessionRef  _Nullable *compressionSessionOut);
```

参数介绍：
* CFAllocatorRef allocator：内存的构造器，默认传递 NULL 即可
* int32_t width, int32_t height：指定编码器的宽高，传入源数据的宽高即可
* CMVideoCodecType codecType： 编码器的类型，H264 使用 `kCMVideoCodecType_H264` 或者 `avc1` 字符
* CFDictionaryRef encoderSpecification：编码规范，传递 NULL 让 `VideoToolbox` 选择
* CFDictionaryRef sourceImageBufferAttributes：设置源图像缓冲区属性
* CFAllocatorRef compressedDataAllocator：指定一个编码后数据的构造器，传递 NULL 即可
* VTCompressionOutputCallback outputCallback：数据编码后的回调接口
* void *outputCallbackRefCon：回调函数的自定义参数
* VTCompressionSessionRef  _Nullable *compressionSessionOut：输出参数，构造好的编码器 Session

#### 编码器的参数配置

编码器的参数是多维度的，都是可以单独的配置，使用下面的函数进行编码器的参数配置：

```
OSStatus VTSessionSetProperty(
	VTSessionRef session, 
	CFStringRef propertyKey, 
	CFTypeRef propertyValue);
```

参数介绍：
* VTSessionRef session：编码器
* CFStringRef propertyKey：属性字段的 Key
* CFTypeRef propertyValue：属性的值

可配置的属性有多种类型如下：

##### 流配置

1. kVTCompressionPropertyKey_Depth：编码器颜色的像素深度，例如 16 位 RGB 或者 24 位 RGB
2. kVTCompressionPropertyKey_ProfileLevel：编码比特流的配置文件和级别
3. kVTCompressionPropertyKey_H264EntropyMode：H.264 压缩的熵编码模式，可以设置为 CAVLC 或者 CABAC

> 更改默认熵模式可能导致与请求的配置文件和级别不兼容的配置。在这种情况下的结果是未定义的，可能包括编码错误或不合规的输出流。

##### 缓冲区配置

1. kVTCompressionPropertyKey_NumberOfPendingFrames：压缩会话中待处理帧的数量
2. kVTCompressionPropertyKey_PixelBufferPoolIsShared：指示公共像素缓冲池是否在视频编码器和会话客户端之间共享
3. kVTCompressionPropertyKey_VideoEncoderPixelBufferAttributes

##### 预期值配置

1. kVTCompressionPropertyKey_ExpectedDuration：压缩会话的预期总持续时间（如果已知）
2. kVTCompressionPropertyKey_ExpectedFrameRate：预期的帧速率，如果知道的话
3. kVTCompressionPropertyKey_SourceFrameCount：源 frame 的数量，如果已知

##### 帧相关

1. kVTCompressionPropertyKey_AllowFrameReordering：指示是否启用了帧重新排序
2. kVTCompressionPropertyKey_AllowTemporalCompression：指示是否启用时间压缩
3. kVTCompressionPropertyKey_MaxKeyFrameInterval：关键帧之间的最大间隔，也称为关键帧速率。
4. kVTCompressionPropertyKey_MaxKeyFrameIntervalDuration：从几个关键帧到下一个关键帧的最长持续时间
5. kVTEncodeFrameOptionKey_ForceKeyFrame：指示当前帧是否被强制为关键帧

##### 硬件加速

1. kVTCompressionPropertyKey_UsingHardwareAcceleratedVideoEncoder：指示是否使用硬件加速视频编码器
2. kVTVideoEncoderSpecification_RequireHardwareAcceleratedVideoEncoder：指示是否需要硬件加速编码
3. kVTVideoEncoderSpecification_EnableHardwareAcceleratedVideoEncoder：指示是否允许硬件加速视频编码

##### 码率控制

1. kVTCompressionPropertyKey_AverageBitRate：期望的平均比特率，以比特/秒为单位
2. kVTCompressionPropertyKey_DataRateLimits：
3. kVTCompressionPropertyKey_MoreFramesAfterEnd：指示压缩会话的帧是否以及如何与其他压缩帧连接以形成更长的序列
4. kVTCompressionPropertyKey_MoreFramesBeforeStart：指示压缩会话的帧是否以及如何与其他压缩帧连接以形成更长的序列
5. kVTCompressionPropertyKey_Quality：所需的压缩质量，该值应该被指定为 0.0 到 1.0 的范围，其中 low = 0.25，normal = 0.50，high = 0.7 5和 1.0 表示支持编码器的无损压缩

##### 运行时

1. kVTCompressionPropertyKey_RealTime：表示是否建议视频编码器实时执行压缩
2. kVTCompressionPropertyKey_MaxH264SliceBytes：H.264编码的最大片大小
3. kVTCompressionPropertyKey_MaxFrameDelayCount：在压缩器必须输出压缩帧之前允许压缩器保持的最大帧数

以上参数并不是都要设置，保持默认值即可。

#### 数据的编码

在进行数据的编码之前，可手动调用下面的方法来申请必要的资源，如果不手动调用，则会在第一次进行数据编码时自动调用：

```
OSStatus VTCompressionSessionPrepareToEncodeFrames(VTCompressionSessionRef session);
```

该函数调用一次之后，后续的调用将是无效的调用。使用一下方法来进行数据的编码：

```
OSStatus VTCompressionSessionEncodeFrame(
	VTCompressionSessionRef session, 
	CVImageBufferRef imageBuffer, 
	CMTime presentationTimeStamp, 
	CMTime duration, 
	CFDictionaryRef frameProperties, 
	void *sourceFrameRefCon, 
	VTEncodeInfoFlags *infoFlagsOut);
```

参数解析：

* VTCompressionSessionRef session：编码器的 session
* CVImageBufferRef imageBuffer：待编码的数据
* CMTime presentationTimeStamp：待编码数据时间戳。当前时间戳必须大于前一个
* CMTime duration：待编码数据的持续时间（画面的持续时间），可传入 `kCMTimeInvalid`
* CFDictionaryRef frameProperties：指定编码当前帧属性的键/值对。一些属性将会影响后续的编码帧
* void *sourceFrameRefCon：传递给回调函数的自定义数据
* VTEncodeInfoFlags *infoFlagsOut：

当需要主动停止编码时，可调用下面方法来强制停止编码器：

```
OSStatus VTCompressionSessionCompleteFrames(
	VTCompressionSessionRef session, 
	CMTime completeUntilPresentationTimeStamp);
```

#### 编码器资源释放与销毁

编码器的销毁需要调用一下两个方法来进行

```
void VTCompressionSessionInvalidate(VTCompressionSessionRef session);
```

该方法用来释放编码器占用的内存资源，在调用一下方法来释放 session 本身的内存资源：

```
void CFRelease(CFTypeRef cf);
```

### VTDecompressionSession 解码器

解码器的使用与编码器类似，大致流程如下：

1. 使用 *`VTDecompressionSessionCreate`* 方法来创一个解码器。
2. 使用 *`VTSessionSetProperty`* 或者 *`VTSessionSetProperties`* 来对解码器进行配置。
3. 使用 *`VTDecompressionSessionDecodeFrame`* 来进行数据的解码。
4. 使用 *` VTDecompressionSessionInvalidate`* 以及 *`CFRelease`* 来对资源的释放。

#### 解码器的创建

```
OSStatus VTDecompressionSessionCreate(
	CFAllocatorRef allocator, 
	CMVideoFormatDescriptionRef videoFormatDescription, 
	CFDictionaryRef videoDecoderSpecification, 
	CFDictionaryRef destinationImageBufferAttributes, 
	const VTDecompressionOutputCallbackRecord *outputCallback, 
	VTDecompressionSessionRef  _Nullable *decompressionSessionOut);
```

参数介绍：

* CFAllocatorRef allocator：构造器，传 NULL 即可
* CMVideoFormatDescriptionRef videoFormatDescription：源数据的格式设置
* CFDictionaryRef videoDecoderSpecification：解码器的规范，可使用 NULL 让 `VideoToolbox` 自动选择
* CFDictionaryRef destinationImageBufferAttributes：缓冲区配置，NULL 保存默认即可
* const VTDecompressionOutputCallbackRecord *outputCallback：用来配置解码后的回调函数以及自定义数据
* VTDecompressionSessionRef  _Nullable *decompressionSessionOut：输出参数用来存储解码器的 session

#### 解码器的参数配置

```
OSStatus VTSessionSetProperty(
	VTSessionRef session, 
	CFStringRef propertyKey, 
	CFTypeRef propertyValue);
```

参数介绍：
* VTSessionRef session：编码器
* CFStringRef propertyKey：属性字段的 Key
* CFTypeRef propertyValue：属性的值

可配置的选项：

##### 解码器行为

1. kVTDecompressionPropertyKey_RealTime：设置解码器是否实时解码数据

#### 数据的解码

```
OSStatus VTDecompressionSessionDecodeFrame(
	VTDecompressionSessionRef session, 
	CMSampleBufferRef sampleBuffer, 
	VTDecodeFrameFlags decodeFlags, 
	void *sourceFrameRefCon, 
	VTDecodeInfoFlags *infoFlagsOut);
```

参数介绍：

* VTDecompressionSessionRef session：解码器的 session
* CMSampleBufferRef sampleBuffer：包含一个或者多个待解码帧的 buffer
* VTDecodeFrameFlags decodeFlags：设置标记为，来影响解码器的解码行为，可传 0
* void *sourceFrameRefCon：传递给回调函数的自定义数据
* VTDecodeInfoFlags *infoFlagsOut：输出参数，解码的一些标志位，可传 NULL

`VTDecodeFrameFlags` 与 `VTDecodeInfoFlags` 的定义如下：

```
typedef CF_OPTIONS(uint32_t, VTDecodeFrameFlags) {
	kVTDecodeFrame_EnableAsynchronousDecompression = 1<<0,
	kVTDecodeFrame_DoNotOutputFrame = 1<<1,
	kVTDecodeFrame_1xRealTimePlayback = 1<<2, 
	kVTDecodeFrame_EnableTemporalProcessing = 1<<3,
};

typedef CF_OPTIONS(UInt32, VTDecodeInfoFlags) {
	kVTDecodeInfo_Asynchronous = 1UL << 0,
	kVTDecodeInfo_FrameDropped = 1UL << 1,
	kVTDecodeInfo_ImageBufferModifiable = 1UL << 2,
};
```

#### 解码器的资源释放以及销毁

解码器的资源释放与销毁也编码器类似，依次调用下面两个方法：

```
void VTDecompressionSessionInvalidate(VTDecompressionSessionRef session);
void CFRelease(CFTypeRef cf);
```

### 基本数据类型

* CVPixelBuffer：编码前和解码后的图像数据结构。
* CMBlockBuffer：编码后 NALU 数据结构。
* pixelBufferAttributes：字典设置.可能包括Width/height、pixel format type、• Compatibility (e.g., OpenGL ES, Core Animation)
* CMVideoFormatDescription：图像存储方式，编解码器等格式描述。
* CMSampleBuffer：存放编解码前后的视频图像的容器数据结构。
* CMTime：时间戳相关。时间以 64-bit/32-bit 的形式出现。

![Alt text](/uploads/CMSampleBuffer.png)

#### CVPixelBuffer

`CVPixelBufferRef` 是 `Core Video` 框架中定义的数据类型，用来描述一张图片的内存空间，它的定义可以从 [API 文档](https://developer.apple.com/documentation/corevideo/cvpixelbuffer?language=objc) 或者头文件中可以查找到定义如下：

```
typedef CVImageBufferRef CVPixelBufferRef;
typedef CVBufferRef CVImageBufferRef;
```

而 `CVBufferRef` 本质上就是一个 `void *` 的指针，至于 `void *` 具体是什么样的数据类型，则只有系统才知道，我们则不需要关心。`CVPixelBufferRef` 描述了一张图片的内存空间，对于该图片的属性可以通过对应的 C 语言函数来获得，例如图片的宽、高、图片的格式通过以下函数来获取：

```
size_t CVPixelBufferGetWidth(CVPixelBufferRef pixelBuffer);
size_t CVPixelBufferGetHeight(CVPixelBufferRef pixelBuffer);
OSType CVPixelBufferGetPixelFormatType(CVPixelBufferRef pixelBuffer);
```

`CVPixelBufferRef`中的图片格式可以有 RGB 或者 YUV 的格式，这两种图片格式都可以有多种模式，对于 RGB 以及 YUV 可以参考之前的文章。

对于 YUV 格式的处理，可以先通过 *`Boolean CVPixelBufferIsPlanar(CVPixelBufferRef pixelBuffer);`* 函数来判断是否为 `Planar` 模式。若是 `Planar`，则通过 *`size_t CVPixelBufferGetPlaneCount(CVPixelBufferRef pixelBuffer);`* 获取 `YUV Plane` 数量，然后由 *`void * CVPixelBufferGetBaseAddressOfPlane(CVPixelBufferRef pixelBuffer, size_t planeIndex);`* 得到相应的通道的存储地址。在进行编码时从摄像头采集的数据就已经是 `CVPixelBufferRef` 格式的数据了，只需要将该数据传入编码器即可。对于解码器输出的就是 `YUV` 格式的 `CVPixelBufferRef` 的数据。对于后续的渲染可以将 `YUV` 转成 `RGB` 模式进行渲染，也可以将 `CVPixelBufferRef` 转成 `OpenGL` 所需要的 `texture` 然后进行渲染。也可以转成 `CIImage` 或者 `UIImage` 对象进行处理。

##### CMBlockBuffer

与 `CVPixelBufferRef` 类似 `CMBlockBufferRef` 也是对内存块的封装描述，在 H264 编码器中用来存储编码后的图像数据，通常是一系列的 nalu 数据。`CMBlockBufferRef` 管理的内存可以是不连续的内存块，也可能是对其他内存的引用，而 `CMBlockBufferRef` 隐藏了这些细节，通过对应的函数来对内存进行操作，比如追加内存，获取内存数据等。相关的操作函数可查看 [API 文档](https://developer.apple.com/documentation/coremedia/cmblockbuffer?language=objc)。

##### CMSampleBuffer

编解码前后的视频图像均封装在 `CMSampleBuffer` 中，如果是编码后的图像，以 `CMBlockBuffe` 方式存储；解码后的图像，以 `CVPixelBuffer` 存储。`CMSampleBuffer` 里面还有另外的时间信息 `CMTime` 和视频描述信息 `CMVideoFormatDesc`

![Alt text](/uploads/CMSampleBuffer.png)

### 参考资料

* [Apple Developer](https://developer.apple.com/documentation/videotoolbox)
* [iOS音视频底层开发与FFmpeg](http://www.jianshu.com/u/c68741efc396)
* [VideoToolbox解析](http://www.jianshu.com/p/6dfe49b5dab8)
* [Direct Access to Video Encoding and Decoding](https://developer.apple.com/videos/play/wwdc2014/513/)
* [CVImageBufferRef中YUV图像](http://blog.csdn.net/kasupermen/article/details/52297138)
* [深入理解 CVPixelBufferRef](https://zhuanlan.zhihu.com/p/24762605?utm_medium=social&utm_source=weibo)
* [在iOS端使用AVSampleBufferDisplayLayer进行视频渲染](http://blog.csdn.net/fernandowei/article/details/52179631)
* [iOS VideoToolbox硬编H.265（HEVC）H.264（AVC）：1 概述](http://www.cnblogs.com/isItOk/p/5964647.html)