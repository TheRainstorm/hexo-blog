---
title: RISC-V trend
date: 2020-01-07 22:24:46
tags:
description: 计算机组成原理的期末作业，在RISC-V精简指令集、异构计算等领域中选择一个并查阅相关资料写一篇报告。
---
# RISC-V报告

author: 计科卓越班 袁福焱 20174260

## 0. 导言

RISC-V是21世纪诞生的一种完全开放的计算机精简指令集，可应用于小到嵌入式系统大到高性能处理器等广泛领域。RISC-V充分汲取了许多指令集的实践经验，具有非常多的优点。并且它完全开放，采用BSD开源协议——全世界任何公司、大学、个人都可以自由免费地使用RISC-V指令集构建自己的系统。

本报告第一部分讲述RISC-V的几个主要特点与优势，第二部分介绍RISC-V的目前已构建建的生态系统。最后一部分，谈论RISC-V在推动开源硬件中起到的作用。

## 1. RISC-V的特点

###  1.1 指令集模块化

以往的指令集为了实现兼容，往往采用增量扩展的方法。即新的处理器不仅要实现新的扩展，还必需要实现过去所有的扩展。如典型的x86架构，其中许多指令早已失效，但为了兼容性却必须实现，这导致硬件实现变得越来越复杂。RISC-V按照功能将指令集分为很多个模块。其中I模块为基础整型指令集，这是RISC-V要求必须实现的指令集，并且永远不会发生改变。只要实现了该指令集便可以运行完整的基于RISC-V的软件。这样，RISC-V在扩展时便不再会像x86那样背上沉重的历史包裹。编译器也可以在保证兼容的基础上，根据已实现的扩展，选择性地生成更高效的代码。

### 1.2 回避以往的设计缺陷

目前主流的指令集有x86, ARM, MIPS等。其中x86为复杂指令集(RISC)，于1978年诞生。而ARM, MIPS为精简指令集(RISC)，都是1986年诞生的。从计算机体系结构的发展上来看，RISC由于简洁性，能够更加充分地利用现代微体系结构中的流水线、分支预测、cache等技术，因此比CISC更加高效。因此，诞生于21世纪(2004)的RISC-V，必然也是精简指令集。并且，由于有了前面指令的经验，RISC-V吸收了以往指令集的优点，并回避了以往指令集的一些缺陷。优点如，RISC-V有32个通用寄存器，有专门的0寄存器。使用专门的移位指令来处理移位操作。使用load/store指令访问内存。跳转指令不使用条件码。缺陷则如，RISC-V废弃了MIPS中的分支延迟槽(有利于架构和实现的分离)，立即数只进行有符号扩展，算数运算不抛出异常(通过软件判断)，更规整的指令格式（源及目的字段位置固定），整数乘除法可选(简化了基础实现)。

### 1.3 完全开放

RISC-V采用BSD开源协议，任何人都可以自由免费地使用RISC-V。其他指令集如x86是完全闭源的，只有intel和AMD等少数公司能够基于x86架构生产产品。而ARM和MIPS都采用授权的方式，通过卖授权的方式盈利，其中ARM的授权费则十分昂贵。RISC-V如今由RISC-V基金会维护，任何公司只要愿意每年支付一笔会员费即可成为会员，会员可以参与到RISC-V之后标准的制定中。RISC-V也因此真正成为一个开放的指令集，不会因为任何一个公司的起伏而受到影响。

## 2. RISC-V的生态

以下内容引用自[开放指令集与开源芯片进展报告-v1p1](http://crva.io/documents/OpenISA-OpenSourceChip-Report.pdf)

> 2011 年，加州大学伯克利分校发布了开放指令集 RISC-V，并很快建立起一个开源软硬件生态系统。截止 2019 年 1 月，已有包括 Google、NVidia 等在内的 200 多个公司和高校在资助和参与 RISC-V 项目。其中，部分企业已经开始将 RISC-V 集成到产品中。例如全球第一大硬盘厂商西部数据（Western Digital）最近宣布将把每年各类存储产品中嵌入的 10 亿个处理器核换成 RISC-V；Google 利用 RISC-V 来实现主板控制模块；NVidia 也将在 GPU 上引入 RISC-V 等。此外，国内阿里巴巴、华为、联想等公司都在逐步研究各自的RISC-V 实现；上海市将 RISC-V 列为重点扶持项目；印度政府也正在大力资助基于 RISC-V 的处理器项目，使 RISC-V 成为了印度的事实国家指令集。

由此可见，RISC-V国内外的兴起使得目前RISC-V的生态已经比较完善了。

## 3. 之于构建开放硬件生态的意义

2019 年度国际计算机体系结构旗舰会议 ISCA 于 6 月在美国亚利桑那州凤凰城召开，会议的主题即是“ 面向下一代计算的敏捷开放硬件（Agile and Open Hardware for Next-Generation Computing ）”。由此可见开放硬件，敏捷开发已成为未来的趋势。而RISC-V则刚好成为这趋势中不可或缺的一环。

目前开源硬件开发中面临着4个关键问题(Yungang Bao, Chinese Academy of Sciences, The Four Steps to An Open-Source Chip Design Ecosystem)：

1. 开放的指令集ISA/开源IPs/开源Socs
2. 硬件描述语言Lanuages/开源的EDA工具
3. 验证和仿真Vertification/Simulation
4. OS/Complier

RISC-V便处于第一环中，

虽然目前每个环节都还未完全解决，但我们可以相信，在未来，开发硬件可以像开发一个软件一样。充分利用已开源的资源，用户只需要定制10%以内的代码，便可以以月为单位开发出客制化的硬件系统。



## 4. 参考资料

[大道至简——RISC-V架构之魂（上）](https://mp.weixin.qq.com/s/deNZzdSfxbdUoO58ok53tw)

[大道至简——RISC-V架构之魂（中）](https://mp.weixin.qq.com/s/rB9ln7-cDb0VjtikD6KWzw)

[大道至简——RISC-V架构之魂（下）](https://mp.weixin.qq.com/s/sIkKnJt6rQLxM5GM60OnLA)

[开放指令集与开源芯片进展报告-v1p1（2019-02-22更新）](http://crva.io/documents/OpenISA-OpenSourceChip-Report.pdf)

[远景研讨会纪要–面向下一代计算的开源芯片与敏捷开发](http://crva.io/documents/SIGARCH-Visioning-Workshop-Summary-Agile-and-Open-Hardware-for-Next-Generation-Computing.pdf)