title: iOS开发者证书以及代码签名学习笔记  
date: 2016-01-15 17:47:06  
categories: 知识整理/总结  
tags: [开发证书, 代码签名, mobileprovision, Provisioning Profiles, 开发者账号]
---

> 最近需要给iOS开发团队做一次关于iOS开发证书以及代码签名的分享，于是花了点时间把这一块的知识重新学习和整理了一遍，从而有了这篇学习笔记。其中很多一些文字都是从网站或者博客上摘抄过来，为了阅读方便也做了一些调整，说白了我只是做了一些知识的梳理和整合。  

> 该笔记涉及到内容有：开发者账号、签名证书、标识符（Identifiers）、设备（Devices）、APP授权机制、配置文件、ipa文件的签名和安全验证。

<!--more-->

### 开发者账号类型

苹果为iOS开发者提供三种账号类型，如下：

* **Apple Developer Program** 年费 $99(或¥688) 可以在iOS App Store和Mac App Store上架应用可以以个人（Individual）或者组织（Organization）的名义加入，以组织身份加入需要提供邓白氏编码（DUNS Number），会多出Team Management功能，允许多人协作开发。在发布署名上以组织（Organization）的名义加入可以填写公司或组织信息（比如某某公司、某某工作室），而以个人（Individual）加入只能默认显示注册时填写的个人信息，并且不能修改。
* **Apple Developer Enterprise Program**年费 $299，用于以InHouse方式发布企业内部应用，不能上架App Store企业证书过期则已经安装的应用无法继续运行。
* **iOS Developer University Program**
高校计划需要提供高校基本信息，免费提供。苹果为鼓励高校更多的参与到苹果开发者计划中来，特意推出这一项计划，高校计划具有在真机上测试等权限，但不能将App发布到App Store。

### 证书（Certificates）

什么是证书？证书就是：证明证书拥有者有证书上所说的能力，一个证书要涉及到颁发者、拥有者、证明拥有者有了什么能力。例如，CET-4证书；颁发者：学校，拥有者：自己，证明的能力：英语达到四级水平。苹果开发者证书也是一样，颁发者：自己，拥有者：安装证书的电脑；证明的能力：可以打包某应用程序。  

#### 开发者证书能力来源

向Member Center申请证书的过程，其实就是将在本地生成的`certSigningRequest`文件提交给苹果，让它进行签名授权的过程。`certSigningRequest`这个文件包含以下内容：

1. 申请者信息，此信息是用申请者的私钥加密的。
2. 申请者公钥，此信息是申请者使用的私钥对应的公钥。
3. 摘要算法和公钥加密算法。

当苹果用私钥对其签名（授权）之后，我们便可以获得一个证书文件。拥有该证书后，我们便可以用对应的私钥对APP签名了。当iOS设备拿到APP时便可以通过证书中的公钥来验证APP的正确性，同时iOS设备本身可以验证证书的是否被授权，因为该证书是苹果自己签名的证书。

> 被苹果签名的证书会随APP一起打包到ipa文件中，并提交到App store中。

当我们获得签名证书之后，还需要一个证书来验证证书是否被正确授权，该证书就是Worldwide Developer Relations Certificate Authority证书。该证书一般都会随Xcode一起安装到我们的电脑中，也可以从Member Center去下载。所以如果没有该证书，开发者将不能使用对应的私钥对APP的签名，因为不能确保证书是否被授权。该证书也就是网上有提到媒介证书（Intermediate Certificate）。

#### 证书的类型

苹果为开发者提供三种证书类型，用来在不同环境下使用，方便开发者的调试和测试。

* **开发证书**：平时用来进行真机调试的证书，用该证书签名的APP，只能安装在指定的设备上。
* **测试证书**：不可以用来真机调试的证书，但是可以编译到指定的真机上（不可以进行调试）。主要用来提交给测试进行功能的验证，和**开发证书**的区别在于，它和**发布证书**类似处于非沙盒坏境。但是用该证书签名的APP无法提交到App store，只能安装在指定设备上。  
* **发布证书**：不可以用来调试和测试，也不能安装在指定设备上，只能提交到App store。

> 使用**企业（Enterprise）**账号下的**发布证书**签名的APP可以安装到所以设备上，但是不能提交到App store。

### 标识符（Identifiers）

