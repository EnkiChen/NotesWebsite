title: iOS 运行时之 Associative(关联) 
date: 2017-02-18 12:42:00  
categories: [知识整理/总结]  
tags: [associative, runtime]
---

iOS 下有很多运行时特性，这里介绍一下 `Associative(关联)` 这个运行时特性，以及它一些使用场景。`Associative` 意思为关联，能够将两个对象建立一种关系。这种关系是一种 `从属` 关系，也就是说有一个 `关联者` 和一个 `被关联者`。比如说我们可以将一个 `NSString` 对象关联到一个 `UIView` 对象上。这里的 `NSString` 对象就是 `被关联者`, `UIView` 对象就是 `关联者`。

在 `objc/runtime.h` 文件中，找到 `Associative ` 相关的 API 定义，如下：

```
/** 
 * Sets an associated value for a given object using a given key and association policy.
 * 
 * @param object The source object for the association.
 * @param key The key for the association.
 * @param value The value to associate with the key key for object. Pass nil to clear an existing association.
 * @param policy The policy for the association. For possible values, see “Associative Object Behaviors.”
 * 
 * @see objc_setAssociatedObject
 * @see objc_removeAssociatedObjects
 */
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);

/** 
 * Returns the value associated with a given object for a given key.
 * 
 * @param object The source object for the association.
 * @param key The key for the association.
 * 
 * @return The value associated with the key \e key for \e object.
 * 
 * @see objc_setAssociatedObject
 */
OBJC_EXPORT id objc_getAssociatedObject(id object, const void *key)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);

/** 
 * Removes all associations for a given object.
 * 
 * @param object An object that maintains associated objects.
 * 
 * @note The main purpose of this function is to make it easy to return an object 
 *  to a "pristine state”. You should not use this function for general removal of
 *  associations from objects, since it also removes associations that other clients
 *  may have added to the object. Typically you should use \c objc_setAssociatedObject 
 *  with a nil value to clear an association.
 * 
 * @see objc_setAssociatedObject
 * @see objc_getAssociatedObject
 */
OBJC_EXPORT void objc_removeAssociatedObjects(id object)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);

```
<!--more-->
同时还提供以下枚举类型的定义：

```
/**
 * Policies related to associative references.
 * These are options to objc_setAssociatedObject()
 */
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

### API 解析

`void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)` API 为我们提供了将两个对象建立关联关系的能力，参数解析为：

* **id object**：指定 `关联者`
* **id value**：指定 `被关联者`
* **const void *key**：`被关联者` 的 KEY 值，方便后续可以通过该 KEY 值找到该 `被关联者`
* **objc_AssociationPolicy policy**: 该参数作用用来表示 `被关联者` 的引用策略，也就是内存如何进行管理的，可通过上述定义的枚举类型来设置。

`id objc_getAssociatedObject(id object, const void *key)` API 可以通过之前设置的 KEY 值，来获取 `被关联者` 对象，参数解析如下：

* **id object**：`关联者` 对象
* **const void *key**：要获取的 `被关联者` 的 KEY 值，一个 `关联者` 可以被关联多对象，一个 `关联者` 也可以是 `被关联这`，可以通过不同的 KEY 来获取不同的 `被关联者` 对象。

`void objc_removeAssociatedObjects(id object)` 该 API 可以移除一个 `关联者` 对象所有的 `被关联者`。当需要移除特定的对象时，我们可以使用 `objc_setAssociatedObject ` 方法并指定 `id value` 参数对象为空即可。

以上就是关于 `Associative(关联)` 特性相关的 API 介绍了，下面介绍一下常用的使用场景。

### Associative 特性的应用

#### 剪切板的信息复制

在一些时候我们希望用户可以长按文案信息，弹出系统的复制菜单，提供文案信息的复制功能，比如长按 	`UITableViewCell` 提供复制详情的功能，在 iOS 下我们可以使用 `UIMenuController` 类来显示系统菜单，同时为该 `UITableViewCell` 添加长按手势，代码如下：

```
// 添加长按手势
UILongPressGestureRecognizer *longPressGR = [[UILongPressGestureRecognizer alloc] initWithTarget:self action:@selector(handleLongPress:)];
[cell addGestureRecognizer:longPressGR];

// 手势处理
- (void)handleLongPress:(UILongPressGestureRecognizer *) longPressGR {
    UIMenuController *menu = [UIMenuController sharedMenuController];
    [menu setTargetRect:longPressGR.view.frame inView:self.view];
    [menu setMenuVisible:YES animated:YES];
}

// UIMenuController 相关
- (BOOL)canBecomeFirstResponder {
    return YES;
}

- (BOOL)canPerformAction:(SEL)action withSender:(id)sender {
    if ( action == @selector(copy:) ) {
        return YES;
    }
    return NO;
}

- (void)copy:(UIMenuController *)menu {
    UIPasteboard *pasteboard = [UIPasteboard generalPasteboard];
}
```

从上述代码可以看到，复制信息的逻辑处理是在 `copy:` 方法中，但是在该方法中，并不能访问到 `cell.detailTextLabel` 对象，在该场景中，我们可以使用 `Associative` 特性将 `UITableViewCell` 对象关联到 `UIMenuController` 对象中，再在 `copy:` 方法中获取到被关联对象，从而获取到 `UITableViewCell` 对象，进而访问 `cell.detailTextLabel.text`。添加代码如下：

```
// 处理手势时，添加如下代码
objc_setAssociatedObject(menu, @"UITableViewCell", longPressGR.view, OBJC_ASSOCIATION_RETAIN_NONATOMIC);

// 处理 copy 时，添加如下代码，来获取被关联的 UITableViewCell 对象
objc_getAssociatedObject(menu, @"UITableViewCell");
pasteboard.string = cell.detailTextLabel.text;
```

上述场景中，并不一定非得用 `Associative` 特性来实现，还有很多可行的方法，这里为大家提供一种方法，并且该方法还算是比较优雅的。

#### 其他一些应用场景

另一个常见的应用场景就是，为一个系统类或是一个第三方的类添加一个属性时，可以结合 **Category** 为类添加一个属性，当然也可以使用继承来达到目的。在一些特殊场景下，比如想知道一个系统内部对象或者第三方对象是何时被释放时，我们可以为该对象关联一个自定义的对象，并且使用 `OBJC_ASSOCIATION_RETAIN_NONATOMIC ` 来指定内存管理策略，当关联者被释放是，被关联者也会跟着被释放，这样可以在我们自定义的对象中，知道感兴趣的对象何时被释放的。在调试一些内存问题时，该方法还是蛮有用的。