title: 优化APP时发现的SDWebImage的问题
date: 2015-12-22 19:34:17
categories: 开发笔记 
tags: [SDWebImage, NSCache, 旋转]
---
### 起因 旋转了的图片
前段时间在对程序的绘图逻辑进行了使用OpenGL做优化，但是在使用OpenGL进行绘制时发现一些图片会莫名其妙的旋转，在细看了代码之后发现问题发生在**UIImage**对象的**imageOrientation**属性。该属性为枚举类型可以有以下取值：  
<!--more-->

```
typedef enum {
    UIImageOrientationUp,            // default orientation  默认方向   
    UIImageOrientationDown,          // 180 deg rotation     旋转180度    
    UIImageOrientationLeft,          // 90 deg CCW           逆时针旋转90度
    UIImageOrientationRight,         // 90 deg CW            顺时针旋转90度
    UIImageOrientationUpMirrored,    // horizontal flip      向上水平翻转
    UIImageOrientationDownMirrored,  // horizontal flip      向下水平翻转
    UIImageOrientationLeftMirrored,  // vertical flip        逆时针旋转90度，垂直翻转
    UIImageOrientationRightMirrored, // vertical flip        顺时针旋转90度，垂直翻转
} UIImageOrientation;
```

该属性用来记录照片的方向信息，在使用iPhone或者iPad自带的照相机拍摄出来的照片含有EXIF信息，而在使用Core Graphics进行绘制时，会进行一些转换，我在使用OpenGL绘制时直接使用的原图，转换成纹理对象时是旋转的图片对象，所以在使用OpenGL绘制会出现旋转问题。在网上找了一下代码做了调整：

```
- (UIImage *)fixOrientation{
    
    // No-op if the orientation is already correct
    if (self.imageOrientation == UIImageOrientationUp)
        return self;
    
    // We need to calculate the proper transformation to make the image upright.
    // We do it in 2 steps: Rotate if Left/Right/Down, and then flip if Mirrored.
    CGAffineTransform transform = CGAffineTransformIdentity;
    
    switch (self.imageOrientation) {
        case UIImageOrientationDown:
        case UIImageOrientationDownMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.width, self.size.height);
            transform = CGAffineTransformRotate(transform, M_PI);
            break;
            
        case UIImageOrientationLeft:
        case UIImageOrientationLeftMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.width, 0);
            transform = CGAffineTransformRotate(transform, M_PI_2);
            break;
            
        case UIImageOrientationRight:
        case UIImageOrientationRightMirrored:
            transform = CGAffineTransformTranslate(transform, 0, self.size.height);
            transform = CGAffineTransformRotate(transform, -M_PI_2);
            break;
        default:
            break;
    }
    
    switch (self.imageOrientation) {
        case UIImageOrientationUpMirrored:
        case UIImageOrientationDownMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.width, 0);
            transform = CGAffineTransformScale(transform, -1, 1);
            break;
            
        case UIImageOrientationLeftMirrored:
        case UIImageOrientationRightMirrored:
            transform = CGAffineTransformTranslate(transform, self.size.height, 0);
            transform = CGAffineTransformScale(transform, -1, 1);
            break;
        default:
            break;
    }
    
    // Now we draw the underlying CGImage into a new context, applying the transform
    // calculated above.
    CGContextRef ctx = CGBitmapContextCreate(NULL, self.size.width, self.size.height,
                                             CGImageGetBitsPerComponent(self.CGImage), 0,
                                             CGImageGetColorSpace(self.CGImage),
                                             CGImageGetBitmapInfo(self.CGImage));
    CGContextConcatCTM(ctx, transform);
    switch (self.imageOrientation) {
        case UIImageOrientationLeft:
        case UIImageOrientationLeftMirrored:
        case UIImageOrientationRight:
        case UIImageOrientationRightMirrored:
            // Grr...
            CGContextDrawImage(ctx, CGRectMake(0,0,self.size.height,self.size.width), self.CGImage);
            break;
            
        default:
            CGContextDrawImage(ctx, CGRectMake(0,0,self.size.width,self.size.height), self.CGImage);
            break;
    }
    
    // And now we just create a new UIImage from the drawing context
    CGImageRef cgimg = CGBitmapContextCreateImage(ctx);
    UIImage *img = [UIImage imageWithCGImage:cgimg];
    CGContextRelease(ctx);
    CGImageRelease(cgimg);
    return img;
}
```
### 再次遇到旋转问题

旋转问题得到解决之后，使用了**SDWebImage**自带的**SDImageCache**类进行了图片的缓存管理，该类能够做到内存不足时释放不需要的图片对象。类底层使用了系统的**NSCache**实现的，能够在系统内存不足时释放被管理的对象。   

在使用**SDImageCache**类做缓存之后，我们的测试MM发现在一些情况下会出现图片旋转的问题。而且是 Core Graphics 和 OpenGL 两个环境下都会出现。因为有了前面的经验，很快就找到了方向，那就是**UIImage**对象的**imageOrientation**属性出问题了，经过一步步调试发现，在**UIImage**对象交给**SDImageCache**管理之后，再从缓存中拿出来时**imageOrientation**属性会不一致。于是找到了**SDImageCache**类的部分源码，如下：

