title: 两则开发小笔记
date: 2016-07-22 14:22:53
categories: 开发笔记
tags: [Core Graphics, GCDAsyncSocket]
---

好久没有写博客了，介绍一下项目中一个 Core Graphics 绘制图片时的效率优化方法，以及记录一下 GCDAsyncSocket 框架在读取网络数据要注意的一个点。

### Core Graphics 绘图性能对比

Core Graphics 框架为我们提供了 2D 绘图能力，在使用 Core Graphics 绘制图片时，不同的实现方法使得渲染图片的效率有很大差异，下面是提供两种不同实现方式的对比，看看 Core Graphics 在绘制图片时效率上的差异。

测试方法为自定义一个 `UIView`，重写`- (void)drawRect:(CGRect)rect` 方法，在该方法中使用 Core Graphics 方式绘制图片。然后在开启一个定时器不断的调用 `- (void)setNeedsDisplay;` 方法来计算绘图所使用的时间。
<!--more-->
#### 方法一

```
- (void)awakeFromNib {
    self.coreImage = [UIImage imageNamed:@"IMG_0079.png"];
}

- (void)drawRect:(CGRect)rect {
    struct timeval start , end ;
    gettimeofday(&start, NULL);
    [self.coreImage drawAtPoint:CGPointZero];
    // [self.coreImage drawInRect:CGRectMake(0, 0, 512, 512)];
    gettimeofday(&end, NULL);
    long cost = ((end.tv_sec - start.tv_sec) * 1000000 + end.tv_usec - start.tv_usec ) / 1000 ;

    NSLog(@"cost:%ld", cost);
}
```

#### 方法二

```
- (void)awakeFromNib {
    self.coreImage = [UIImage imageNamed:@"IMG_0079.png"];
    CGSize size = self.frame.size;
    UIGraphicsBeginImageContextWithOptions(size, NO, 0.0);
    [self.coreImage drawInRect:CGRectMake(0, 0, size.width, size.height)];
    self.coreImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
}

- (void)drawRect:(CGRect)rect {
    struct timeval start , end ;
    gettimeofday(&start, NULL);
    [self.coreImage drawAtPoint:CGPointZero];
    // [self.coreImage drawInRect:CGRectMake(0, 0, 512, 512)];
    gettimeofday(&end, NULL);
    long cost = ((end.tv_sec - start.tv_sec) * 1000000 + end.tv_usec - start.tv_usec ) / 1000 ;

    NSLog(@"cost:%ld", cost);
}
```

> 使用 `[self.coreImage drawInRect:CGRectMake(0, 0, 512, 512)];` 方法来进行图片的缩放，大小可自定义，只要跟图片本身大小不一致即可。

#### 对比结果

| 方法 | 是否缩放 | 耗时时间 |
|  :--: | :--: | :--: |
| 方法一 | NO | 10ms ~ 20ms |
| 方法一 | YES | 100ms ~ 120ms |
| 方法二 | NO | 1ms ~ 2ms |
| 方法二 | YES | 30ms ~ 40ms |

我测试的手机为 iPhone 6 机型，图片为 iPhone 拍摄的照片，上表的中耗时时间跟图片以及机器有关，看对比结果即可；从对比结果可以知道如果在绘制时绘制的大小跟图片本身大小不一致时，系统会对其进行缩放，从而导致绘制效率变低，如果我们提前将图片缩放到指定大小，从而可以提高在绘制时的效率；使用 `UIGraphicsGetImageFromCurrentImageContext` 之前有看到过一篇文章提到，该方法会将图片缓存到显存中，从而使得绘制效率的提高，不过这篇文章已经找不到了。

在实际项目中可选择合适的方法的进行优化，比如图片过大可以使用方法二进行缩放，同时可以放到子线程中进行处理，从而提供绘制时的效率。我在实际项目中的需求为图片可能进行缩放，所以我针对图片做了两级缓存，缩放后在子线程中生成缩放后的图片，同时在缩放的过程中使用了 OpenGL 进行渲染。

### GCDAsyncSocket 读取数据的注意事项

今天一同事使用 GCDAsyncSocket 在测试一个网络程序，发现使用我给的代码，无法接收到数据。代码大概如下：

```
- (void)push {
    if (!self.asyncSocket) {
        self.asyncSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
    }
    
    if (!self.bufferData) {
        self.bufferData = [[NSMutableData alloc] init];
    }
    
    NSError *err = nil;
    if (![self.asyncSocket connectToHost:PUSH_HOST onPort:PUSH_PORT error:&err]) {
        NSLog(@"I goofed: %@", err);
        return;
    }
}

- (void)socket:(GCDAsyncSocket *)sender didConnectToHost:(NSString *)host port:(UInt16)port {
    NSString *contentString = @"data"
    NSData *data = [contentString dataUsingEncoding:NSUTF8StringEncoding];
    [self.asyncSocket writeData:data withTimeout:-1 tag:PUSH_DATA_TAG];
    [self.asyncSocket readDataToLength:100 withTimeout:-1 buffer:self.bufferData bufferOffset:0 tag:PUSH_DATA_TAG];
}

- (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(NSError *)err {
    DDLogWarn(@"SocketDidDisconnect:WithError: %@", err);
}

- (void)socket:(GCDAsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag {
    DDLogWarn(@"socket:didReadData: %ld", tag);
}
```

代码流程很简单，客户端在连接上服务器上之后，会给服务器发送一段数据，服务器接收到数据后，在返回一个状态码给到客户端，方法使用上都是正常的；服务器在响应客户端之后，会直接断开连接，上述代码在客户端的实际表现为发送数据到服务器之后，直接就收到断开的回调消息了，并没有接收到服务器响应的数据。

经过测试后发现，`- (void)readDataToLength:(NSUInteger)length withTimeout:(NSTimeInterval)timeout buffer:(NSMutableData *)buffer bufferOffset:(NSUInteger)offset tag:(long)tag;` 一定要接收到指定数据长度后，才会回调对应的方法，而实际服务器返回给到客户端的数据只有两个字节，导致接收数据的方法无法得到回调，就算是连接被断开后也是无法被回调的，导致上层是没办法获取到对应的数据，针对该问题我们可以使用 `- (void)readDataWithTimeout:(NSTimeInterval)timeout buffer:(NSMutableData *)buffer bufferOffset:(NSUInteger)offset tag:(long)tag;` 来解决该问题。

所以在使用 `GCDAsyncSocket` 时得要注意使用合适的方法来获取数据，否则会导致特定情况下无法接收数据回调。