在Member Center中，Identifiers可以管理App IDs、Pass Type IDs、Website Push IDs、iCloud Containers、App Groups、Merchant IDs、这里主要介绍App IDs。  

App ID其实就是一个字符串，用来做APP唯一标识的字符串，App ID是大小写敏感的。一个APP有且只能有一个ID，并且唯一。在Project中称为Bundle ID（但是会有些小差别，Bundle ID不能包含**[ * ]**号）。在Member Center、Project、iTunes Connect都是需要此ID去标示此App的唯一性。App ID添加之后不能进行修改和删除。

#### App ID字符的组成和类型

![](http://www.enkichen.com/uploads/3.png)

如上图所示，App ID由Apple产生的一个Team ID作为前缀，后面跟的是开发者自定义的标识符，App ID字符串中只能包含字符（A-Z，a-z，0-9），连接符（-），点（.）而且此字符串最好是reverse-DNS格式的。例如你公司的域名是cctv.com，你App的名字是Hello，那么你可以用com.cctv.Hello作为你的Bundle ID。

App ID中也可以以**[ .* ]**来结尾，用来表示一个通配类型，如图：

![](http://www.enkichen.com/uploads/4.png)

* 精准类型的App ID：在标识符中不带**[ .* ]**来结尾的App ID可以称作为精准类型，该类型的App ID可以用来做APP的Bundle ID。
* 通配符类型App ID：在标识符中以**[ .* ]**结尾的App ID为通配符类型的App ID，该类型的App ID不能用来做APP的Bundle ID，其作用后续会讲到。

> 每个APP还会对应一串数字的字符串（在**itunesconnect**创建之后可以得到），通过该字符串可以向Apple提供的http接口（http://itunes.apple.com/lookup?id=**），获取对应的APP在App store上的信息，可以用来检测版本更新，更新的log一些其他资料。

#### App ID的作用

* 在Xcode工程中，Bundle ID储存在Info.plist中，当你编译工程的时候，他会把此文件拷贝到你的app包中。  
* 在iTunes Connect，用Bundle ID去标识App，在你第一次构建上传之后，你就不能在改变或者删除你的Bundle ID了。  
* 在Member Center，你创建一个和Bundle ID相匹配的App ID。如果App ID是精准类型的，你就必须精确的去匹配你的Bundle ID。  

### 授权机制 (Entitlements)

授权机制决定了哪些系统资源在什么情况下允许被一个应用使用。简单的说它就是一个沙盒的配置列表，上面列出了哪些行为被允许，哪些会被拒绝。Xcode 会将这个文件作为`--entitlements`参数的内容传给 **codesign**。

在 Xcode 的 Capabilities 选项卡下选择一些选项之后，Xcode 就会生成这样一段 XML。 Xcode 会自动生成一个 .entitlements 文件，然后在需要的时候往里面添加条目。当构建整个应用时，这个文件也会提交给 **codesign** 作为应用所需要拥有哪些授权的参考。这些授权信息必须都在开发者中心的 App ID 中启用，并且包含在配置文件中。

> 授权列表在Member Center中的**App ID**中配置，这样便可以对应到具体的APP。

### 设备（Devices）

这里的Device指的就是用来测试或者调试用的设备。可以是iPhone、iPad、iPod、Apple watch以及Apple TV，在Member Center中添加测试Device的步骤其实很简单，只要拿到对应Deveice的UDID就可以添加了。我们可以利用iTunes、iTools、Xcode这些工具都可以拿到设备的UDID。

> 需要注意的就是，每个开发者账号，每年最多可以添加100台调试设备，而且添加之后不能更改和删除，想要修改就要等到下一年重新续费的时候才能进行修改或者删除调试设备了。

### 配置文件（Provisioning Profiles）

上述提到了证书可以证明APP的所属以及APP的完整性，保证APP的本身的安全。但是却不能细化到APP所使用的服务被苹果认可，比如APN推送服务，并且证书无法限制调试版APP的装机规模。于是苹果想出了`mobileprovision `。一个`mobileprovision `文件包含一下内容：

1. **AppID** 这里的AppId可以是精准类型的也可以是通配符类型。
2. **证书列表** 在多人协议开发时，一个`mobileprovision `文件中可以包含多个证书文件。
3. **功能授权列表**
4. **可安装的设备列表** 测试和调试`mobileprovision `文件中包含设备列表，`mobileprovision `发布类型的文件中则不包含设备列表。
5. **苹果的签名**

> 上述提到的**苹果的签名**是用的苹果自己的私钥对应的公钥是Worldwide Developer Relations Certificate Authority证书（媒介证书）中的公钥，所以该文件生成后，我们是不能进行修改的，必须从Member Center中配置并生成。

#### 配置文件的区分

* 从`mobileprovision `文件中是否包含设备列表，可以分为带device信息的描述文件和不带device信息的描述文件如图：

![](http://www.enkichen.com/uploads/1451875454246244.png)  

![](http://www.enkichen.com/uploads/1451875469811263.png) 

* 也可以根据配置文件中包含的证书文件的类型来区分：**开发类型**、**测试类型**、**发布类型**。  

* 也可以根据配置文件中包含的App ID来做区分，如果文件中App ID是精准类型的，那么该配置只能用来对指定的APP进行使用。如果是通配类型的，那么该证书可以用来对匹配的Bundle ID的APP进行使用。如果是Company类型的开发者账号，可以生成一个供团队使用的Team Provisioning Profile，通过这个配置文件，团队内成员可以共用一个配置文件来进行开发调试，当然，App ID得指定成通配类型的。`mobileprovision `文件结构如下：

![](http://www.enkichen.com/uploads/5.png)  

总的来说描述文件就是整合了**证书**、**AppID**、**设备**以及**功能授权列表**，从而确定了可由哪台电脑，把哪个App，安装到哪台手机上面。

### APP的签名和安全验证过程

#### ipa文件的签名过程

这张图阐述了，开发iOS应用程序时，从申请证书，到打包的大致过程。

![](http://www.enkichen.com/uploads/iOS证书和校验.png)

#### ipa文件的组成

iOS程序最终都会以.ipa文件导出，ipa文件只是一个zip包，可以直接解压，先来了解一下ipa文件的结构：

![](http://www.enkichen.com/uploads/ipa组成.png)

解压后，得到上图的Payload目录，下面是个子目录，其中的内容如下：

* 资源文件，例如图片、html、等等。
* _CodeSignature/CodeResources。这是一个plist文件，可用文本查看，其中的内容就是是程序包中（不包括Frameworks）所有文件的签名。注意这里是所有文件。意味着你的程序一旦签名，就不能更改其中任何的东西，包括资源文件和可执行文件本身。iOS系统会检查这些签名。
* 可执行文件。此文件跟资源文件一样需要签名。
* 一个mobileprovision文件.打包的时候使用的，从MC上生成的。
* Frameworks。程序引用的非系统自带的Frameworks，每个Frameworks其实就是一个app，其中的结构应该和app差不多，也包含签名信息CodeResources文件。

#### ipa文件的安全验证过程

1. 解压ipa文件
2. 取出embedded.mobileprovision，通过签名校验是否被篡改过 a. 其中有几个证书的公钥，其中开发证书和发布证书用于校验签名 b. BundleId c. 授权列表
3. 校验所有文件的签名，包括Frameworks
4. 比对Info.plist里面的BundleId是否符合embedded.mobileprovision文件中的

### 其他涉及到的问题

* 团队开发证书的管理
* Xcode7下的免年费的真机调试

### 总结

当加入到苹果开发者计划之后，苹果通过证书来授权给开发者开发iOS应用，并提供了多种证书类型来满足不同的需求。为了保证APP的安全性和完整性，APP中所有的文件都将被签名。除非重新签名，否则不能对其做任何修改。

`mobileprovision`文件是一个配置文件，由苹果签名后发布给开发者的。其中包含了**证书**、**App ID**、**设备列表**、**授权列表**。通过这些信息从而确定了可由哪台电脑，把哪个App，安装到哪台手机上面。所以**证书**和`mobileprovision`文件是签名和打包的两个必要文件。

### 参考资料

* [**不让苹果开发者账号折磨我**](http://www.cocoachina.com/ios/20160104/14859.html)    
* [**苹果开发者账号那些事儿**](http://ryantang.me/blog/2013/09/03/apple-account-2/)  
* [**漫谈iOS程序的证书和签名机制**](http://www.pchou.info/ios/2015/12/14/ios-certification-and-code-sign.html)
* [**代码签名探析**](http://objccn.io/issue-17-2/)
* [**iOS Code Signing 学习笔记**](http://www.cocoachina.com/ios/20141017/9949.html)

