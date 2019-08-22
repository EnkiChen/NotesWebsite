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

### 使用脚本自动同步至 GitHub

WebRTC 的更新还是非常频繁的，可以使用 python 脚本将自动同步至 GitHub 上，首先在自己的 GitHub 空间中创建用于存储 webrtc 的仓库，然后在 webrtc 源码目录下执行以下命令，用于关联 GitHub 仓库：

```shell
$ git remote add github git@github.com:xxxx/webrtc.git 
```

这样 webrtc 就存在两个远程仓库，使用以下脚本来同步两个仓库，脚本如下：

```python
#!/usr/bin/env python3

import os
import subprocess
import re

remote_repo = 'github'
webrtc_path = "xxxx"

def execute_shell(cmd):
    print('$ ' + cmd)
    (status, output) = subprocess.getstatusoutput(cmd)
    if (len(output) > 0):
        print(output)
    return (status, output)


def sync_tag(name, commit):
    execute_shell('git tag -a m%s %s -m m%s' % (name, commit, name))
    execute_shell('git push %s --tags' % (remote_repo))


def sync_branch(name, commit):
    execute_shell('git checkout -b ' + name + ' ' + commit)
    execute_shell('git checkout ' + name)
    execute_shell('git pull origin ' + name)
    execute_shell('git push ' + remote_repo + ' ' + name)


def sync_repo(path):
    os.chdir(path)
    execute_shell('git checkout master')
    execute_shell('git fetch')
    execute_shell('git pull origin')
    (status, output) = subprocess.getstatusoutput('git br -av')
    if status != 0:
        print('webrtc path error with code:', status)
        return None

    pattern = re.compile(r'remotes/origin/(\S+)\s+([a-zA-Z0-9]{10})\s+', re.M)
    results = pattern.findall(output)
    for (branch, commit) in results:
        sync_branch(branch, commit)

    execute_shell('git checkout master')


def main():
    sync_repo(webrtc_path)


if __name__ == '__main__':
    main()
```

脚本中的 `remote_repo` 就是指 GitHub 的远程仓库的名字，`webrtc_path` 就是 webrtc 源码目录，完成配置后，执行脚本就可以同步两个仓库了，这可能需要点时间。我已经在我的 [GitHub](https://github.com/EnkiChen/webrtc) 空间中进行了同步，有需要的自取。