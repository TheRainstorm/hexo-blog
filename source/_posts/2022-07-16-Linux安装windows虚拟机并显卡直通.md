---
title: Linux安装windows虚拟机并显卡直通
date: 2022-07-16 00:42:11
tags:
---

### 说明

在linux上使用KVM安装windows虚拟机。然后将显卡直通(pci passthrough)进虚拟机，从而可以在Windows虚拟机上打游戏。

达到一台机器同时运行两个系统，充分利用硬件。
<!-- more -->

### 系统信息

- CPU: AMD Ryzen™ 7 5800X
- RAM: Gloway DDR4 2666MHz 32GB
- GPU slot1: Zotac GTX 1060 3g
- GPU slot2: Yeston RX550 2g
- 主板：ASUS TUF GAMING B550M-PLUS (WI-FI)
- M.2 slot1: Samsung PM9A1 1TB
- M.2 slot2: KIOXIA RC10 500GB
- SATA: 西数紫盘 2TB

#### 华硕B550m主板

官方参数：[TUF GAMING B550M-PLUS (WI-FI) - Tech Specs｜Motherboards｜ASUS Global](https://www.asus.com/Motherboards-Components/Motherboards/TUF-Gaming/TUF-GAMING-B550M-PLUS-WI-FI/techspec/)

- PCIe slots
  - 来自ryzen 5000 CPU（非G，否则为PCIE 3.0）
    - 1 x PCIe x16 (4.0x16 mode)
  - 来自B550 chipset
    - 1 x PCIe x16 (3.0x4 mode)
    - 1 x PCIe x1
      *1 PCIEX16_2 will run x2 mode when PCIEX1 is used.
- Storage
  - 2 M.2：CPU(PCIE4.0 x4), PCH(PCIE3.0 x4)
  - 4 SATA 6Gb/s：PCH

### 基本windows虚拟机

本部分创建一个可用的windows虚拟机，虽然没有显卡，但是也能完成很多事情，比如挂机录屏。

#### 通过virt-manger创建虚拟机

**libvirt**是一个开源的虚拟机管理API，可以管理KVM, Xen, VMvare, QEMU等虚拟化工具的虚拟机。包含库(libvirt)、命令行工具(virsh)、和virt-manger等GUI工具。参考：https://wiki.libvirt.org/page/FAQ

使用virt-manger创建win10虚拟机，详细步骤参考：[How To Install Windows 10 on Ubuntu KVM? – Getlabsdone.com](https://getlabsdone.com/install-windows-10-on-ubuntu-kvm/)。这里就不再详细介绍了，只提一些注意点。

##### 添加磁盘不同方式

添加磁盘有多种方式，不同方式的性能对比：https://www.youtube.com/watch?v=oSpGggczD2Y

- 创建qcow2文件
  - 可以选择virtio, SATA等总线协议。virtio性能最好，但是安装windows时需要额外安装驱动。
- 利用已有磁盘分区，同样可以选择virtio, SATA等总线协议。
- pci直通磁盘

##### SMB扩展存储

在linux上开一个smb服务，方便host和guest间文件传输。

由于可以直接运行smb中的程序，可以将一些程序文件都保存在smb的磁盘中。这样笔记本等其它设备也能直接访问。

*p.s. smb有些程序可以直接运行，有些无法运行（运行没反应），不知道是什么原因导致的。*

##### Windows RDP连接(更流畅)

- rdp连接效果比virt-manger的spice好很多

- 创建虚拟机时，默认会使用默认的网桥`virbr0`，而它是NAT的，有自己的地址段。host可以访问虚拟机，但是从host外无法访问虚拟机。

为了通过笔记本等其它设备访问虚拟机，可以使用ssh建立一个隧道（VNC教程中也有用到）

```bash
ssh -L 3389:192.168.122.253:3389 -C -N ryzen
```

或者使用下一节创建桥接网络的方法，LAN内其它设备直接访问虚拟机

#### 桥接网络

使用docker、lxc、libvirt时，都会创建默认的bridge设备

```bash
➜  ~ brctl show
bridge name     bridge id               STP enabled     interfaces
br-d1cfd67efd10         8000.0242abdcc8f5       no              veth461ccef
br0             8000.be316828842b       yes             enp7s0
                                                        vnet1
docker0         8000.0242ff18344e       no              veth2305c74
                                                        veth69f27d6
lxdbr0          8000.00163ecbf2fc       no
virbr0          8000.525400bf6a2c       yes             vnet0
```

bridge设备表示的是虚拟交换机，虚拟机通过bridge上网。如下图所示

<img src="https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220715122100204.png" alt="image-20220715122100204" style="zoom:67%;" />

libvirt创建不同类型网络参考：[Networking - Libvirt Wiki](https://wiki.libvirt.org/page/Networking)

##### bridge类型

bridge有一些不同的转发模式，有不同的用途。参考：[libvirt Networking Handbook — Jamie Nguyen (jamielinux.com)](https://jamielinux.com/docs/libvirt-networking-handbook/)

- nat，表示会做NAT转换
- route，不会做NAT转换，但是要求路由器知道如何将目的地址是bridge网段的包，路由给host
- bridge
  - 在真实的机器中，可以让一台电脑通过另一台电脑上网（解决寝室只有一个网口的问题）
  - 在虚拟机中，适合虚拟机提供服务，需要从外部访问的情况
  - Bridge和虚拟机共享一个真实的物理以太网设备。每个虚拟机能够直接获得LAN内的IPv4和IPv6地址，就像物理机一样。

##### 虚拟机使用桥接网络

虚拟机想要使用bridge模式来说，分为两步

- 在linux中创建bridge设备

- virt-manger中创建网络，使用该bridge设备

###### NetworkManger创建bridge

ubuntu使用NetworkManger管理网络，一些教程使用/etc/network/interfaces创建bridge设备已经不适用了

使用NetworkManger创建bridge[How to add network bridge with nmcli (NetworkManager) on Linux - nixCraft (cyberciti.biz)](https://www.cyberciti.biz/faq/how-to-add-network-bridge-with-nmcli-networkmanager-on-linux/)

```bash
nmcli con add ifname br0 type bridge con-name br0
nmcli con add type bridge-slave ifname enp7s0 master br0
```

**注意点**

- 启用了bridge设备时，需要关闭原有以太网设备。之后访问机器得通过br0。

  ```bash
  sudo nmcli con down "Wired connection 1"
  sudo nmcli con up br0
  ```

  - ssh连接时会导致连接断开，因此需要用脚本执行上面两条命令。
  - 新启用的br0会拥有LAN的ip地址，但是地址相较于原本地址会发生改变。可以在路由器中看到新的ip地址，然后ssh连接
    - 或者使用wifi维持另一个网络连接

- 无法在wifi设备上创建bridge，只能是有线以太网。

###### virt-manger XML

virt-manger中，在编辑->连接详情->虚拟网络添加一个网络

```xml
<network>
  <name>br0</name>
  <forward mode="bridge"/>
  <bridge name="br0" />
</network>
```

### 显卡直通

#### 参考

- KVM创建虚拟机并显卡直通详细博客：[Creating a Windows 10 kvm VM on the AMD Ryzen 9 3900X using VGA Passthrough - Heiko's Blog % Virtualization (heiko-sieger.info)](https://www.heiko-sieger.info/creating-a-windows-10-vm-on-the-amd-ryzen-9-3900x-using-qemu-4-0-and-vga-passthrough/)
- 最完整的Arch wiki：https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
- 帮助我解决了windows蓝屏问题的博客：https://blog.twenska.de/blog/GPU_passthrough/

#### 主要过程

PCI设备直通过程大概为

- bios开启**虚拟化**(intel: vt-x, AMD: SVM)和**iommu**(intel: vt-d, AMD: AMD-Vi)支持

  - 以我的AMD平台为例，需要开启SVM，IOMMU，ACS（位于AMD CBS中），BIOS中搜索关键词即可

- grub修改内核启动参数，开启iommu

  - 对于修改内核启动参数方式，grub2通过编辑/etc/default/grub，然后sudo update-grub

- 通过lspci命令，查看pci设备的总线地址，设备id，以及所处的IOMMU group

- 对于可热插拔的设备，在virt-manger中添加需要直通的host pci设备即可

  - 可热插拔的设备，虚拟机启动时vfio_pci接管设备驱动，关闭虚拟机时，host可重新访问

- 对于显卡这种无法热插拔的设备，需要在启动时将显卡绑定到vifi_pci驱动上。方法为添加内核启动参数

  最终我的内核启动参数示例

  ```bash
  GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on vfio_pci.ids=10de:1d01,10de:0fb8 kvm.ignore_msrs=1"
  ```

- 显卡处于虚拟化环境中会拒绝工作，需要在XML中添加额外参数https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Video_card_driver_virtualisation_detection

  ```
  <features>
    ...
    <hyperv>
      ...
      <vendor_id state='on' value='randomid'/>
      ...
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
    ...
  </features>
  ```

  - 比如对于Nvidia的显卡，会报43错误。在早期的驱动需要上面的vendor_id，但是最新的显卡驱动已经不需要了，所以更新显卡驱动便会正常工作。

#### IOMMU group

关于IOMMU和ACS：https://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/d/d6/MMU_and_IOMMU.svg/564px-MMU_and_IOMMU.svg.png" alt="img" style="zoom:50%;" />

- PCIE支持requst id，因此可以识别不同设备，每个设备可以使用自己的虚拟地址空间(I/O virtual address, IOVA)，IOMMU负责转换
- 然后设备之间还可以通过DMA直接peer to peer通信，这使得分离不同设备变得困难。PCIe Access Control Services (ACS) 便用于解决这个问题。它使我们能够知道设备间是否能够直接通信以及是否能够关闭。
- 对于不支持ACS的设备，IOMMU将其归为一个group。
- PCI直通必须直通一个IOMMU group的所有设备（除掉特殊的PCI root设备）

##### 我的主板iommu组分布

通过`lspci`可以列出host上的pci设备，使用下面脚本，得到我的主板上iommu组的信息

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

可知：

- 靠近CPU一侧的nvme插槽（PCIE 4.0 x 4）位于单独的14号组
- 位于靠近CPU的一侧的显卡插槽（PCIE 4.0 x16）位于单独的16号组
- 其余：显卡插槽2（PCIE 3.0 x4）、nvme插槽（PCIE 3.0 x 4）、以太网卡、USB、SATA均位于15号组

```
Group:  14  0000:01:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller PM9A1/PM9A3/980PRO [144d:a80a]   Driver: nvme

Group:  15  0000:02:00.0 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Device [1022:43ee]   Driver: xhci_hcd
Group:  15  0000:02:00.1 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] Device [1022:43eb]   Driver: ahci

Group:  15  0000:04:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin [Radeon RX 550 640SP / RX 560/560X] [1002:67ff] (rev ff)   Driver: amdgpu
Group:  15  0000:04:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin HDMI/DP Audio [Radeon RX 550 640SP / RX 560/560X] [1002:aae0]   Driver: snd_hda_intel

Group:  15  0000:05:00.0 Non-Volatile memory controller [0108]: KIOXIA Corporation NVMe SSD [1e0f:0009] (rev 01)   Driver: nvme

Group:  15  0000:06:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)   Driver: iwlwifi
Group:  15  0000:07:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 05)   Driver: r8169

Group:  16  0000:08:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP108 [GeForce GT 1030] [10de:1d01] (rev a1)   Driver: vfio-pci
Group:  16  0000:08:00.1 Audio device [0403]: NVIDIA Corporation GP108 High Definition Audio Controller [10de:0fb8] (rev a1)   Driver: vfio-pci
```

##### 一个group里有多个设备

主板上的不同PCIe slot可以连接到CPU上或者PCH(主板芯片组)上。我的主板貌似将所有设备都放入了一个group。因此我的显卡2和nvme固态盘等设备都无法直通。

对于一个组里有很多设备有一些解决方法：[IOMMU Groups - What You Need to Consider - Heiko's Blog - VFIO (heiko-sieger.info)](https://www.heiko-sieger.info/iommu-groups-what-you-need-to-consider/)

- 更新内核版本，新的内核版本可能对主板支持的更好，IOMMU group会改变
- 移动设备位置
- 安装kernel patch：https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)

但是我目前就选择直通slot 1显卡算了。

#### 直通slot1或slot2的权衡

直通slot1

- slot1显卡的风扇会被slot2显卡的PCB版挡住，散热不太好
  - 解决方法：使用PCIE显卡延长线（但是x16的太贵了，90RMB）
- boot gpu默认为slot 1的问题。BIOS无法设置primary display，导致windows虚拟机会蓝屏([PCI passthrough via OVMF - Passing_the_boot_GPU_to_the_guest](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_the_boot_GPU_to_the_guest)
  - 该问题已解决

直通slot2

- slot2和主板上的其它设备位于一组，无法直通slot2
  - 使用ACS patch
- 我B550的主板，slot2的虽然是x16长度的，但是只有x4 PCIE 3.0的带宽，显卡一般最少需要x8 PCIE 3.0，否则会对性能有影响，甚至无法工作
  - 但是一般来说双显卡时，两个槽应该都是工作在x8的模式下的吧。为何我的主板不支持将slot1的lane分给slot2呢

### 遇到的问题

#### RDP很卡

发现设备管理器中显示的是Microsoft基本显示适配器，在显示适配器上手动安装QXL驱动后解决（位于virtio光盘的qxlod目录下）

##### VNC与RDP

- 远程桌面RDP使用自己的remote display显示适配器，和一个通用非即插即用设备(uPnP)

- tightvnc则会使用QXL controller + PnP

  ![image-20220718124846271](https://raw.githubusercontent.com/TheRainstorm/.image-bed/main/picgo/image-20220718124846271.png)

- 后面发现貌似tightvnc和parsec远程连接（使用虚拟显示适配器）和VNC（使用显卡或者QXL）都会使用，优先VNC。

##### parsec连接玩游戏很糊

发现是显示器选择了125%的缩放导致的

#### steam link连接时windows用户登录弹窗

参考：[Can't get past Steam Input Dialog Box :: Steam Remote Play (steamcommunity.com)](https://steamcommunity.com/groups/homestream/discussions/0/1696049513769785227/)

问题为使用steam link连接时，windows需要登录。但是会弹出一个窗口，说：Would you like to accept secure desktop input from Steam? steam link远程登陆时无法点击系统弹窗，导致无法进入。弹窗也说需要坐在电脑前点击确认

解决：如果windows没有锁屏，那么steam link登录就不需要输入密码登录windows。可以在windows上安装tightvnc，通过vnc连接后，steam link连接时就不会输密码了（而且vnc连接也不会掉）

#### Windows虚拟机内OBS录屏

通过rdp远程连接windows时使用OBS录屏，会发现断开rdp连接后，录到的是黑屏。原因是rdp连接时，使用的是虚拟的显示适配器（因此无法调节分辨率），虚拟的声卡。当断开连接时，这些都会消失，故无法录屏。

解决办法为使用vnc，vnc连接后开启录屏，断开连接后，仍然在录屏。

#### Linux启动时黑屏

由于host gpu处于第二个槽，导致X启动失败

https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#X_does_not_start_after_enabling_vfio_pci

这里也提到了

https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Host_unable_to_boot_and_stuck_in_black_screen_after_enabling_vfio

解决

- kernel cmd

  ```
  video_vifib=off
  ```

- xorg

  ```
  /etc/X11/xorg.conf.d/second_gpu.conf
  Section "Device"
          Identifier "AMD GPU"
          Driver "amdgpu"		#填lspci看到的驱动
          BusID  "PCI:4:0:0"	#bus id, device id, function id
  EndSection
  ```

#### Windows启动时循环蓝屏

windows启动时蓝屏，蓝屏两次后进入恢复界面，选择关机后再次启动依然蓝屏。

##### 进一步发现

重启linux时，如果gpu 1上连着显示器，那么windows启动就会循环蓝屏。解决办法为先拔掉显示器，linux启动后再插上，之后启动VM-windows就不会蓝屏。

蓝屏时代码为

```
终止代码: VIDEO TDR FAILURE
失败的操作：nvlddmkm.sys
```

##### 解决

发现问题可能是由于将boot GPU直通导致的[PCI passthrough via OVMF - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_the_boot_GPU_to_the_guest)。但是我的主板没办法调节使用哪个GPU作为启动GPU

在https://blog.twenska.de/blog/GPU_passthrough/中看到类似问题，并提到解决方法

查看` /var/log/libvirt/qemu/win10.log.0`会发现大量的以下错误：

```bash
2022-07-21T06:50:56.222055Z qemu-system-x86_64: vfio_region_write(0000:08:00.0:region1+0x345990, 0x13801,8) failed: Device or resource busy
```

然后启动虚拟机前运行以下脚本即可

```bash
#!/bin/bash
echo 1 > /sys/bus/pci/devices/0000\:08\:00.0/remove && echo 1 > /sys/bus/pci/rescan
virsh start win10
```

发现Arch wiki提到相同的解决方法[BAR_3:_cannot_reserve[mem]](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#%22BAR_3:_cannot_reserve_[mem]%22_error_in_dmesg_after_starting_virtual_machine)

### 附录

#### 完整iommu group

```
➜  ~ ./iommu-viewer.sh 
Please be patient. This may take a couple seconds.
Group:  0   0000:00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  1   0000:00:01.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]   Driver: pcieport
Group:  2   0000:00:01.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]   Driver: pcieport
Group:  3   0000:00:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  4   0000:00:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  5   0000:00:03.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]   Driver: pcieport
Group:  6   0000:00:04.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  7   0000:00:05.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  8   0000:00:07.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  9   0000:00:07.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]   Driver: pcieport
Group:  10  0000:00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
Group:  11  0000:00:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Internal PCIe GPP Bridge 0 to bus[E:B] [1022:1484]   Driver: pcieport
Group:  12  0000:00:14.0 SMBus [0c05]: Advanced Micro Devices, Inc. [AMD] FCH SMBus Controller [1022:790b] (rev 61)   Driver: piix4_smbus
Group:  12  0000:00:14.3 ISA bridge [0601]: Advanced Micro Devices, Inc. [AMD] FCH LPC Bridge [1022:790e] (rev 51)
Group:  13  0000:00:18.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 0 [1022:1440]
Group:  13  0000:00:18.1 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 1 [1022:1441]
Group:  13  0000:00:18.2 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 2 [1022:1442]
Group:  13  0000:00:18.3 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 3 [1022:1443]   Driver: k10temp
Group:  13  0000:00:18.4 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 4 [1022:1444]
Group:  13  0000:00:18.5 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 5 [1022:1445]
Group:  13  0000:00:18.6 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 6 [1022:1446]
Group:  13  0000:00:18.7 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Matisse/Vermeer Data Fabric: Device 18h; Function 7 [1022:1447]
Group:  14  0000:01:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller PM9A1/PM9A3/980PRO [144d:a80a]   Driver: nvme
Group:  15  0000:02:00.0 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Device [1022:43ee]   Driver: xhci_hcd
Group:  15  0000:02:00.1 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] Device [1022:43eb]   Driver: ahci
Group:  15  0000:02:00.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:43e9]   Driver: pcieport
Group:  15  0000:03:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:43ea]   Driver: pcieport
Group:  15  0000:03:04.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:43ea]   Driver: pcieport
Group:  15  0000:03:08.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:43ea]   Driver: pcieport
Group:  15  0000:03:09.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Device [1022:43ea]   Driver: pcieport

Group:  15  0000:04:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin [Radeon RX 550 640SP / RX 560/560X] [1002:67ff] (rev ff)   Driver: amdgpu
Group:  15  0000:04:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin HDMI/DP Audio [Radeon RX 550 640SP / RX 560/560X] [1002:aae0]   Driver: snd_hda_intel

Group:  15  0000:05:00.0 Non-Volatile memory controller [0108]: KIOXIA Corporation NVMe SSD [1e0f:0009] (rev 01)   Driver: nvme
Group:  15  0000:06:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)   Driver: iwlwifi
Group:  15  0000:07:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8125 2.5GbE Controller [10ec:8125] (rev 05)   Driver: r8169

Group:  16  0000:08:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP108 [GeForce GT 1030] [10de:1d01] (rev a1)   Driver: nouveau
Group:  16  0000:08:00.1 Audio device [0403]: NVIDIA Corporation GP108 High Definition Audio Controller [10de:0fb8] (rev a1)   Driver: snd_hda_intel

Group:  17  0000:09:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
Group:  18  0000:0a:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
Group:  19  0000:0a:00.1 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Cryptographic Coprocessor PSPCPP [1022:1486]   Driver: ccp
Group:  20  0000:0a:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]   Driver: xhci_hcd
Group:  21  0000:0a:00.4 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse HD Audio Controller [1022:1487]   Driver: snd_hda_intel
```

