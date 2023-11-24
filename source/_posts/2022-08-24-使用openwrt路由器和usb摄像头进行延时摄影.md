---
title: 使用openwrt路由器和usb摄像头进行延时摄影
date: 2022-08-24 14:13:52
tags:
- openwrt
- usb摄像头
- 延时摄影
categories:
- 折腾
---

### 作用

放在楼顶，拍摄一天24h的户外录像、延时摄影。像超级小桀每天的直播录屏p1那样。

- 可以远程通过网页实时查看摄像头直播画面

<!-- more -->

### 问题

- 树莓派+usb摄像头手动DIY？
  - 能否旧手机改造（摄像头画质秒杀）
  - 能否使用路由器替代树莓派（有一个闲置的小米路由器3G，带USB 3.0口）
    - [USB camera on router with OpenWrt and MJPG-Streamer and shinobi cctv on docker in your home network - YouTube](https://www.youtube.com/watch?v=KlfS-oO_2Sw)
    - MIPS mtk7261串流，CPU性能是否足够（MIPS® 1004KEc™ dual-core CPU @880MHz）
      - [OpenWrt Forum Archive](https://forum.archive.openwrt.org/viewtopic.php?id=60201&p=1#p296414)：
        - TL-MR3020: MIPS 24kc@400MHz
        - 720p@30下CPU负载19-32%
      - [MIPS32 - DeviWiki (ex WikiDevi)](https://deviwiki.com/wiki/MIPS32#MIPS32_Classic_Processor_Cores)
        - 24kc: 3.1coremark/MHz
        - 1004KEc: 3.05 coremark/MHz
- 好奇淘宝上169的网络监控摄像头产品内部使用的是什么CPU
  - 可以手机app远程查看
  - **有云台**，可以调整视角
  - 居然还有太阳能供电版本，功耗很低？
- 超级小桀那样的画面需要多高分辨率（**需要怎样的摄像头**）
  - 不知道拍天空户外，普通的监控摄像头画面效果
  - 是否需要广角摄像头
- 能否集成温度，湿度监控。或者如何通过网络获取，显示在画面左上角。

### 了解

- usb摄像头，很便宜（淘宝20元）
  - 需要支持UVC免驱
- openwrt使用kmod-video-uvc驱动usb摄像头。串流
  - MJPG-Streamer
  - ffmpeg/ffserver
  - gstreamer
- usb摄像头支持什么格式编码
  - YUY2：没压缩，对上位机性能要求高
  - MJPEG 带宽要求高 1080p@30 1680 - 2700 kB/s
- webcam含义
  - [Webcam - Wikipedia](https://en.wikipedia.org/wiki/Webcam)
  - 早期表示低成本的低端的摄像头，如笔记本前置摄像头。但如今基本都为720p以上了
  - 90度FOV用于家庭和直播足够。更大的适合中大型会议室

### 摄像头

- 笔记本摄像头效果：1280x720 

  ![image-20220814120138521](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220814120138521.png)

  ![屏幕截图(224)](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE(224).png)

- 69 1080p摄像头

  ![image-20220814223205817](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220814223205817.png)

- [Review of the ELP-USBFHD01M camera (joancharmant.com)](http://joancharmant.com/blog/review-of-the-elp-usbfhd01m-camera/)

- 罗技C920: 400。直播摄像头，1080p，300万

- 罗技C270i: 100。日常视频。720p

- elgato: https://www.bilibili.com/video/BV1gY411s7zH

- https://www.bilibili.com/video/BV1WN4y157hA

### 基本方案

- host：mi r3g
- usb摄像头：

- usb摄像头模块
  - ![image-20220814192037749](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220814192037749.png)

软件

- openwrt:[[OpenWrt Wiki\] USB Video Support](https://openwrt.org/docs/guide-user/hardware/video/usb.video)
  - motion: [Motion (motion-project.github.io)](https://motion-project.github.io/index.html)
    - live stream
    - timelapse video
    - many type device
      - network camera via RTSP, RTMP, HTTP
      - V4L2 webcams
      - Video capture cards
      - Existing movie files

### 测试

- 640x480@30 BW=25Mb, CPU 单核<10%，3线程总<20%，待机6%
- 1080p @30 BW=53Mb, CPU单核22%, 总 44%，待机20%
- **1280x720 @30 BW=36Mb，CPU 30%，待机10%**
  - 画面还行，比较合适

### 监控

[如何选择安防监控摄像机？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/37515369)