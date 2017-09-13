---
title: WebRTC 的丢帧策略
date: 2017-07-29 17:35:15
categories: [音视频/WebRTC]
tags: [WebRTC]
---

> 最近在弄一个 Android 平台下远程桌面项目，是基于 WebRTC 框架实现的，由于平台限制，芯片是用的特定厂商的芯片，桌面采集以及 H264 硬编码都是芯片厂商提供方案，能够有很好性能表现。在将桌面的采集以及 H264 的编码库整合到 WebRTC 框架中时，发现的存在画面延时的问题，于是对 WebRTC 的编解码框架的源码进行了分析，了解到 WebRTC 的两个特性。

在 WebRTC 中有很多可以控制视频帧率和码率的行为，这里介绍一下当视频采集过快以及编码器输出码率过高时，WebRTC 主动丢帧的策略。这里采集过快是指视频采集的速度大于编码器处理的速度。例如视频采集线程每秒采集 30 帧，而编码线程每秒只能处理 15 帧，这时候就出现了采集过快或者说是编码器太慢。编码器输出码率过高，是指编码器的输出码率大于设定的最高输出码率值，例如设置编码器码率为 1024kbps，而实际产生的码率是 1500kbps 这时候就出现了输出码率过高的情况。本文主要介绍上述两种情况下 WebRTC 是如何实现丢帧行为的。

