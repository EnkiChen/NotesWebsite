---
title: 同步 WebRTC 主仓库
date: 2018-11-14 09:29:01
categories: [音视频/WebRTC]
tags: [WebRTC]
---

之前在 [WebRTC iOS&OSX 库的编译](http://www.enkichen.com/2017/05/12/webrtc-ios-build/) 介绍了如何下载和编译 WebRTC 的源码，但是很多时候可能只需要查看最新 WebRTC 的源码，并不需要下载庞大的第三方库以及编译依赖。

WebRTC 的仓库是使用 Git 来管理的，地址为 [https://chromium.googlesource.com/external/webrtc/](https://chromium.googlesource.com/external/webrtc/) 所以我们只需要 clone 到本地就好了（前提得备好梯子），如下命令：
<!--more-->

```
git clone https://chromium.googlesource.com/external/webrtc
```

同步到本地后，你会发现只有 `master`、`infra/config` 以及 `lkgr` 几个分支，其中 `master` 是官方的主开发分支了，每天都有大量的代码提交，但并没有类似的 `M70` 的分支或者 `tag`。

我们可以修改仓库的 `.git/config` 文件，在 `[remote "origin"]` 节中添加以下内容：

```
fetch = +refs/branch-heads/*:refs/remotes/origin/* 
```

修改后内容如下：

```
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
        ignorecase = true
        precomposeunicode = true
[remote "origin"]
        url = https://chromium.googlesource.com/external/webrtc
        fetch = +refs/heads/*:refs/remotes/origin/*
        fetch = +refs/branch-heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
```

之后就可以用 `git branch -av` 来查看远程分支了，当然也可以 `checkout` 到其他分支。