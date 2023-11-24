---
title: 安装centos虚拟机以及配置gameboy.live记录
date: 2020-02-02 19:08:52
tags:
- centos
- gameboy
categories:
- 折腾
---

安装centos虚拟机以及配置gameboy.live记录
<!-- more -->

## 1. 下载镜像

在清华源等镜像源都可以，我下载的是everything版本（有10G左右）

## 2. VirtualBox安装CentOS

需要选择安装那些软件（默认是minimalist)，我选择了手动，然后选择了网络相关的（默认有sshd)。然后设置root密码，新建初始账户。

## 3. 配置（换源）

1. 发现刚安装的centos没办法使用yum（repository没有配置好）。找到如何换源的方法，主要分为本地源（iso镜像里有一个Packages，包含官方的所有软件(rpm包)，10000个左右）和网络源。

2. 先安装下VBox扩展，

   ```bash
   mkdir /mnt/cdrom
   mount /dev/cdrom /mnt/cdrom
   cd /mnt/cdrom
   ./VBoxLinuxAdditions.run (中途会编译)
   ```

   之后可以设置共享文件夹，剪切板等。

3. 安装本地源：

   1. 挂载centos系统镜像（everything那个）:

      先在虚拟机设置里存储，IDE控制器那分配光驱，把iso添加进去。

      ```
      mkdir /media/cdrom
      mount /dev/cdrom /media/cdrom
      ```

   2. 修改源配置文件

      - /etc/yum.config包含main配置，在/etc/yum.repos.d/下为各种源（本地源CentOS-Media.repo，网络源CentOS-Base.repo）

      - 由于/media/cdrom已经在baseurl里有了，故只需将enabled设为1即可

      - 将CenOS-Base.repo重命名为其它

   3. 使之生效

      ```
      yum clean all
      yum makecache
      ```

      愉快地安装vim, git, python吧。

      不知道如何安装netstat，ifconfig？直接yum search xxx会告诉你答案。
   
4. 设置镜像源

   阿里云镜像比较齐全，且有帮助文档。 https://developer.aliyun.com/mirror/ 

## 4 配置gameboy.live

### 1. 安装golang

yum install 会显示没有golang，或者版本不够新

在官网上有已经编译好的golang，直接下载解压就可以使用。

```bash
wget https://dl.google.com/go/go1.13.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
添加PATH变量

go version #查看版本
```

   tips: 关于linux安装路径：

- /usr目录下为系统软件目录，有/usr/bin, /usr/lib等文件夹。相当于windows的C://windows/
- /usr/local为用户软件目录，也有/usr/local/bin, /usr/local/lib等。用户安装的软件默认会安装在这里。相当于C://Program Files/
- /opt，代表可选软件，如firefox等独立的大型软件可以安装在这里。相当于D://

### 2. 配置golang代理

直接go get很容易超时。

```bash
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

   可以查看https://goproxy.cn官网介绍

   

### 3. 安装x11

直接go编译gameboy.live会报各种.h文件找不到。以下

```bash

# https://github.com/go-gl/glfw
...

yum install libX11-devel libXcursor-devel libXrandr-devel libXinerama-devel mesa-libGL-devel libXi-devel


# github.com/hajimehoshi/oto
../../go/pkg/mod/github.com/hajimehoshi/oto@v0.3.1/driver_linux.go:23:28: fatal error: alsa/asoundlib.h: No such file or directory

yum install alsa-lib-devel (ubuntu下的安装libasound2就好了，yum下好不容易才找到可以安装这个代替libasound2。)
```



#### 4. 运行

- 发现由于mobaxterm自带x11server，因此在阿里云上运行，便会自动打开图形窗口。

- 发现在putty上可以正确显示，但按键只有Enter起作用，ctrl+z等可以产生Right的效果。
- 在VB虚拟机（无图形界面）上无法正确显示画面。WSL上也不行。
- 在ubuntu桌面版的终端可以完美运行。

#### 5. 结束ssh后不停止运行

 https://blog.csdn.net/v1v1wang/article/details/6855552 