```
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk {
    if (!image || !key) {
        return;
    }

    [self.memCache setObject:image forKey:key cost:image.size.height * image.size.width * image.scale * image.scale];

    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;

            if (image && (recalculate || !data)) {
#if TARGET_OS_IPHONE
                // We need to determine if the image is a PNG or a JPEG
                // PNGs are easier to detect because they have a unique signature (http://www.w3.org/TR/PNG-Structure.html)
                // The first eight bytes of a PNG file always contain the following (decimal) values:
                // 137 80 78 71 13 10 26 10

                // We assume the image is PNG, in case the imageData is nil (i.e. if trying to save a UIImage directly),
                // we will consider it PNG to avoid loosing the transparency
                BOOL imageIsPng = YES;

                // But if we have an image data, we will look at the preffix
                if ([imageData length] >= [kPNGSignatureData length]) {
                    imageIsPng = ImageDataHasPNGPreffix(imageData);
                }

                if (imageIsPng) {
                    data = UIImagePNGRepresentation(image);
                }
                else {
                    data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
                }
#else
                data = [NSBitmapImageRep representationOfImageRepsInArray:image.representations usingType: NSJPEGFileType properties:nil];
#endif
            }

            if (data) {
                if (![_fileManager fileExistsAtPath:_diskCachePath]) {
                    [_fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
                }

                [_fileManager createFileAtPath:[self defaultCachePathForKey:key] contents:data attributes:nil];
            }
        });
    }
}

```
可以知道**UIImage**对象在IO线程中直接使用**UIImagePNGRepresentation**或者**UIImageJPEGRepresentation**方法转换成**NSData**对象然后直接存储到文件中了。 **imageData**参数可以从以下两个方法知道，默认情况下是传的**nil**值。

```
- (void)storeImage:(UIImage *)image forKey:(NSString *)key {
    [self storeImage:image recalculateFromImage:YES imageData:nil forKey:key toDisk:YES];
}

- (void)storeImage:(UIImage *)image forKey:(NSString *)key toDisk:(BOOL)toDisk {
    [self storeImage:image recalculateFromImage:YES imageData:nil forKey:key toDisk:toDisk];
}
```
 
也就是说我们在使用**SDImageCache**做缓存时，默认会当做PNG图片存储到文件。而经过测试发现**UIImagePNGRepresentation**方法转换成**NSData**对象时，EXIF信息将会丢失，而**UIImageJPEGRepresentation**方法则不会。在找到原因之后，问题就可以很好的解决了，只要在加入到缓存之前，将图片恢复到正常方向，再将图片保存到缓存中即可。  

### 获取缓存失败
本以为问题都解决了，然而第二天我们测试发现在一些情况下，插入图片时会显示一张默认图片（图片如果加载失败会显示一张默认的小图），在跟测试MM做了一些沟通之后，走读了一下代码流程，并进行了调试，发现图片被放进缓存之后，再次进行获取时，却获取不到数据。  

想不通为什么，于是看了一下**SDImageCache**类的源码（前面有提到），结合调试发现的一些信息，在加载iPad（我们的应用是iPad）拍摄的照片并且在插入多张图片之后才会出现这种现象，于是猜想是不是在**UIImage**对象在加入到缓存中之后，这时候收到内存警告或者超过设定的阀值，导致被加入的对象在IO线程还未将图片写入到磁盘之前就被释放了，这样将导致从内存和磁盘中都获取不到数据。为了验证这个猜想，修改了**SDImageCache**源码，设置了memCache的delegate，**NSCache**有个delegate，协议如下：

```
@protocol NSCacheDelegate <NSObject>
@optional
- (void)cache:(NSCache *)cache willEvictObject:(id)obj;
@end
```

通过该protocol便可以知道被加入缓存中的对象什么时候被释放。在delegate中、IO线程中写入文件成功之后以及获取**UIImage**对象时添加一些Log信息。这样便可以知道各个逻辑的执行流程。经过验证，执行流程我猜想的一样，**UIImage**对象在加入到缓存一小会时间之后立马被释放了，这时候IO线程还未执行完成，这时候从缓存中是获取不到缓存数据的。从而导致失败显示默认图片。  

问题已经找到了，但是要解决这个问题很是蛋疼，分别有以下几种做法：
* 修改上层逻辑代码等待IO线程写入成功后，才执行后续流程，这样确保一定能获取到数据。
* 修改**SDWebImage**让内存数据不那么快被释放。
* 自己重新造个轮子达到我们想要的要求。

对于上述修改第一条有点恶心，有点打补丁的节奏。针对后两条不太实际，工作量太大，最近在[**ibireme**](http://blog.ibireme.com/)的博客上看一个[**YYCache**](https://github.com/ibireme/YYCache)的开源项目，看了介绍还不错，于是将项目中的缓存直接换成了**YYCache**，做了相同的测试，发现问题没有出现了。^o^，后续将阅读一下**YYCache**的源代码了，看看具体的实现方法。

### 总结
在优化的过程中发现了**SDWebImage**的两个问题分别为：
* 在使用**SDWebImage**做图片缓存时，图片默认会被当做PNG格式存储，而**UIImagePNGRepresentation**方法转换成**NSData**时会丢失EXIF信息，这样当再次从磁盘读取数据时，将是丢失EXIF信息的图片，从而导致无法进行正常的图片旋转。
* 使用**SDWebImage**做缓存时，当内存到达一个临界值时，加入的新的缓存对象，会在IO线程写入文件之前释放，在内存对象被释放，IO未完成写入的这段时间内，无法正确获取到缓存数据。  

[**SDWebImage**](https://github.com/rs/SDWebImage)很多人在用，而且使用起来也很方便，尤其在加载网络图片时非常方便，项目中还将继续使用用来做网络图片的加载，不过内存缓存会使用[**YYCache**](https://github.com/ibireme/YYCache)， 针对[**YYCache**](https://github.com/ibireme/YYCache)这个类，还没来得及细看代码，网上评价作者代码相当工整，看过[**YYCategories**](https://github.com/ibireme/YYCategories)部分源码实现，确实写的不错，今后多看看，向大牛们学习。









