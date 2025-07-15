---
title: 看看fortiweb寻找持久化手段
date: 2025-7-14 19:00:00

cover: https://s2.loli.net/2024/10/05/D8cHzbgr2Ej4y5p.png

tags:

  - Fortiweb

  - CVE-2025-25257

  - guestmount

  - 固件分析
categories:

  - 整活

keywords: Fortiweb,CVE-2025-25257,guestmount,固件分析
description: Disassemble and analyze the FortiWeb virtual machine to explore its startup process in order to identify persistence hook points
---

## 叠甲/Delaimer
The disassembly and analysis of the FortiWeb virtual machine described herein is conducted solely for the purposes of legitimate security research and educational understanding. This activity is performed on a virtual machine instance for which I have explicit authorization and legal access rights. The intent is to explore the internal startup mechanisms to identify potential persistence hook points strictly within the context of enhancing defensive security knowledge, vulnerability research (with responsible disclosure principles). If this article fk u up, contact me to delete it.

本文所述针对 FortiWeb 虚拟机的拆解与分析行为，仅用于合法的安全研究及教育学习目的。该活动在已获得明确授权及合法访问权限的虚拟机实例上进行，旨在通过研究其内部启动机制，识别潜在的持久化植入点，严格限定于：提升防御性安全认知和遵循负责任披露原则的漏洞研究。侵删。

转载注明出处/Reprinting Requires Authorization with Clear Attribution

滥用文章中的技术所造成的后果自负，作者不承担责任/Any consequences arising from the misuse of techniques described in this article are solely the responsibility of the implementer. The author shall not be held liable for any damages or legal violations whatsoever resulting from such actions.

## 基本特性
Fortiweb在重启之后，绝大部分的运行时创建的文件都会消失，也没有corntab之类的东西，所以必须从启动流程入手做持久化。在我看来唯一的方法就是去启动流程之中做手脚，hook住拉起服务的脚本或者别的什么，所以说要对fortiweb的启动流程有详细的了解。

## 开拆
某海鲜市场花费5CNY拿到虚拟机，一个boot.vmdk和两个适配不同版本的ovf，很显然boot.vmdk是问题的关键点。

直接贴virt-filesystems的结果
``` bash
└─$ virt-filesystems --all --long -a ./boot.vmdk
Name       Type        VFS      Label  MBR  Size         Parent
/dev/sda1  filesystem  ext3     -      -    379658240    -
/dev/sda2  filesystem  unknown  -      -    409600000    -
/dev/sda3  filesystem  unknown  -      -    102400000    -
/dev/sda4  filesystem  unknown  -      -    32212254720  -
/dev/sda1  partition   -        -      83   409600000    /dev/sda
/dev/sda2  partition   -        -      83   409600000    /dev/sda
/dev/sda3  partition   -        -      83   102400000    /dev/sda
/dev/sda4  partition   -        -      83   32212254720  /dev/sda
/dev/sda   device      -        -      -    34359738368  -
```
后面unkown的暂时不管，多半是LVM逻辑卷，ext3排第一个一般按照惯例都是/boot分区，挂载上进去看看
``` bash
sudo guestmount -a ./boot.vmdk -m /dev/sda1 --ro /tmp/boot
```
su之后cp -r 把文件都弄出来，记得chmod 755

发现比较关键的几个文件，vmlinuz，这是内核；rootfs.gz，文件；krootfs.gz，也是文件？datafs.tar.gz，多半是数据之类的。注意到还有很多.chk文件，对应着这几个文件的文件名，那我直接默认有启动时文件校验防止被打patch。那继续往下看，拆文件。

这几个文件看似写的是gz，实际上不是gzip而是xz
``` bash
./krootfs.gz: XZ compressed data, checksum CRC32
./rootfs.gz: XZ compressed data, checksum CRC32
...

7z t ./rootfs.gz                            

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=en_US.UTF-8 Threads:32 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 234834640 bytes (224 MiB)

Testing archive: ./rootfs.gz
WARNING:
./rootfs.gz
Cannot open the file as [gzip] archive
The file is open as [xz] archive

--
Path = ./rootfs.gz
Open WARNING: Cannot open the file as [gzip] archive
Type = xz
Physical Size = 234834640
Method = LZMA2:23 CRC32
Streams = 1
Blocks = 33
Cluster Size = 25165824
Characteristics = BlockPackSize BlockUnpackSize

Everything is Ok

Archives with Warnings: 1
Size:       819200000
Compressed: 234834640

```
直接拿7z去解压缩就好了，解出来又是几个二进制文件，估计是block;那个datafs比较老实不是二进制，打开一看是etc