下图是这次涉及到一些类的结构图，如果要在 WebRTC 源码中找对对应的文件，直接搜索类名称即可。WebRTC 的源码由于在墙外，我这里为了后续方便拷贝了一份到我的 GitHub 空间中，[点击这里](https://github.com/EnkiChen/webrtc) 可以找到源码。

![Example](/uploads/webrtc_class_drop.png)
<!--more-->
### 采集过快时的丢帧策略

在 WebRTC 中视频或者桌面图像的采集和图像的编码线程各自是独立线程进行工作的，采集线程会将采集好的图像打包成 `VideoFrame` 对象，这里的 `VideoFrame` 封装了经过转换的 `YUV` 数据而非 `RGB` 数据， `VideoFrame` 在经过一系列的周转后，最终会在 *`ViEEncoder`* 类中得到处理，在 *`ViEEncoder`* 类一部分工作是将数据从采集线程 `post` 到编码线程中，这里就会涉及到多线程间协作的问题。

这里用的是 `rtc::TaskQueue` 队列类，在 *`ViEEncoder`* 类内部，会将所有跟编码器相关的操作都将打包成一个 `Task` 扔到队列中，确保所有的操作都是顺序执行的。在 *`onFrame()`* 方法中接收到采集线程传进来的 `VideoFrame` 对象，并将其打包成一个 `EncodeTask` 对象放入队列中。代码片段如下：

```
void ViEEncoder::OnFrame(const VideoFrame& video_frame) {

    ...

    last_captured_timestamp_ = incoming_frame.ntp_time_ms();
    encoder_queue_.PostTask(std::unique_ptr<rtc::QueuedTask>(new EncodeTask(
      incoming_frame, this, rtc::TimeMicros(), log_stats)));
}
```

`EncodeTask` 是一个内部类，代码非常少，这里就直接贴上来了：

```
class ViEEncoder::EncodeTask : public rtc::QueuedTask {
 public:
  EncodeTask(const VideoFrame& frame,
             ViEEncoder* vie_encoder,
             int64_t time_when_posted_us,
             bool log_stats)
      : frame_(frame),
        vie_encoder_(vie_encoder),
        time_when_posted_us_(time_when_posted_us),
        log_stats_(log_stats) {
    ++vie_encoder_->posted_frames_waiting_for_encode_;
  }

 private:
  bool Run() override {
    RTC_DCHECK_RUN_ON(&vie_encoder_->encoder_queue_);
    RTC_DCHECK_GT(vie_encoder_->posted_frames_waiting_for_encode_.Value(), 0);
    vie_encoder_->stats_proxy_->OnIncomingFrame(frame_.width(),
                                                frame_.height());
    ++vie_encoder_->captured_frame_count_;
    if (--vie_encoder_->posted_frames_waiting_for_encode_ == 0) {
      vie_encoder_->EncodeVideoFrame(frame_, time_when_posted_us_);
    } else {
      // There is a newer frame in flight. Do not encode this frame.
      LOG(LS_VERBOSE)
          << "Incoming frame dropped due to that the encoder is blocked.";
      ++vie_encoder_->dropped_frame_count_;
    }
    if (log_stats_) {
      LOG(LS_INFO) << "Number of frames: captured "
                   << vie_encoder_->captured_frame_count_
                   << ", dropped (due to encoder blocked) "
                   << vie_encoder_->dropped_frame_count_ << ", interval_ms "
                   << kFrameLogIntervalMs;
      vie_encoder_->captured_frame_count_ = 0;
      vie_encoder_->dropped_frame_count_ = 0;
    }
    return true;
  }
  VideoFrame frame_;
  ViEEncoder* const vie_encoder_;
  const int64_t time_when_posted_us_;
  const bool log_stats_;
};
```

第一段代码生成 `EncodeTask` 后直接调用 `PostTask` 方法将其放入队列中，该代码是在采集线程中执行的。在 `EncodeTask` 的构造方法中会对 `posted_frames_waiting_for_encode_` 属性执行一次自增操作，用来记录待编码的帧数。

`EncodeTask`  的 *`Run`* 方法 是在 `rtc::TaskQueue` 的队列线程中执行的，方法中的条件判断可以理解为，当队列中只有当前一个待编码的帧时，才进行编码，否则丢弃当前帧并记录被丢弃帧的数量。由于是一个队列，采集的帧会被依次存入队列中，如果编码线程处理不过来时，编码线程将队列中的待编码的帧依次丢弃，直到最后一帧，然后调用 *`EncodeVideoFrame`* 方法进入下一步的编码处理。

### 编码器码率过高时的丢帧策略

采集过快时，会在 `ViEEncoder` 类中处理，而输出码率过高时则是在 `VideoSender` 类中处理。数据会沿着 *`EncodeVideoFrame`* 方法，数据会经过 `VideoSender` 类的 *`AddVideoFrame`* 方法，在该方法中存在以下逻辑：

```
// Add one raw video frame to the encoder, blocking.
int32_t VideoSender::AddVideoFrame(const VideoFrame& videoFrame,
                                   const CodecSpecificInfo* codecSpecificInfo) {
  ...
  
  if (_mediaOpt.DropFrame()) {
     LOG(LS_VERBOSE) << "Drop Frame "
                     << "target bitrate "
                     << encoder_params.target_bitrate.get_sum_bps()
                     << " loss rate " << encoder_params.loss_rate << " rtt "
                     << encoder_params.rtt << " input frame rate "
                     << encoder_params.input_frame_rate;
     post_encode_callback_->OnDroppedFrame();
     return VCM_OK;
  }
  
  ...
  
  int32_t ret =
      _encoder->Encode(converted_frame, codecSpecificInfo, next_frame_types);
  if (ret < 0) {
    LOG(LS_ERROR) << "Failed to encode frame. Error code: " << ret;
    return ret;
  }
  
```

*`DropFrame`* 方法是 `MediaOptimization` 类的一个方法，用来判断是否丢弃当前帧。 `MediaOptimization` 类实际上并不直接处理该逻辑，而是交给关联的 `FrameDropper` 类对象进行处理。`MediaOptimization` 类主要任务是依赖于编码后的数据对象来计算和记录当前输出码率、帧率以及一些其他信息，这里介绍该类其他两个主要方法：

 *`void SetEncodingData(int32_t max_bit_rate, uint32_t bit_rate, uint16_t width, uint16_t height, uint32_t frame_rate, int num_temporal_layers, int32_t mtu)`* 

该方法用来初始一些配置参数，例如最大码率、帧率等，另一个方法就是：

*`int32_t UpdateWithEncodedData(const EncodedImage& encoded_image);`* 

该方法用来接收编码后的 `EncodedImage` 对象，该对象包含了编码的数据，以及长度时间等信息。至于该方法什么时候会被调用，以及在哪里被调用，这里就不介绍了，因为涉及到编码类的一些结构，比较复杂，有时间单独写一篇专门介绍 WebRTC 的编解码模块的框架结构。

`FrameDropper` 类的声明如下：

```
// The Frame Dropper implements a variant of the leaky bucket algorithm
// for keeping track of when to drop frames to avoid bit rate
// over use when the encoder can't keep its bit rate.
class FrameDropper {
 public:
  FrameDropper();
  explicit FrameDropper(float max_time_drops);
  virtual ~FrameDropper() {}

  // Resets the FrameDropper to its initial state.
  // This means that the frameRateWeight is set to its
  // default value as well.
  virtual void Reset();

  virtual void Enable(bool enable);
  // Answers the question if it's time to drop a frame
  // if we want to reach a given frame rate. Must be
  // called for every frame.
  //
  // Return value     : True if we should drop the current frame
  virtual bool DropFrame();
  // Updates the FrameDropper with the size of the latest encoded
  // frame. The FrameDropper calculates a new drop ratio (can be
  // seen as the probability to drop a frame) and updates its
  // internal statistics.
  //
  // Input:
  //          - frameSizeBytes    : The size of the latest frame
  //                                returned from the encoder.
  //          - deltaFrame        : True if the encoder returned
  //                                a key frame.
  virtual void Fill(size_t frameSizeBytes, bool deltaFrame);

  virtual void Leak(uint32_t inputFrameRate);

  // Sets the target bit rate and the frame rate produced by
  // the camera.
  //
  // Input:
  //          - bitRate       : The target bit rate
  virtual void SetRates(float bitRate, float incoming_frame_rate);
  
  ...
}
```

可以了解到  `FrameDropper` 类是一个漏斗算法的一种变体实现，当编码器无法保证比特率时， 用来跟踪何时应该丢弃帧。具体的算法实现代码我没有细看，有兴趣的同学可以看看，从方法的输入参数可以推测，计算模型跟时间、帧率、数据量以及是否为关键帧几个参数相关。可以理解为通过计算之前编码后的实际码率，来判断当前是否丢弃帧的。

从上面的分析可以了解到在 *`VideoSender::AddVideoFrame`* 方法中，间接的依赖了 `FrameDropper` 类的漏斗算法模型，来处理是否丢弃当前帧。从而确保输出码率不大于设定的值。

### 总结

当采集过快或者编码过慢都可能导致编码队列中存在很多待处理的帧，从而导致码率过高或者画面延时很大（可能还取决于编码器）。主动丢弃一些帧行，可以有效的控制码率和画面延时，代价就是损失帧率。在一些情况下编码器的输出码率并不能达到预先设定的值，从而导致码率过高增加了网络负担，第二种丢帧行为可以根据实际的码率来控制最终的码率，而非通过参数来调整。

可能有些人会有疑问，为什么采集就会过快呢？编码器可以通过参数设定码率，为何又不能按要求输的给定的码率呢？这里从两方面来表达我个人的看法。第一个问题可以从软件设计原则角度来解析，采集模块与编码模块可以说是两个相对独立的模块，两个模块分别独立工作，各自都不需要依赖对方的具体实现，知道的细节越少越好。简单的说就是依赖接口编程，而非具体实现。第二个问题也比较好解析，H264 的编码参数有很多例如帧率、码率、Qp、Profiles 级别等一些参数，并且一些参数之间相互影响，一些特定的编码器可能还存在一些特定的编码参数。在配置参数时可能存在一些不合理的情况。编解码器（暂且为H264）可以有两种实现方式，一种是硬编码，依赖于一些特点硬件设备；另一种是纯软件算法实现；软编码相对来说稳定，硬编解码在不同平台下存在差异，尤其是 Android 设备下，差异更大。