---
title: Openwrt Mesh和快速漫游配置
date: 2022-08-25 22:17:13
tags: [折腾, openwrt, mesh]
---

### 背景

家里原本有两个路由器，一个负责楼上，一个负责楼下。但是仍然有许多覆盖不到的地方，比如厨房。并且更影响体验的是楼上楼下的WIFI使用不同的SSID，手机无法很好地自动切换。经常是楼上连接到楼下的网络导致信号很差。因此决定对家里的网络改造一番。

以前听过mesh这个技术，可以将很多台路由器通过无线连接起来，共同提供一个网络。于是去搜索了openwrt是否支持mesh，发现是可以的。并且经过进一步的了解，纠正了自己之前对无线网络的一些错误认知。

- 首先，mesh解决的并不是如何让设备自动切换网络，而是如何进行**无线组网**，可以避免布线的困难。mesh节点通过同一个信道的的无线相互连接，而通过另一个无线提供WIFI。
- **快速漫游**(fast roaming)协议，精确来说叫做802.11r协议，可以减少设备切换无线网络的耗时。

硬件设备上，我选择使用小米路由3G来搭建，主要是因为硬件配置上在wifi5的路由器中算是很不错的了，且某鱼上一个只需要50元。于是又买了两个，加上原本的，现在有4台openwrt路由器，一个红米AC2100作为主路由，3个R3G作为AP节点。

