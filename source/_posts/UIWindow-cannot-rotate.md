title: 无法旋转的UIWindow子类
date: 2016-01-05 10:36:34
categories: 开发笔记
tags: [UIWindow, 旋转]
---

项目中一直用了两个**UIWindow**，一个是框架默认的，另一个是用来显示调试信息的**UIWindow**。至于为什么要用一个**UIWindow**，而不是用**UIView**呢，是因为**UIView**旋转控制太麻烦了，如果用**UIView**加一个**UIViewController**会方便很多，而且也不用考虑系统版本的区别。  
<!--more-->
最近一个同事用同样的方法使用**UIWindow**子类做一个提示框的公用组件，但发现`rootViewController`无法自动旋转。于是我对比了两个**UIWindow**的子类，发现代码并没什么区别，只是多了一些公用方法和一个`delegate`。但就是不能支持旋转，而我的都正常，试过好多方法，都没找到原因。最后只好一部分一部分注释掉一些代码。当我把一个`delegate`的属性注释掉以后，神奇的事情发生了，竟然可以旋转了！！！！ 于是把所有改动都还原了，然后只注释掉`delegate`属性（改一个名称也一样），发现旋转一切正常了。于是在我之前的**UIWindow**子类中添加`delegate`属性，发现也不能进行旋转。  

猜想是不是子类`delegate`属性覆盖了父类的`delegate`属性，而**UIWindow**在控制旋转的逻辑跟`delegate`属性有关。至少有一下两种方式进行调试：  

* 子类**UIWindow**中添加`delegate`属性，然后在setDelegate方法中添加日志。
* 使用Category为**UIWindow**添加一个`delegate`属性声明，然后打印`delegate`属性

通过上述方法可以发现，**UIWindow**内部实现是存在一个`delegate`属性的，根据日志可以确定该属性是指向`rootViewController`对象。也就是说如果我们在**UIWindow**子类添加`delegate`属性就会影响到**UIWindow**对旋转的控制。  

后续调试发现旋转的相关方法在`rootViewController`中都会正常被调用，但就是没办法进行旋转，也就是说在**UIWindow**子类中添加`delegate`属性和没有添加该属性，对应我们应用来说，该走的方法都会按照正常流程跑完，但就是不能进行旋转。具体原因还未知，后续有新发现将会更新，如果有人知道，还请告知，谢谢，以下我的调试环境：  

> OSX版本为：10.10.5 (14F27)  
> Xcode 版本为：7.2 (7C68)   
> SDK：iOS 9.2  
