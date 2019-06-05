---
title: mac 平台 git 命令自动补全
date: 2019-06-05 09:19:37
categories: [知识整理/总结]
tags: [Git]
---

#### 安装 brew

如果有安装 brew 则跳过该步骤，执行如下命令进行安装：

```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

#### 安装 bash-completion

执行 `brew list` 查看是否安装了 `bash-completion` 如果没有则执行以下命令进行安装：

```
$ brew install bash-completion
```

<!--more--> 

#### 下载 git 源码

这里需要用到 git 仓库中一个文件，需要将 git clone 到本地，使用以下命令下载 git 仓库源码：

```
$ git clone https://github.com/git/git.git
```

下载完成后，需要将 *`contrib/completion/`* 目录下的 `git-completion.bash` 文件复制到当前用户根目录下：

```
$ cp git-completion.bash ~/.git-completion.bash
```

#### 配置环境

在 *`~/.bash_profile`* 文件（如果没有则需要手动创建）中添加以下代码：

```
if [ -f ~/.git-completion.bash ]; then
  . ~/.git-completion.bash
fi
```

在 *`~/.bashrc`* 文件（如果没有则需要手动创建）中添加以下内容：

```
source ~/.git-completion.bash
```

> ~/ 表示当前用户的根目录

重启终端后使用 tab 键来补全 git 命令。