file验证了我的猜想
``` bash
./krootfs: Linux rev 1.0 ext2 filesystem data, UUID=224b4282-c280-402b-8fb0-58006b03c060 (large files)
./rootfs: Linux rev 1.0 ext2 filesystem data, UUID=c2d46191-007e-464d-a405-bc815f63a492 (large files)
```

同时注意到一个文件extlinux.conf
``` conf
PROMPT 1
TIMEOUT 10
DEFAULT FWEB_image

LABEL FWEB_image
  MENU LABEL FWEB_IMAGE
  KERNEL vmlinuz
  APPEND rw panic=5 clocksource=tsc root=/dev/ram0 ramdisk_size=800000 eagerfpu=on mitigations=off crashkernel=128M softlockup_panic=0 hung_task_panic=0 initrd=/rootfs.gz 

LABEL FWEB_rescue
  MENU LABEL FADC_rescue
  KERNEL vmlinuz.kdump
  INITRD krootfs.gz
  APPEND rw panic=5 clocksource=tsc root=/dev/ram0 ramdisk_size=800000 eagerfpu=on mitigations=off crashkernel=128M softlockup_panic=0 hung_task_panic=0 initrd=/krootfs.gz 
```
捕捉到关键词，initrd=/rootfs.gz，同时注意到整个fs跑在内存里面，怪不得重启就没了

直接mount来看一看
``` bash
mkdir /tmp/rootfs
sudo mount -t ext2 ./rootfs /tmp/rootfs

bin   dev  home  lib64     mnt      proc    share  tmp  var
data  etc  lib   migadmin  modules  script  sys    usr  VERSION
```
就是文件系统捏，直接cp出来

打个tree . 扫一眼没看到常见的启动流程文件，init.rc什么的,然后直接甩给Gemini2.5 pro，妈的Gemini2.5 Pro太爽了，要是deepseek这波肯定超content length了, API还能按次数而非token白嫖

gemini分析了一下，应该是/etc/init负责拉起整个业务，去看看
``` bash
./init: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 5.4.0, stripped
```
[Cutter](https://github.com/rizinorg/cutter),启动！

其实可以到string的x-ref看具体的启动流程，但是不搞这么麻烦，直接在.rodata段把string都扒出来，看一看有没有下手的空间

``` bash
/bin/smit startup
/bin/syslogd -n -b 99 -s 500
/bin/klogd -n
/bin/hamain
/bin/dhcp
/bin/udp_tunneld
/bin/getty ttyS0
/bin/sshd
/bin/httpsd
/bin/mariadb_exec
/bin/redis-server /data/etc/redis/redis_6381.conf
/bin/redis-server /data/etc/redis/redis_6380.conf
..... 理论上来说都给拉起来了
```
我们重点关注下面一些感觉好下手的目标，还有**smit**这哥们，感觉有说法的
``` bash
/bin/smit startup
/bin/smit save-redis-6379
/bin/smit save-redis
/bin/smit save-bot-redis
/bin/smit reboot
/bin/smit reset
/bin/smit update %s %s %d
/bin/smit cmdbsvr restart
/bin/smit restart-gui
/bin/smit erasedisk flash %d
/bin/smit erasedisk disk %d
/bin/smit restart-sshd
/bin/smit upgrade secondary-image up
/bin/smit flush-/bin/smit hot-re%s%s
....
/bin/vmtoolsd --common-path=/lib/open-vm-tools/plugins/common --plugin-path=/lib/open-vm-tools/plugins/vmsvc
/bin/python3 /script/web_cache_clean.py
/bin/filebeat -c /data/etc/filebeat/filebeat.yml
/lib64/ld-linux-x86-64.so.2
/data/etc/ld.so.preload
....
```
## 思路
接下来的思路，就着init拉起来的进程看有没有说法，比如说这个带参数的python，一看就有让人hook的欲望，但是别忘了rootfs是有校验的，而且整个fs跑在内存里面，不抗重启；这种脚本也不抗更新，连带着checksum一块hook？filebeat这个，虽然不知道在干吗，但是后面这个参数-c /data/etc/，data和etc是datafs的东西，没有校验，是个持久化的好方向，就是不知道这个程序和yml有没有利用空间....（看一眼就知道了，我就不看了）

不妨也来看看[官方思路](https://www.fortinet.com/blog/psirt-blogs/importance-of-patching-an-analysis-of-the-exploitation-of-n-day-vulnerabilities),笑嘻了哥们

后面的操作大家自己去探索吧，我先开吃了
