title: HTTP 之 Content-Type
date: 2016-08-29 20:27:25
categories: [开发笔记]
tags: [HTTP]
---

在移动端开发中 HTTP 协议经常被用到，但是在面试或者工作中问到客户端和服务器传输业务数据，使用的数据的封装格式是什么以及怎么被封装到 HTTP 协议中时，很少有人能讲明白。

在移动开发中 HTTP 协议为客户端到服务器端，提供了一条数据通道，能够将我们的业务数据传输到服务器，并且从服务器上获取响应数据。在 HTTP 协议头中的 **Content-Type** 字段描述了 **HTTP** 的 **BODY** 体的数据格式，而在 **BODY** 中可以定义我们业务数据的数据格式。**Content-Type** 字段可以用很多种类型，具体有哪些可以看 [**这里**](http://tool.oschina.net/commons)，并且也可以根据自身业务来自定义，不过这种做法比较少。这次我主要讲解三种格式分别为 **application/x-www-form-urlencoded**、**application/json** 以及 **multipart/form-data**，其他的格式可以自己理解。

下面给各位吃瓜群众介绍上述三种格式在 **GET** 和 **POST** 两种请求方法中的区别，为了更直观的看到我们的数据的组织方式，我使用了 **Charles** 来进行抓包分析。  
<!--more-->   
我们测试的业务需求为：

1. 客户端发起登录请求，服务器进行响应，请求的数据为 **user** 字段值为 **admin**； **pass** 字段值为 **123456**；
2. 客户端上传一张图片到服务器，字段名为 **imageFile**，内容为图片的二进制数据。

为了更好的测试，我对 Web 的请求基于 AFNetworking 2.5.0 版本做了一层简单的封装，发起请求的代码：

```
- (void)webRequest
{
    NSDictionary *bParams = @{ @"user" : @"admin",
                               @"pass" : @"123456"};
    
    NSDictionary *params = @{ kWebServiceMethod     : @"POST",
                              kWebServiceParams     : bParams,
                              kWebServiceIdentifier : kPlatformAccountService,
                              kWebServiceApiUrl     : @"login"};
    
    [CMLWebProxyService webRequestWithParam:params completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
        DDLogInfo(@"responseObject:%@", responseObject);
    }];
}

```

用来指定数据格式的代码封装在了叫 BaseService 的类中，方法如下：

```
- (NSString *)apiContentType
{
    return @"application/x-www-form-urlencoded";
}
```

Web 请求的封装的代码（基于 AFNetworking 2.5.0 版本）

```
+ (NSURLSessionDataTask *)webRequestWithParam:(NSDictionary *) params completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error)) completionHandler
{
    id<CMLService> service = [[CMLServiceFactory sharedInstance] serviceWithIdentifier:params[kWebServiceIdentifier]];
    NSString *urlString = [NSString stringWithFormat:@"%@%@/%@", service.apiBaseUrl, service.apiVersion, params[kWebServiceApiUrl]];
    AFHTTPRequestSerializer *requestSerializer = nil;
    
    if ( [service.apiContentType isEqualToString:@"application/x-www-form-urlencoded"] ) {
        
        requestSerializer = [AFHTTPRequestSerializer serializer];
        
    } else if ( [service.apiContentType isEqualToString:@"application/json"] ) {
        
        requestSerializer = [AFJSONRequestSerializer serializer];
    }
    
    [requestSerializer setValue:@"application/json" forHTTPHeaderField:@"Accept"];
    requestSerializer.timeoutInterval = kWebServiceTimeoutInterval;
    
    NSDictionary *headParams = params[kWebServiceHeadParams];
    [headParams enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
        [requestSerializer setValue:obj forHTTPHeaderField:key];
    }];
    
    NSURLRequest *request = nil;·
    NSArray *binParams = params[kWebServiceBinParams];
    if ( binParams != nil && binParams.count != 0 ) {
        request = [requestSerializer multipartFormRequestWithMethod:params[kWebServiceMethod]
                                                          URLString:urlString
                                                         parameters:params[kWebServiceParams]
                                          constructingBodyWithBlock:^(id<AFMultipartFormData> formData) {
                                              
                                              for ( NSDictionary *binData in binParams ) {
                                                  [formData appendPartWithFileData:binData[kWebBinData]
                                                                              name:binData[kWebBinName]
                                                                          fileName:binData[kWebBinFileName]
                                                                          mimeType:binData[kWebBinMimeType]];
                                              }
                                              
                                          } error:nil];
    } else {
        request = [requestSerializer requestWithMethod:params[kWebServiceMethod] URLString:urlString parameters:params[kWebServiceParams] error:nil];
    }
    
    NSURLSessionDataTask *dataTsk = [[CMLWebProxyService sessionManager] dataTaskWithRequest:request completionHandler:completionHandler];
    
    [dataTsk resume];
    
    return dataTsk;
}

```

### POST 下的数据组织形式

先看看各个格式的抓包的截图，在做对比分析。

**application/x-www-form-urlencoded** 格式的抓包截图

![url 编码格式](/uploads/http_content_type_1.png)

**application/json** 格式的抓包截图

![url 编码格式](/uploads/http_content_type_2.png)

**multipart/form-data** 格式的请求代码以及抓包的截图：

```
	NSDictionary *bParams = @{ @"user" : @"admin",
                               @"pass" : @"123456"};
    
    NSData *imageBin = UIImagePNGRepresentation([UIImage imageNamed:@"avatar_cat"]);
    
    NSDictionary *binPar = @{ kWebBinData : imageBin,
                              kWebBinName : @"imageFile",
                              kWebBinFileName : @"image.png",
                              kWebBinMimeType : @"image/png"
                             };
    
    NSDictionary *params = @{ kWebServiceMethod     : @"POST",
                              kWebServiceParams     : bParams,
                              kWebServiceBinParams  : @[binPar],
                              kWebServiceIdentifier : kPlatformAccountService,
                              kWebServiceApiUrl     : @"login"};
                              
```

![url 编码格式](/uploads/http_content_type_3.png)

**application/x-www-form-urlencoded** 格式对请求参数进行了 [**URL 编码**](http://deyimsf.iteye.com/blog/1776082) 并将参数放在了 **BODY** 中；**application/json** 格式是将参数进行了 [**JSON**](http://www.w3school.com.cn/json/json_syntax.asp) 格式的转换，放在了 **BODY** 中；

前两种都是传的普通的文本数据，如果我们需要同时传输文本数据以及二进制数据时，就得用到 **multipart/form-data** 编码格式了，可以从图中看到在 **HTTP** 的头部的 **Content-Type** 中多了 **Boundary** 字段，该字段的作用为对多项数据进行分割；截图中 3 个参数都使用了指定的字符串进行了分割，在图片的那一项中也包含了 **Content-Type** 字段用来描述该项的数据格式为 **PNG** 图片。如果没有图片参数的话，也可以直接使用 **multipart/form-data** 格式来进行组织。

> 用来分割的字符串的值是每次随机生成的。不同的客户生成的字符串长度也会不一样。

### GET 下的数据组织形式

**HTTP** 的 **GET** 请求方式，是将我们要请求的参数放在 **URL** 地址后面，所以 **GET** 方式只支持 **application/x-www-form-urlencoded** 格式，而 **application/json** 和 **multipart/form-data** 不支持的，这个应该比较容易理解。下面是我抓包的截图：

![url 编码格式](/uploads/http_content_type_4.png)

### 总结

其实抓包截图做对比就很容易理解不同的参数的意义了，在很多情况下后端和前端开发人员都是使用成熟的第三方框架来帮我们做这部分工作，对 **HTTP** 协议了解的还不够细致，有时候发现在调试接口无法解析数据，其实很有可能 **Content-Type** 类型不一致导致的。如果觉得文本协议不够精简，也可以使用二进制协议来传输 **user** 和 **pass** 字段，比如 **protobuf** ，只要和服务器端协商好就可以。