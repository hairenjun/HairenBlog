---
title: 在Android设备上使用BBR拥塞控制算法

date: 2024-10-5 19:00:00

cover: https://s2.loli.net/2024/10/04/GXtswnKOzPvFYVe.png

tags:

  - Google BBR

  - Linux Kernel Compile

  - Congestion Control

  - Android

  - OnePlus 9

categories:

  - 整活

keywords: Android,BBR,Linux Kernel Compile, Oneplus 9
description: Use BBR Congestion Control Algorithm to improve network performance in SHIT Chinese Sub-6 5G NR environment on Android Phone and RNDIS share
---

# 在Android设备OP9上启用BBR来争夺有限的流量带宽

## Chapter 1 前情提要👀

由于条件的特殊性，确实会存在这么一种地方,几百号人在一个巨大院子里，没有有线互联网连接，上互联网全靠智能手机，而且通常来说只会有一个基站。由于Privacy考虑我就不贴CellularZ的截图了。考虑到5G NR的空口承载能力很高，但是基站到骨干网就不好说了。想在其中丝滑上网，必须有点狠活。

有一个很反常识的事情，就是这种环境中对上传带宽也是有要求的。不管是因为阻塞导致的重传，还是普通的fetch资源，你得先把请求传出去才能有回复对吧。所以光有一个下载带宽是不够的，很可能出现打不开网页，或者部分资源超时的问题，实际体验吊差。

## Chapter 2 Why BBR

BBR相比于默认的Cubic，其根据探测的最大的带宽而非丢包率而判断当前网络的状况，所以在网络带宽受限而人很多的情况下更能抢到带宽，而且原来玩VPS的时候看到过暴力多倍发包的方法。这样便能占据更多的上传带宽，人家还在等重传呢，我这边请求到位了，Server的数据下来自然的我的下载带宽也算是占据到了。

## Chapter 3 在Linux环境下使用BBR

没记错的话Linux Kernel中很早就有BBR的代码了，不晚于5.4。由于BBR作用在TCP Stack中，所以必须在内核中运行，这部分实话实说我不太了解，但大概是这个意思，不是什么双击运行就能解决的事情。想让内核支持BBR，最好的选择就是在compile的阶段就config上BBR，把它编译在kernel里面。

可以用这个命令检查一下内核是否支持BBR

```
sysctl net.ipv4.tcp_available_congestion_control
```

理论上来说我的Linux kernel是支持BBR的，但是我运行这个命令却只得到了reno和Cubic的结果，怪诶。我想了想，可能是编译了，但是为了内存什么的奇奇怪怪的原因没有载入到kernel中去。这就要说第二种方式了，使用modprobe或者insmod讲bbr以kernel mod的形式实时的装到kernel中运行，试了一下下面的命令：
```
sudo modprobe tcp_bbr
```

嗨嗨嗨，没有报错，然后再查看sysctl就已经支持bbr了

