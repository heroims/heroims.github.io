---
title: Mesh组网与智能路由器系统
date: 2021-12-06 01:09:05
tags:
    - Mesh组网
    - 异地组网
    - OpenWRT
    - Libremesh
    - batman
    - bmx
    - OpenVPN
    - frp
typora-root-url: ../
---



## Mesh路由系统

这里首先记录下当时调研的一些路由器系统，这方面的资料再网上比较少。

开源的智能路由系统其实挺多不少这个后面再说，比较偏向专业Mesh组网的智能路由系统比较少下面就列一下我了解到的几个还不错的。

![image.png](/assets/blogImage/83356518.png)

1.libremesh

http://libremesh.org/
支持各种漫游协议，各种连接方法，固件支持常见路由，但7620系统无线有bug，mini系统有简易配置页面，全系统可设置更改全面协议，理论上只要设备足，覆盖一个城都没有问题，而且最牛逼的是持续更新开发者都很活跃，参与人较多，文档较全，而且有每月互动email可以演习。我们当时户外版就是采用的这个系统，这倒是跟区块链很配绝对去中心，而且是完全自治独立于现有互联网的平行网络。

2.qMp,quik mesh project 

http://qmp.cat/Home
基本属于傻瓜式设置，只需要配置一下无线频道及无线属性就能连接，信息页面较少，无法全面了解连接情况，固件支持常见路由，test版本基本向libremesh靠拢，但似乎和老版本不兼容。

3.openwisp
http://openwisp.org/
这个看名字就能看出是干什么的，基于olsr协议，固件支持ramips、ar71xx以及x86，固件支持8M以上，太大了。

4.gluon

 https://github.com/freifunk-gluon/gluon

这个是我在github上推荐看见的也是个主攻mesh的智能路由系统，而且里面还提供了MeshVPN,而且也是一直有更新，看样子很厉害由于是后来发现的所以没深入了解，但看起来这算是一个不错的新选择，再有空搞Mesh相关的肯定会首选这个。

<!-- more -->

## [OpenWRT](https://github.com/openwrt/openwrt)

OpenWRT不同于其他许多用于路由器的发行版，它是一个从零开始编写的、功能齐全的、容易修改的路由器操作系统。实际上，这意味着您能够使用您想要的功能而不加进其他的累赘，而支持这些功能工作的linux kernel又远比绝大多数发行版来得新。

OpenWRT支持各种处理器架构，无论是对ARM，X86，PowerPC或者MIPS都有很好的支持。 其多达3000多种软件包，囊括从工具链(toolchain)，到内核(linux kernel)，到软件包(packages)，再到根文件系统(rootfs)整个体系，使得用户只需简单的一个make命令即可方便快速地定制一个具有特定功能的嵌入式系统来制作固件。
一般嵌入式 Linux 的开发过程, 无论是 ARM, PowerPC 或 MIPS 的处理器, 都必需经过以下的开发过程：

1. 创建 Linux 交叉编译环境；
2. 建立 Bootloader；
3. 移植 Linux 内核；
4. 建立 Rootfs (根文件系统)；
5. 安装驱动程序；
6. 安装软件；

熟悉这些嵌入式 Linux 的基本开发流程后，不再局限于 MIPS 处理器和无线路由器, 可以尝试在其它处理器, 或者非无线路由器的系统移植嵌入式 Linux, 定制合适自己的应用软件, 并建立一个完整的嵌入式产品。

现在还把各种分支版本吸收了进去可以说智能路由器系统的大成之作。

### Mesh组网

OpenWRT上提供了多种Mesh组网的方式，官网介绍了非常详细的三种，主要有[OLSR](https://openwrt.org/docs/guide-user/network/wifi/mesh/olsr)，[batman-adv](https://openwrt.org/docs/guide-user/network/wifi/mesh/batman)，[802.11s](https://openwrt.org/docs/guide-user/network/wifi/mesh/80211s)，除了这三种还有[BMX6](https://github.com/bmx-routing/bmx6)，[BMX7](https://github.com/bmx-routing/bmx7)也是比较有名好用的。到这里算是比较常见的组网方式，并且适配也比较广泛，还是开头提到的系统的前身，但不能在互联网中远程与其他设备Mesh组网。

### 异地组网

首先有要看看有没有真实IP，没有的话要先解决透传问题。[frp](https://github.com/fatedier/frp)是个不错的选择，配合[OpenVPN](https://github.com/OpenVPN/openvpn)可以比较简单的实现异地组网，但这种组网很明显有着一个弊端就是中心化，首先是frp依赖着1台拥有公网真实IP的机器要部署frp server在上面，使用者则要部署frp client，其次OpenVPN也是要分OpenVPN server和client。简单的说就是这个方案缺少Mesh的特性。

后来也就是最近本来找了人投了钱我们想结合区块链做一个这种带有异地Mesh组网功能的系统，怎奈币圈大涨投资人觉得项目再好也不让买币好久就行了资金，在这就简单说一下这项目对标的就是filecoin，frp server通过区块链记录配置信息，用二层网络从链上同步可用的frp配置到client上，至于OpenVPN我们则设计了MeshVPN,有点[peervpn](https://github.com/peervpn/peervpn)的意思，而VPN的配置信息也是二层网络从链上同步，配置上就可以加入对应的网络中，这样就实现了异地Mesh组网，而提供服务的人也都能自然而然的收到奖励，相当于挖矿但这种矿机要比传统没有任何实际贡献的矿机有意义的多，它促使Mesh网络的部署。