为了方便配置还实现了一个自动配置脚本放在github：[TheRainstorm/my-openwrt-config (github.com)](https://github.com/TheRainstorm/my-openwrt-config)

<!-- more -->
### 名词

- Wireless Access Point

  无线接入点，通常简称为AP。AP其实隐含dumb的含义。路由器设置里一般会有：路由器模式、AP模式等选项。AP模式下，路由器不提供路由、DHCP、DNS等功能，而是通过主路由来提供。

  - AP通常的作用是为现有的网络提供额外的无线覆盖，比如主路由器在二楼，通过AP将网络覆盖到一楼。

  - 实现上，AP将无线接口和LAN接口桥接在一起，可以理解为一个**二级交换机**。

- SSID、BSSID

  SSID是无线网络的名称，不同的AP可以提供相同的SSID。BSSID是无线网络的MAC地址，是唯一的。

### 无线漫游的过程

需要明白两件事

- 第一，设备倾向于赖在同一个网络上。当WIFI的信号强度弱于-70dB，且新WIFI的信号强度比原信号高8-12dB时才会选择切换网络。
- 第二，连接一个网络需要经过一些耗时过程。
  1. 需要进行密钥交换、协商加密算法
  2. 设备需要获得一个IP地址（通常通过DHCP协议）

第一个问题，理论上路由器可以给设备发送信号使其选择切换无线网络，不过不知道目前有没有这么做。

第二个问题

- 对于密钥交换和协商的过程很明显是可以避免的，802.11s协议应该就是节约了这部分的时间。只要配置成同一个域，那么在不同BSSID间切换就可以利用原有的连接。
- 而对于获得IP，只要路由器是AP模式，那么便可以直接使用原本的IP地址，因此也可以使用原本的IP地址。

### Openwrt配置

可以详细参考这篇文章：[OpenWrt 802.11s mesh | My wiki (bmaupin.github.io)](https://bmaupin.github.io/wiki/other/openwrt/openwrt-80211s.html)

自己写了一个自动配置脚本：[TheRainstorm/my-openwrt-config (github.com)](https://github.com/TheRainstorm/my-openwrt-config)

简单来说可以分为dumb AP的配置和WIFI的配置

#### 802.11s是否需要硬件支持

理论上不需要，但是有些硬件的wifi驱动可能会有问题（开源驱动基本不会有问题）
可以通过iw查看驱动是否有mesh选项。

```
iw list | grep "Supported interface modes" -A 9
```

#### 安装wpad

openwrt wifi功能由wpad包提供。openwrt提供了多种wpad包，针对不同功能进行了剪裁，大小不同。不同包会冲突，只能使用一个。使用wpad-mesh-openssl即可。

- wpad：最完整
- wpad-openssl
- wpad-mesh-openssl
- wpad-basic-openssl：精简

#### AP配置

官方文档：[[OpenWrt Wiki\] Wireless Access Point / Dumb Access Point](https://openwrt.org/docs/guide-user/network/wifi/dumbap)

##### 关闭DHCP服务器

ipv4勾选ignore interface即可

![image-20220825185126933](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825185126933.png)

ipv6需要将ra, dhcpv6, ndp均设置为disabled

![image-20220825185217075](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825185217075.png)

##### LAN接口ip地址

LAN接口需要设置ip地址，有两种方式（推荐第二种方法）

- 设置静态地址，需要同时设置网关、DNS为主路由

  ![image-20220825185519836](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825185519836.png)

  ![image-20220825185451076](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825185451076.png)

- 更简单的方式为将LAN设置为dhcp客户端。这样好处是不用配置，重启后路由器加入mesh后便可以成功上网。在之后可以在主路由DHCP设置中分配静态地址，这样路由器下次重启便会获得指定ip地址了。

  ![image-20220825185619578](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825185619578.png)

##### 关闭防火墙, dns等服务

system->starup中关闭一些不必要的服务，如firewall, dnsmasq, odhcpd

![image-20220825185835670](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825185835670.png)

主要部分已经配置完成。接下来是一些可选操作

- 删除掉WAN, WAN6等其余接口
- 删除掉所有防火墙zone

#### mesh无线配置

mesh需要占掉一个无线频段，这里我选择使用2.4GHz。因为家里的宽带是100MHz，因此2.4GHz的144Mbps带宽足够，然后就是2.4GHz的穿墙性能要好得多。

- 选择一个radio新建一个网络，比如2.4GHz radio

  ![image-20220825200934634](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825200934634.png)

- 设置WIFI信道，所有mesh节点需要使用相同信道

  ![image-20220825221157178](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825221157178.png)

- 然后然后选择mesh point模式。mesh id为一个字符串，保证mesh节点一致即可。加密必须选择wpa3-SAE加密。

  ![image-20220825200917273](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825200917273.png)

#### 无线802.11r配置

另一个radio用于提供上网，可以新建一个接口选择AP模式（默认已经存在）

![image-20220825220833532](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825220833532.png)

为了实现快速漫游需要设置802.11r，主要需要设置Mobility Domain，保证所有节点相同

![image-20220825220609260](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825220609260.png)

FT协议的区别，个人感觉不太明显。

### 效果

二楼是一台RM2100作为主路由，提供2.4GHz和5GHz WIFI。5GHz使用52信道。

一楼有3台R3G，通过mesh无线相互连接。其中一台通过网线连接到二楼RM2100。3台R3G启用5GHz WIFI名字相同，分别占用36信道，52信道，100信道。不同信道间刚好相差80MHz等于WIFI5的频宽，因此可以相互不干扰。

下图分别是二楼主路由和一楼一台路由的信道分析图

![image-20220825190223518](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825190223518.png)

![image-20220825214252429](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220825214252429.png)

#### 结果

在主路由上开启iperf3服务，手机上安装iperf3客户端测速，并在一楼和二楼走动。经验证确实可以自动实现网络漫游，走到某些位置时网速在降速后马上可以恢复。

### 参考

- 设置AP和快速漫游：[CHEAP WI-FI MESH ALTERNATIVE with fast roaming OpenWrt Wi-Fi Access points - YouTube](https://www.youtube.com/watch?v=kMgs2XFClaM)
- batman，更复杂也功能更多的mesh协议: [DIY MESH WiFi with batman-adv and OpenWrt - YouTube](https://www.youtube.com/watch?v=t4A0kfg2olo)
- 802.11k, v, r的介绍：[Fast Roaming with 802.11k, 802.11v, and 802.11r - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/network/fast-roaming-with-802-11k--802-11v--and-802-11r)
- FT over DS还是FT over Air[When does 802.11r "FT over DS" make sense to use? - Installing and Using OpenWrt / Network and Wireless Configuration - OpenWrt Forum](https://forum.openwrt.org/t/when-does-802-11r-ft-over-ds-make-sense-to-use/88893/2)