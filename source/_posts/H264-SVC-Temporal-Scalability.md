---
title: H264/SVC Temporal Scalability
date: 2018-05-23 17:46:10
categories: [音视频/WebRTC]
tags: [h264]
---

在多人远程会议或直播系统中，参与的用户可能处于不同的网络环境（有线、Wifi、3G、4G）中，网络质量各不一致，为了所有用户可进行远程会议或者直播的观看，简单的做法就是降低发送端的视频码流，这样不管网络质量好坏，参与的用户都将观看低码率的视频流。这种方案缺点在于大部分网络较好的用户会被少数的网络较差的用户给拖累。这里介绍 H264 编码器中的 `Temporal Scalability` 机制来优化该方案。

`Temporal Scalability` 是 H264/SVC 编码器中的概念，意思为时间上可伸缩的，在实际编码中编码器进行了分层编码，可简单的理解为 **编码器对同一组输入的数据进行编码，可以输出不同帧率的码流**，例如当前编码器输入帧率为 30fps 的流，编码器可同时输出多个码流，例如同时输出 3 层码流，从而可以得到不同帧率的码流。
<!--more-->
> 这里的 3 层码流是有依赖关系，比如输出有 A、B、C 3 层码流，单独的发送 A 层则得到低帧率的码流例如 5fps，如果同时发送 A 和 B 两层码流，则能得到相对较高的码流例如 15fps，如果同时发送 ABC ，则能得到最高的码流例如 30fps。

在具体应用中便可以根据用户的网络质量来分配不同码率的视频流，这样网络质量较好的用户能看到高帧率的视频流，而网络较差的用户则看到低帧率的视频流。如下图所示：

![Alt text](/uploads/SVC_Temporal_Scalability.png)

在 H264/SVC 编码器中进行了分层编码，例如上图描述编码端同时将所有的码流层发送至服务器，如果用户网络质量非常好，服务器将所有层的码流数据转发至该用户例如 Receiver2；如果用户网络非常差，则只需要将低层级码流数据转发给该用户例如 Receiver1。实际项目中可根据接收端带宽情况，转发对应码率的流即可。

### 编码器的支持