最后来一套丝滑小连招（记得sudo）
```
echo net.core.default_qdisc=fq >> /etc/sysctl.conf
echo net.ipv4.tcp_congestion_control=bbr >> /etc/sysctl.conf
sysctl -p
```
default_qdisc是排队队列，fq代表公平排队，但这似乎是BBR推荐的方式，关于这两个参数的科普可以看一个知乎老哥写的[文章](https://zhuanlan.zhihu.com/p/605735064)

这样其实还有一个问题，写在sysctl.conf里面的配置虽然不会丢失，但是上面那句modprobe是会的。方法有很多，有用sysctl的有用systemd的，挺多的反正

性能怎么样？测不出来，因为我就是那个用智能手机上网的，BBR只在电脑数据线手机之中起效果，手机到基站要看手机TCP Stack是什么情况。

## Chapter 4 看看手机上应该怎么操作

Oneplus9(China LE2110) aka lemonade 是一款搭载了高通傻逼888的中间忘了后面忘了的手机。好消息是它可以Unlock Bootloader，这就给操作带来了无限可能。我可以接受手机原厂Rom不给Root广告又多预装的司马东西和亲🐎的私生活一样乱，但是踏马的一个二个不给解锁Bootloader是干嘛？我踏马花两三千的价格租一个手机用来利用碎片时间看广告吗？

不扯远了，LE2110原生支持OxygenOS，这也是一加手机的标配，当然小粉红的原厂是沟槽的ColorOS，祖国的花朵一定要多看一点广告捏不然不利于生长。我因为工作的原因不得不选择与GMS完全切割（GMS包爽的），所以选择了[LineageOS](https://wiki.lineageos.org/devices/lemonade/) ，体验极佳，除了AOSP的必要应用外啥都没有，打打电话拍个照片和Nokia一样，有一种烂尾楼的美，开局一块儿地，装修靠自己。就是带着手机进澡堂子听音乐导致手机进水了现在用不了power键😭，好在有Tap2Wake/Lock不然手机就彻底废了。

那Unlock bootloader后就好说了，把LineageOS的Boot.img交给magisk打个patch，再用fastboot直接flash进boot分区，reboot！再刷进[Nethunter-Lite](https://www.kali.org/get-kali/#kali-mobile)侧载一个Kali，这会自动安装Busybox，这样那些被阉割的command也回来了，不过LineageOS的kernel是带了toybox的所以会有sysctl和modprobe，基本上就是一个常见发行版的体验。

事不宜迟，赶紧插线用adb shell看看默认的拥塞控制算法是什么


```
#:uname -a
Linux kali 5.4.280-qgki-ga0bd26c86333 #1 SMP PREEMPT Mon Sep 30 09:53:49 UTC 2024 aarch64 Toybox

#:sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = reno cubic

#:sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = cubic
```

现在就有两种方案，但是都要对Kernel下手，一种是想办法重新Compile整个Kernel，make的时候把BBR加上，第二种是使用modprobe和insmod在boot的时候就把BBR挂在kernel上。

第一种方案的好处是兼容性不用担心，上面的uname中qgki代表的就是Qualcomm Generic Kernel Interface，毕竟是给特制设备用的，Kernel多多少少做了一些修改。直接compile整个kernel就不用担心一些奇奇怪怪的兼容性问题，还不用想办法modprobe了；不好的地方自然是编译整个kernel开销太大，而且我从哪搞来kernel的source呢？第二种方案就刚好相反了，好操作，但是未知的风险很多。

考虑到对kernel的魔改应该是为了适配SoC，对TCP Stack这样的逻辑的软件的部分应该没什么特别大的修改，我觉得，用compile单个mod再挂载的方式应该没什么问题，毕竟是临时挂载，就算kennel panic了大不了reboot嘛。就决定是这样了！

计划就是找到对应的5.4.280这个Version的kernel source，然后直接compile出tcp_bbr.ko，adb push到手机上最后来modprobe+sysctl的丝滑小连招。

## Chapter 5 喜闻乐见的看代码环节

可以很容易的在[Linux Kernel Archive CDN](https://cdn.kernel.org/pub/linux/kernel/v5.x/)上找到5.4.280这个版本的Source Code

然后我们进net/ipv4去翻找一下，果然找到了tcp_bbr.c，同时还在这个目录下找到了诸如Westwood，dctcp等算法的Source Code，byd有代码不编译给我阉割了是吧（多半是出于节省内存的考虑）

BBR后面的v2v3版本特别考虑了“公平性问题”，可是我用BBR就是要利用探测最大BW占满带宽的这个特性的，要公平个der。所以要看看源码，将其修改为我要的样子（指明抢）。除此之外，将一个不明不白地程序挂载进kernel也是极为不好的习惯，多多少少看看源码。

**OK，咱们正式开始看Source**

首先来看看在Source中贴出来的模型，这概述了整个BBR算法的设计思路和工作原理

![BBR 模型](https://s2.loli.net/2024/10/05/zBr6lYbCigGctqo.jpg)

核心思路是探测最大带宽（BW，bandwidth）和往返时间（RTT，Round-Trip Time），并根据此来算出应该以多快的速度发包来充分利用网络资源。

总共有**四个状态**
- **STARTUP：** 启动状态，在这个过程中BBR逐渐增加发送包速率来试探带宽到底有多宽
- **DRAIN：** 开苟状态，发现带宽被挤占殆尽则降低发送速率，防止动漫量导致脱出（指丢包）
- **PROBE_BW：** 稳定状态，但是还是时不时浅浅的试探一下BW，看有没有可能再加点速度
- **PROBE_RTT：** 稳定状态，但是还是时不时降一下速测量RTT，这个RTT怎么用咱们后面看Source就知道了

差不多就是这样，细节咱后面看Source

注意Line 159-165
``` C
/* The pacing_gain values for the PROBE_BW gain cycle, to discover/share bw: */
static const int bbr_pacing_gain[] = {
	BBR_UNIT * 5 / 4,	/* probe for more available bw */
	BBR_UNIT * 3 / 4,	/* drain queue and/or yield bw to other flows */
	BBR_UNIT, BBR_UNIT, BBR_UNIT,	/* cruise at 1.0*bw to utilize pipe, */
	BBR_UNIT, BBR_UNIT, BBR_UNIT	/* without creating excess queue... */
};
```
这里面的第二个int，在DRAIN状态降速到75%来防止丢包并让出带宽给别人，防止丢包我可以理解，但是让给别人我不能接受。如果降速，那转发队列就会被其他人填充，BW就会再下降，发包速率就会更低，什么君子协定带宽条约，丧权辱机的事情，这肯定不能接受啊，我们先记着，先不着急改，看看后面会怎么用这个数组


