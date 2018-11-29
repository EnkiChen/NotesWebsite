---
title: Mac OSX 鼠标键盘事件的监听和模拟
date: 2018-09-12 19:15:45
categories: [知识整理/总结]
tags: [OSX]
---

最近完成了 Mac OSX 平台下的远程控制功能，期间找了不少资料，这里做个总结，主要涉及到一下知识点：

1. OSX 的事件机制
2. OSX/iOS 响应链者链
3. 鼠标事件的监听及模拟（鼠标单击、双击、拖动、滚动等事件）
4. 键盘事件的监听及模拟（包括组合键的模拟）
5. Keycode 键盘编码（统一 Windows、OSX、浏览器端键盘按键的编码值）

### 事件分发机制

在 OSX 系统中鼠标和键盘的活动事件都会产生底层的系统事件，首先传递到 IOKit 框架处理后存储到队列中，通知 Window Server 服务层处理。Window Server 存储到 FIFO 优先队列中，然后逐一转发到当前活动窗口或者能响应这个事件的应用程序去处理。

<!--more-->

在 OSX 或者 iOS 程序中，都会有一个 Main Run Loop 的线程，RunLoop 循环中会遍历 event 消息队列，逐一分发这些事件到应用中合适的对象去处理。具体来说就是调用 `NSApp` 的 *`sendEvent:`* 方法发送消息到`NSWindow`，`NSWindow` 再分发到 `NSView` 视图对象，由其鼠标或键盘事件响应方法去处理。

![Alt text](/uploads/EventDispatch.png)

###  事件响应链

在 OSX 和 iOS 程序中响应者链是 Application Kit 事件处理架构的中心机制，它由一系列链接在一起的响应者对象组成，事件或者动作消息可以沿着这些对象进行传递。如果一个响应者对象不能处理某个事件或动作，也就是说，它不响应那个消息，或者不认识那个事件，则将该消息重新发送给链中的下一个响应者。消息沿着响应者链向上、向更高级别的对象传递，直到最终被处理（如果最终还是没有被处理，就会被抛弃）。

事件响应者 `Responders` 类为核心应用程序架构的三个主要模式或机制定义了一个接口：

* 它声明了一些处理事件消息（也就是源自用户事件的消息，比如象鼠标点击或按键按下这样的事件）的方法。
* 它声明了数十个处理动作消息的方法，它们和标准的键绑定（比如那些在文本内部移动插入点的绑定）密切相关。动作消息会被派发到目标对象；如果目标没有被指定，应用程序会负责检索合适的响应者。
* 它定义了一套在应用程序中指派和管理响应者的方法。这些响应者组成了我们所知道的响应者链，即一系列响应者，事件或动作消息在它们之间传递，直到找到能够对它们进行处理的对象。

当 Application Kit 在应用程序中构造对象时，会为每个窗口建立响应者链。响应者链中的基本对象是`NSWindow` 对象及其视图层次。在视图层次中级别较低的视图将比级别更高的视图优先获得处理事件或动作消息的机会。**`NSWindow` 中保有一个第一响应者的引用，它通常是当前窗口中处于选择状态的视图，窗口通常把响应消息的机会首先给它**。对于事件消息，响应者链通常以发生事件的窗口对应的 `NSWindow` 对象作为结束，虽然其它对象也可以作为下一个响应者被加入到 `NSWindow` 对象的后面。

从层级上看离观察者最近的视图优先响应事件，通过 `view` 的 *`hitTest`* 方法检测，满足 *`hitTest`* 方法的的子视图优先响应事件。

`NSApplication`, `NSWindow`, `NSDrawer`, `NSWindowController`, `NSView` 以及继承于 `NSView` 的所有控件对象都直接或间接继承了 `Responders` 类，所以这些类都能处理鼠标和键盘事件。

iOS 程序相比于 OSX 程序会有点不一样：