Temporal Scalability 机制并不是所有的编码都支持的，Apple 平台下（包括iOS和MacOS）的硬件编解码框架 `VideoToolbox` 是不支持的该特性的，因为没有看到任何的可设置的参数或者 API 接口。Android 平台下硬件编解码框架 `MediaCodec` 在 API 21 一下是不支持的，在 API 21 时添加了对 `KEY_TEMPORAL_LAYERING` 编码属性的配置，但只支持 VP8 编码格式 H264 编码格式不支持（毕竟 Android 和 VPx 都是属于 Google 的产品）。Android 下有需要了解详情的可以查看 [MediaFormat](https://developer.android.google.cn/reference/android/media/MediaFormat.html#KEY_TEMPORAL_LAYERING) 类的说明。

在两大主流的移动平台上，硬件编码器都是不支持 H264/SVC Temporal Scalability 特性，如果要在移动平台上使用该特性只能使用软编码器，可以使用开源的 openh264 编解码库，代码托管在 GitHub 上对应的地址为 [openh264](https://github.com/cisco/openh264) ，其中有该项目的介绍和使用方法。

> x264 编码器当前还不支持 Temporal Scalability 特性。

在 openh264 中使用 `SEncParamExt` 结构体来对编码进行参数的配置，Temporal Scalability 的特性使用 `iTemporalLayerNum` 字段来配置，用来指定输出多少层，这里最大只支持 4 层。

使用 *`EncodeFrame`* 方法进行 H264 的编码后可以得到 `SFrameBSInfo` 的输出结果，该结构体以及关联的结构体的定义如下：

```
typedef struct {
  int           iLayerNum;
  SLayerBSInfo  sLayerInfo[MAX_LAYER_NUM_OF_FRAME];
  EVideoFrameType eFrameType;
  int iFrameSizeInBytes;
  long long uiTimeStamp;
} SFrameBSInfo, *PFrameBSInfo;

typedef struct {
  unsigned char uiTemporalId;
  unsigned char uiSpatialId;
  unsigned char uiQualityId;
  EVideoFrameType eFrameType;
  unsigned char uiLayerType;
  int   iSubSeqId;
  int   iNalCount;
  int*  pNalLengthInByte;
  unsigned char*  pBsBuf;
} SLayerBSInfo, *PLayerBSInfo;
```

在 `SLayerBSInfo` 结构体的 `uiTemporalId` 字段用来描述当前编码后的数据流属于哪一层。openh264 的使用可查看其它文档。

> openh264 对 H264/SVC 的支持，不仅在时间维度上，并且也在空间和质量维度上都有支持。

#### openh264 中 temporal layer 的输出顺序与帧率

在 openh264 的内部，存在一张表，用来记录每一层的输出顺序，可以在 [encoder_data_tables.cpp](https://github.com/cisco/openh264/blob/master/codec/encoder/core/src/encoder_data_tables.cpp) 中找到内容如下：

```
const uint8_t   g_kuiTemporalIdListTable[MAX_TEMPORAL_LEVEL][MAX_GOP_SIZE + 1] = {
  {
    0, 0, 0, 0, 0, 0, 0, 0,
    0
  },  // uiGopSize = 1
  {
    0, 1, 0, 0, 0, 0, 0, 0,
    0
  },  // uiGopSize = 2
  {
    0, 2, 1, 2, 0, 0, 0, 0,
    0
  },  // uiGopSize = 4
  {
    0, 3, 2, 3, 1, 3, 2, 3,
    0
  }  //uiGopSize = 8
};
```

解析如下：

* `iTemporalLayerNum` 的值为 1 时，使用 `uiGopSize = 1` 的配置，即每一帧为一组，每一组的 `uiTemporalId` 值为 0
* `iTemporalLayerNum` 的值为 2 时，使用 `uiGopSize = 2` 的配置，即每两帧为一组，每一组中对应的
`uiTemporalId` 为 `[0, 1]`
* `iTemporalLayerNum` 的值为 3 时，使用 `uiGopSize = 4` 的配置，即每四帧为一组，每一组中对应的
`uiTemporalId` 为 `[0, 2, 1, 2]`
* `iTemporalLayerNum` 的值为 4 时，使用 `uiGopSize = 8` 的配置，即每 8 帧为一组，每一组中对应的
`uiTemporalId` 为 `[0, 3, 2, 3, 1, 3, 2, 3]`

根据上述描述以及输入的帧率可计算每一层的帧率是多少，例如在 30fps 下分两层输出，则 T0 帧率为 15fps；分 3 层时，每 4 帧组则有完整 7 组，则 T0 的帧率有 7 + 1 = 8fps，T1 的帧率有  8 + 7 = 15fps，T2 则有 30fps；分 4 层的情况可按相同的方法计算每一层的帧率。

### RTP 数据包的封装

Temporal Scalability 属于 H264/SVC 的范畴，在封装成 RTP 包时也有对应的标准做法，但流程相对复杂，有兴趣的同学可以了解以下资料：

* [SVC的RTP封装算法及其应用](https://wenku.baidu.com/view/dfc855d5195f312b3169a5dd.html)
* [可伸缩视频编解码SVC技术介绍应用分析](https://blog.csdn.net/vblittleboy/article/details/11872469)

大致的思想是将基本层（低帧率）和增强层（高帧率）的层通过不同的 payload 分开传输，同时两个通道分别使用不同的 QoS 机制来保证传输的质量。

除了上述的较为复杂的做法外也可以使用 RTP 的扩展头来实现，在 RTP 头后面添加一个扩展头用来描述当前数据包的 temporal layer id，这样服务器就能区分 RTP 包分别属于哪个层级，从而根据客户端的网络质量来转发不同的帧率的码流。具体的做法可参考其他资料。

### 总结

H264/SVC 是以 H264/AVC 为基础，利用了AVC 编解码器的各种高效算法工具，在语法和工具集上进行了扩展，所以 H264/SVC 时间可扩展性完全向后兼容 H.264/AVC，即当使用 SVC 编码出多层的码流时，AVC 的编码器也是可以正常解码的。当 Temporal Scalability 帧率降低一半时，比特率通常不会降低一半，因为低层级的帧可能是其他高层级帧的参考帧，编码器会为其分配更多的码流。

由于编码端到服务器以及服务器到接收端的码流有了更灵活的传输方式，结合网络上其他的优化方案，可以衍生出更多的优化方案，例如针对低层级的丢包重传、缓存低层级的帧等。

可惜的是在 iOS 和 Android 两大主流移动平台上，对应的硬件编码器都不支持该特性，不过在分辨率不高的场景下还是可以尝试使用软编码器来实现的。一些时候用户更偏向于接受低分辨率的图像而需要更流畅的画面，这时候 `Temporal Scalability` 机制并不能达到要求，后续将介绍 H264/SVC 中的空间分层来解决该问题。

### 参考资料

* [The Scalable Video Coding Amendment of the H.264/AVC Standard](https://blog.csdn.net/worldpharos/article/details/3369933)
* [Rate Control Optimization for Temporal-LayerScalable Video Coding](https://www.researchgate.net/publication/224227711_Rate_Control_Optimization_for_Temporal-Layer_Scalable_Video_Coding)
* [Chrome’s WebRTC VP9 SVC Layer Cake: Sergio Garcia Murillo & Gustavo Garcia](https://webrtchacks.com/chrome-vp9-svc/)
