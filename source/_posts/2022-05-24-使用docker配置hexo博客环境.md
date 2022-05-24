---
title: 使用docker配置hexo博客环境
date: 2022-05-24 22:19:21
tags: [linux, docker]
---

由于nodejs版本较多，直接在宿主机上安装nodejs环境不容易管理。因此可以使用docker维护一个专门用于hexo的环境。

<!-- more -->

## hexo直接安装

教程: https://hexo.io/zh-cn/docs/index.html

首先需要安装node.js和git，然后安装hexo

```
npm install -g hexo-cli
```

常用hexo命令

```
#建站
hexo init
npm install     #init后需要使用npm安装hexo所需依赖

hexo new [layout] <title>

hexo clean      #删除public目录(生成的html)
hexo s -p PORT	#运行server，对markdown的修改会实时反应
hexo g		    #生成html
hexo d		    #部署，或hexo d -g
```

## docker安装

主要过程：
1. 使用Dockerfile，基于node基镜像，创建含hexo的镜像
2. 运行该镜像，将blog目录映射到docker容器中，然后进行hexo操作
3. [可选]为了避免部署时输入github密码，可以将.ssh目录映射到容器中

### dockerfile

说明

- 参考的dockerfile：[使用docker搭建Hexo - wanf3ng's blog](https://wanf3ng.github.io/2021/01/29/使用docker搭建Hexo/)

- 设置用户问题：[在docker中執行hexo | Plusku@blog](https://blog.plusku.net/2020/08/08/docker-hexo/)
    - 默认了用户uid=1000。因此可以直接指定User node来避免容器内生成的文件为root所有问题。

- 添加了git设置

- 使用node12.0，否则hexo运行会报错

```
FROM node:12.6.0-alpine

# 切换中科大源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
# 安装bash git openssh
RUN apk add bash git openssh

# 设置容器时区为上海，不然发布文章的时间是国际时间，也就是比我们晚8个小时
RUN apk add tzdata && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
&& echo "Asia/Shanghai" > /etc/timezone \
&& apk del tzdata

# 安装hexo
RUN \ 
npm config set registry https://registry.npm.taobao.org \
&&npm install \
&&npm install hexo-cli -g \
&& npm install hexo-server --save \
&& npm install hexo-asset-image --save \
&& npm install hexo-wordcount --save \
&& npm install hexo-generator-sitemap --save \
&& npm install hexo-generator-baidu-sitemap --save \
&& npm install hexo-deployer-git --save

ARG GIT_NAME
ARG GIT_EMAIL

User node

# 设置git push时的用户名和邮箱
RUN \
	git config --global user.email "$GIT_EMAIL" \
	&& git config --global user.name "$GIT_NAME"

WORKDIR /home/node/blog

EXPOSE 4000

ENTRYPOINT ["/bin/bash"]
```

##### build镜像

```
docker build -t yfy/hexo-blog\
	--build-arg GIT_NAME="Fuyan Yuan"\
	--build-arg GIT_EMAIL="33221728+TheRainstorm@users.noreply.github.com"\
	.
```

##### 运行

```
docker run -it --name hexo-blog\
	-p 4000:4000 \
	-v /home/yfy/Documents/hexo-blog:/home/node/blog/ \
	-v /home/yfy/.ssh:/home/node/.ssh\
	yfy/hexo-blog
```

如果blog下没有node_modules，则需要

```
npm install
```

之后正常使用hexo即可

>  docker技巧：连接到已有容器。
>
> ```
> docker attach CONTAINER
> docker exec -it CONTAINER CMD
> ```
>
> `attach`命令将终端的标准输入，标准输出，标准错误连接到正在运行的容器。如果容器正在运行的命令是bash，则exit后，容器退出
>
> `exec`命令在容器内启动新的进程。常见如bash