1. OSX 程序可能存在多个窗口，会有多个响应者链，iPhone 的应用程序就一个窗口，所以只会有一个响应者链。
2. 在 iOS 程序中与加速计、陀螺仪和磁力计相关的运动事件不遵循响应者链，Core Motion 会将这些事件直接传递给我们指定的对象。有关更多信息，可以参看 [Core Motion Framework](https://developer.apple.com/library/content/documentation/Miscellaneous/Conceptual/iPhoneOSTechOverview/CoreServicesLayer/CoreServicesLayer.html#//apple_ref/doc/uid/TP40007898-CH10-SW27)。

### 相关类的解析说明

#### NSResponder 

`NSResponder` 在这里是非常重要的一个类，其中定义了鼠标键盘触控板等多种事件，这里列举一些鼠标跟键盘的主要方法：

```
// 鼠标按下事件
- (void)mouseDown:(NSEvent *)event;

// 鼠标右键按下事件
- (void)rightMouseDown:(NSEvent *)event;

// 鼠标抬起事件
- (void)mouseUp:(NSEvent *)event;

// 鼠标右键抬起事件
- (void)rightMouseUp:(NSEvent *)event;

// 鼠标移动事件
- (void)mouseMoved:(NSEvent *)event;

// 鼠标拖拽事件
- (void)mouseDragged:(NSEvent *)event;

// 鼠标滚动事件
- (void)scrollWheel:(NSEvent *)event;

// 鼠标右键拖拽事件
- (void)rightMouseDragged:(NSEvent *)event;

// 鼠标进入监控区域事件
- (void)mouseEntered:(NSEvent *)event;

// 鼠标离开监控区域事件
- (void)mouseExited:(NSEvent *)event;

// 键盘按下事件
- (void)keyDown:(NSEvent *)event;

// 键盘按下事件
- (void)keyUp:(NSEvent *)event;

// 键盘控制键的按下标记状态发送改变，后面用该方法来获取控制按下事件，参考 NSEventModifierFlags 定义
- (void)flagsChanged:(NSEvent *)event;
```

`NSResponder` 除了定义基本的响应事件外，还定义了很多其他事件方法。具体请参考 *`NSResponder.h`* 的头文件定义。

#### NSEvent

`NSEvent` 类描述了事件的具体信息，这里列举跟鼠标和键盘相关的一些字段的介绍：

```
// 事件类型
@property (readonly) NSEventType type;

// 键盘控制键的按下状态的标记
@property (readonly) NSEventModifierFlags modifierFlags;

// 事件的时间戳
@property (readonly) NSTimeInterval timestamp;

// 鼠标点击的次数（只有鼠标事件，才可使用）
@property (readonly) NSInteger clickCount;
@property (readonly) NSInteger buttonNumber; 
@property (readonly) NSInteger eventNumber;

// 压力值
@property (readonly) float pressure;

// 鼠标在窗口的位置
@property (readonly) NSPoint locationInWindow;

// 鼠标滚动时。分别在 X 和 Y 轴上的偏移 
@property (readonly) CGFloat scrollingDeltaX NS_AVAILABLE_MAC(10_7);
@property (readonly) CGFloat scrollingDeltaY NS_AVAILABLE_MAC(10_7);

// 键盘事件的字符编码和 key code 值
@property (nullable, readonly, copy) NSString *characters;
@property (readonly) unsigned short keyCode;
```

#### NSEventType

`NSEventType` 类型定义了事件的具体类型，如下：

```
typedef NS_ENUM(NSUInteger, NSEventType) {        /* various types of events */
    NSEventTypeLeftMouseDown             = 1,
    NSEventTypeLeftMouseUp               = 2,
    NSEventTypeRightMouseDown            = 3,
    NSEventTypeRightMouseUp              = 4,
    NSEventTypeMouseMoved                = 5,
    NSEventTypeLeftMouseDragged          = 6,
    NSEventTypeRightMouseDragged         = 7,
    NSEventTypeMouseEntered              = 8,
    NSEventTypeMouseExited               = 9,
    NSEventTypeKeyDown                   = 10,
    NSEventTypeKeyUp                     = 11,
    NSEventTypeFlagsChanged              = 12,
    NSEventTypeAppKitDefined             = 13,
    NSEventTypeSystemDefined             = 14,
    NSEventTypeApplicationDefined        = 15,
    NSEventTypePeriodic                  = 16,
    NSEventTypeCursorUpdate              = 17,
    NSEventTypeScrollWheel               = 22,
    NSEventTypeTabletPoint               = 23,
    NSEventTypeTabletProximity           = 24,
    NSEventTypeOtherMouseDown            = 25,
    NSEventTypeOtherMouseUp              = 26,
    NSEventTypeOtherMouseDragged         = 27,
    /* The following event types are available on some hardware on 10.5.2 and later */
    NSEventTypeGesture NS_ENUM_AVAILABLE_MAC(10_5)       = 29,
    NSEventTypeMagnify NS_ENUM_AVAILABLE_MAC(10_5)       = 30,
    NSEventTypeSwipe   NS_ENUM_AVAILABLE_MAC(10_5)       = 31,
    NSEventTypeRotate  NS_ENUM_AVAILABLE_MAC(10_5)       = 18,
    NSEventTypeBeginGesture NS_ENUM_AVAILABLE_MAC(10_5)  = 19,
    NSEventTypeEndGesture NS_ENUM_AVAILABLE_MAC(10_5)    = 20,
    
#if __LP64__
    NSEventTypeSmartMagnify NS_ENUM_AVAILABLE_MAC(10_8) = 32,
#endif
    NSEventTypeQuickLook NS_ENUM_AVAILABLE_MAC(10_8) = 33,
    
#if __LP64__
    NSEventTypePressure NS_ENUM_AVAILABLE_MAC(10_10_3) = 34,
    NSEventTypeDirectTouch NS_ENUM_AVAILABLE_MAC(10_10) = 37,
#endif
};
```

#### NSEventModifierFlags

`NSEventModifierFlags` 类型描述了一些控制键，是否处于按下状态，定义如下：

```
/* Device-independent bits found in event modifier flags */
typedef NS_OPTIONS(NSUInteger, NSEventModifierFlags) {
    NSEventModifierFlagCapsLock           = 1 << 16, // Set if Caps Lock key is pressed.
    NSEventModifierFlagShift              = 1 << 17, // Set if Shift key is pressed.
    NSEventModifierFlagControl            = 1 << 18, // Set if Control key is pressed.
    NSEventModifierFlagOption             = 1 << 19, // Set if Option or Alternate key is pressed.
    NSEventModifierFlagCommand            = 1 << 20, // Set if Command key is pressed.
    NSEventModifierFlagNumericPad         = 1 << 21, // Set if any key in the numeric keypad is pressed.
    NSEventModifierFlagHelp               = 1 << 22, // Set if the Help key is pressed.
    NSEventModifierFlagFunction           = 1 << 23, // Set if any function key is pressed.
    
    // Used to retrieve only the device-independent modifier flags, allowing applications to mask off the device-dependent modifier flags, including event coalescing information.
    NSEventModifierFlagDeviceIndependentFlagsMask    = 0xffff0000UL
};
```

### 事件的监听方法

鼠标键盘事件的监听有多种方法，第一种方法是重写事件响应者 `Responders` 对应的方法来获取对应的事件；第二是通过重写 `NSWindow` 的 *`sendEvent:`* 方法； 第三是通过的 `NSEvent` 提供静态方法来监听对应的事件：

```
+ (nullable id)addGlobalMonitorForEventsMatchingMask:(NSEventMask)mask handler:(void (^)(NSEvent*))block`

+ (nullable id)addLocalMonitorForEventsMatchingMask:(NSEventMask)mask handler:(NSEvent* __nullable (^)(NSEvent*))block

+ (void)removeMonitor:(id)eventMonitor
```

`NSEvent` 提供的静态方法可以用监听整个系统的事件或者当前应用程序内的事件。全局事件是异步过程因此无法修改事件，应用程序内的消息可以在捕获到消息后，修改事件然后继续交由响应者链中下一个响应者处理。

### 鼠标事件监听

这里介绍鼠标的一下事件处理方法和注意事项：

1. 左/右键的按下抬起事件
2. 左键的双击（或者多击事件）
3. 左键或者右键的拖拽事件
4. 鼠标移动事件
5. 鼠标的滚动事件

前面介绍了三种监听事件的方法，这里使用重写 `Responders` 的方法来监听鼠标事件：

```
- (void)mouseDown:(NSEvent *)event;
- (void)rightMouseDown:(NSEvent *)event;
- (void)mouseUp:(NSEvent *)event;
- (void)rightMouseUp:(NSEvent *)event;
- (void)mouseMoved:(NSEvent *)event;
- (void)mouseDragged:(NSEvent *)event;
- (void)rightMouseDragged:(NSEvent *)event;
- (void)scrollWheel:(NSEvent *)event;
```

鼠标按键的按下抬起事件，只要判断一下 `NSEvent` 的 *`type`* 属性即可知道。

当前鼠标的位置信息可通过 *`locationInWindow`* 属性来获取，该坐标是当前 Window 窗口的坐标，其中包含了 Window 窗口标题栏的高度，所以如果要想获取当前鼠标在当前 `NSView` 中的位置，需要做一次坐标转换，可以调用 `NSView` 的 *`convertPoint:`* 方法来转换坐标。

鼠标左键的 `按下 - 抬起` 两个连续的动作被定义为单击事件，*`clickCount`* 属于用于描述当前点击的次数。在模拟鼠标双击时，需要用到该字段值，而不能用连续两次点击事件来模拟双击。

监听鼠标的移动事件时需要设置一个跟踪区域，只有在跟踪区域内的鼠标移动事件才会触发。可以通过 `NSView` 的 `- (void)addTrackingArea:(NSTrackingArea *)trackingArea` 方法来设置跟踪区域。同时需要重写 `- (void)updateTrackingAreas` 方法，当跟踪区域发送改变时，需要手动将之前的跟踪区域移除，再添加新的跟踪区域。

鼠标的拖拽事件是指用户按下鼠标左键或右键移动鼠标，当拖拽事件发生时 *`mouseMoved:`* 事件将不会触发。

鼠标的滚动可以通过 *`deltaX`* 和 *`deltaY`* 两个属性来获取分别在水平方向和垂直方向的滚动偏移。

> OSX 的坐标系统以左下角为 (0,0) 右上角为 (x_max, y_max)

### 键盘事件的监听

键盘事件的监听也使用重写事件响应者 `Responders` 对应的方法来实现，需要重写的方法如下：

```
- (void)keyDown:(NSEvent *)event;
- (void)keyUp:(NSEvent *)event;
- (void)flagsChanged:(NSEvent *)event;
```

键盘事件重写上述方法外还需要重写以下方法：

```
- (BOOL)acceptsFirstResponder;
```

该方法用来说明是否成为响应者链的第一个响应者，这里需要返回 `YES` 表示成为第一响应者，否则无法监听键盘消息。

`NSEvent` 的 *`characters`* 描述了当前键盘按键的字符，*`keyCode`* 描述了按键的值，每个按键的 `keyCode` 值定义可以在 *`Carbon/HIToolbox/Events.h`* 文件中找到对应的按键的宏定义。

在 *`keyDown:`* 和 *`keyUp:`* 方法中可以监听到大部分的按键的消息，但一些控制键需要通过 *`flagsChanged:`* 方法来处理，当 `NSEventModifierFlags` 定义的按键状态发送改变时，该方法就会被触发。这里需要注意的是大小写锁定键 `NSEventModifierFlagCapsLock` 只有当大写锁定或者取消锁定时，该方法才会被调用，并不会因为 `CapsLock` 按键按下或者抬起时触发。

> keyCode 值在 Windows 和浏览器上都有对应的键盘按键的值的定义，当需要与其他平台进行通信时，例如远程控制时，可以将 Mac 下的 keyCode 值转换成浏览器 JS 上的对应值定义，因为浏览器和 Windows 平台的定义是一致的。

>  *`CGEventSourceKeyState(kCGEventSourceStateHIDSystemState, kVK_CapsLock)`* 方法可以用来获取按键是否处于按下状态。

### 鼠标键盘事件的模拟

OSX 下的鼠标和键盘事件模拟需要用到 `CoreGraphics` 及 `Carbon` 框架，在 `CoreGraphics` 框架中定义了一些用于创建底层事件的方法，`Carbon` 框架定义了一些跟键盘相关的宏和方法。

在模拟鼠标或者键盘事件时，都需要使用 *`CGEventSourceCreate(CGEventSourceStateID stateID)`* 方法来创建事件源，事件源类型定义了 3 个类型，如下：

```
typedef CF_ENUM(int32_t, CGEventSourceStateID) {
  kCGEventSourceStatePrivate = -1,
  kCGEventSourceStateCombinedSessionState = 0,
  kCGEventSourceStateHIDSystemState = 1
};
```

* `kCGEventSourceStatePrivate`：代表专门的应用，如远程控制程序可以生成和跟踪事件源状态独立于其他进程。
* `kCGEventSourceStateCombinedSessionState`：该状态表反映了所有事件源的组合状态发布到当前用户的登录会话。如果您的程序发布的事件在一个登录会话，您应该使用这个源状态当你创建一个事件源。
* `kCGEventSourceStateHIDSystemState`：该状态表反映了组合硬件输入源从 HID 系统硬件层面发送的事件源。生成的事件。 就是外接键盘或者 macbook 本机键盘以及一些系统定义的按键点击事件。

这里自己封装了鼠标事件、鼠标滚动事件及键盘事件的方法，需要引入 ` <Carbon/Carbon.h>` 及 `<AppKit/AppKit.h>` 头文件

#### 1. 模拟鼠标事件：

```
void PostMouseEvent(CGMouseButton button, CGEventType type, const CGPoint &point, int64_t clickCount)
{
    CGEventSourceRef source = CGEventSourceCreate(kCGEventSourceStatePrivate);
    CGEventRef theEvent = CGEventCreateMouseEvent(source, type, point, button);
    CGEventSetIntegerValueField(theEvent, kCGMouseEventClickState, clickCount);
    CGEventSetType(theEvent, type);
    CGEventPost(kCGHIDEventTap, theEvent);
    CFRelease(theEvent);
    CFRelease(source);
}
```

**左键单击模拟：**

```
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseDown, CGPointZero, 1);
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseUp, CGPointZero, 1);
```

**左键双击模拟：**

```
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseDown, CGPointZero, 1);
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseUp, CGPointZero, 1);
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseDown, CGPointZero, 2);
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseUp, CGPointZero, 2);
```

**拖拽事件：** 如果是拖拽事件，例如左键拖拽事件，则需要先发送左键的 `kCGEventLeftMouseDown` 事件，然后连续发送 `kCGEventLeftMouseDragged` 事件，再发送 `kCGEventLeftMouseUp` 事件，代码如下：

```
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseDown, CGPointZero, 1);
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseDragged, CGPointZero, 1);
...
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseDragged, CGPointZero, 1);
PostMouseEvent(kCGMouseButtonLeft, kCGEventLeftMouseUp, CGPointZero, 1);
```

模拟其他鼠标事件，将枚举值修改一下即可。

#### 2. 模拟鼠标滚动事件

```
void PostScrollWheelEvent(int32_t scrollingDeltaX, int32_t scrollingDeltaY)
{
    CGEventSourceRef source = CGEventSourceCreate(kCGEventSourceStatePrivate);
    CGEventRef theEvent = CGEventCreateScrollWheelEvent(source, kCGScrollEventUnitPixel, 2, scrollingDeltaY, scrollingDeltaX);
    CGEventPost(kCGHIDEventTap, theEvent);
    CFRelease(theEvent);
    CFRelease(source);
}
```

鼠标滚轮事件只要传入水平和垂直方向的偏移即可实现。

#### 3. 模拟键盘事件

```
void PostKeyboardEvent(CGKeyCode virtualKey, bool keyDown, CGEventFlags flags)
{
    CGEventSourceRef source = CGEventSourceCreate(kCGEventSourceStatePrivate);
    CGEventRef push = CGEventCreateKeyboardEvent(source, virtualKey, keyDown);
    CGEventSetFlags(push, flags);
    CGEventPost(kCGHIDEventTap, push);
    CFRelease(push);
    CFRelease(source);
}
```

键盘事件的模拟需要注意的就是 *`CGEventFlags flags`* 参数，该参数用来模拟组合键的实现，类型定义如下：

```
typedef CF_OPTIONS(uint64_t, CGEventFlags) { /* Flags for events */
  /* Device-independent modifier key bits. */
  kCGEventFlagMaskAlphaShift =          NX_ALPHASHIFTMASK,
  kCGEventFlagMaskShift =               NX_SHIFTMASK,
  kCGEventFlagMaskControl =             NX_CONTROLMASK,
  kCGEventFlagMaskAlternate =           NX_ALTERNATEMASK,
  kCGEventFlagMaskCommand =             NX_COMMANDMASK,

  /* Special key identifiers. */
  kCGEventFlagMaskHelp =                NX_HELPMASK,
  kCGEventFlagMaskSecondaryFn =         NX_SECONDARYFNMASK,

  /* Identifies key events from numeric keypad area on extended keyboards. */
  kCGEventFlagMaskNumericPad =          NX_NUMERICPADMASK,

  /* Indicates if mouse/pen movement events are not being coalesced */
  kCGEventFlagMaskNonCoalesced =        NX_NONCOALSESCEDMASK
};
```

解析如下：

* kCGEventFlagMaskAlphaShift：大小写锁定键是否处于开启状态
* kCGEventFlagMaskShift：Shift 键是否按下
*  kCGEventFlagMaskControl：Control 键是否按下
*  kCGEventFlagMaskAlternate：Alt 键是否按下，对应 Mac 键盘的 option 键
*  kCGEventFlagMaskCommand：Command 键是否按下，对应 Windows 的 WIN 键
* kCGEventFlagMaskHelp：Help 键
* kCGEventFlagMaskSecondaryFn：Fn 键
* kCGEventFlagMaskNumericPad：数字键盘
* kCGEventFlagMaskNonCoalesced：没有任何键按下

如果有多个控制键同时按下，则使用位运算的或 `|` 加上对应的键值即可。例如模拟 `Command + Control + S`:

```
PostKeyboardEvent(kVK_ANSI_S, true, kCGEventFlagMaskCommand | kCGEventFlagMaskControl)
PostKeyboardEvent(kVK_ANSI_S, false, kCGEventFlagMaskNonCoalesced)
```

> 大小写锁定键，无法通过 kVK_CapsLock 按键的按下和抬起事件来模拟大小键的锁定，同时按键上的 LED 灯也是不会有变化的。

### 参考资料

* [鼠标和键盘事件](http://www.macdev.io/ebook/event.html)
* [macOS上模拟发送键盘事件](https://www.sunyazhou.com/2017/02/22/macOS-simulate-keyborad-NSEvent/)
* [开发mouseSync的初衷](http://zhihaozhang.github.io/2017/09/23/%E8%AE%A9iMac%E4%B8%8EMacBook%E9%AB%98%E6%95%88%E5%8D%8F%E5%90%8C%E5%B7%A5%E4%BD%9C%E2%80%94%E2%80%94mouseSync%E5%BC%80%E5%8F%91%E5%BF%83%E5%BE%97/)
* [监听Mac OS X的全局鼠标事件](https://blog.csdn.net/ch_soft/article/details/7371136)