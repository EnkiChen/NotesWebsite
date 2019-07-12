---
title: macOS 开启 Web 服务器
date: 2019-07-11 15:46:25
categories: [知识整理/总结]
---

macOS 系统下自带了 Apache 和 PHP 环境，如果有需要的话，以下命令来控制服务器的启动、重启与停止：

```sh
# 启动 Apache 服务器
$sudo apachectl start

# 重启 Apache 服务器
$sudo apachectl restart

# 停止 Apache 服务器
$sudo apachectl stop
```

Apache 服务一些信息：

* Web 跟目录在： `/Library/WebServer/Documents`
* 配置文件路径：`/etc/apache2`

将文件放至 Web 根目录下用户即可方法，服务的运行端口的配置都可以在 `httpd.conf` 找到配置项。

开启 PHP 模块：

* 找到并打开配置文件 `/etc/apache2/httpd.conf` 
* 搜索 `libphp` 字样找到类似 `#LoadModule php7_module libexec/apache2/libphp7.so` 的配置项
* 去除前面的 `#` 字样来开启 PHP 模块
* 执行 `sudo apachectl restart` 命令来重启 Apache 服务器