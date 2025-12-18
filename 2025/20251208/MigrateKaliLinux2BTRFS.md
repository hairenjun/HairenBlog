---
title: 不使用LVM而是subvolume安装全新的LUKS+btrfs的KaliLinux
date: 2025-12-08 19:00:00

cover: https://image.hairenjun.link/2025/Cover_%E6%A9%99%E8%89%B2%E5%B0%91%E5%A5%B3%E5%BF%83.webp

tags:

  - Btrfs

  - Linux

  - Kali

  - Luks

  - Manual Install Linux

  - Data Transfer

  - Debootstrap
categories:

  - 踩坑实况

keywords: Kali,Linux,OS Install,Btrfs,Data Migrate
description: Install Kali linux with luks encryption and btrfs then migrate data from old system, without LVM but btrfs subvolume
---
## 不要在看完全文之前上手操作，不然会出问题

## 前情提要
由于历史遗留问题，系统装在一块原厂三星的SSD上，只有500G空间。不过用了四年了，装了VS Code还有各种套件都还120GB空间，搞搞低强度Coding完全够了。但是啊但是，很快强度就上来了，第一个大头就是自己编译手机的Kernel和LineageOS，一下就给我吃完了，还不得不分60G作SWAP不然就要当场爆内存。然后Proton兼容层让我直接把Gaming平台也迁移到Linux上了，这下Windows彻底成Adobe全家桶和国产毒瘤的垃圾场了（谁会放心用一个上个厕所就自己重启更新的系统呢？）。于是乎被迫在存储暴涨的时间段花重金买入新SSD，计划来个赛博大搬家。

## 脱离舒适圈，挑战新架构
既然花了这么大功夫，那就必须一步到位了，搞点狠活😋

首先就是文件系统这一块，Ext4确实很稳，但是我稳了四年了，说不准啥时候误操作rf -rm了，而且在单位确实出现了多起Ext4导致文件系统爆炸机子起不来的情况（突然断电这样的），也误删除过一些文件，毕竟rm没有回收站。再就是安全方面的问题，Linux没有系统还原点，如果被干了直接GG，内核被换了直接双手离开键盘没法玩。

这下Btrfs的优势就体现出来了，有快照，自纠错，有压缩，性能也不太差。除了不能方便的dd出一个块然后mkswap之外还没看到特别要命的问题。你可能会问为啥不用ZFS，ZFS确实强，但是我就这一块盘，ZFS太沉重了属实没必要。

然后你又要问，Kali的Live USB安装器可以直接自动Btrfs，手动装是何意味？这就不得不说自动装的问题了，这是早先时候和Gemini探讨可行性的时候发现的问题：如果要使用Luks，则自动安装会在Luks上用LVM而非直接使用btrfs的subvolume，变成了风味儿btrfs不说，还有额外的性能损失；而且还有必要规划一个更大的boot分区，现在装的东西多了，四年前的700MB可以放四个版本的kernel，现在不把前一个kernel删掉apt upgrade都跑不下来；自动LUKS会全盘写随机数，我这QLC SSD真顶不住怕英年早逝......

最后，该搞点新东西了，这是个拥抱新技术走上未来路的绝佳机会，设备变砖那一晚学的东西远超大学四年听的废话（）

## 开始操作🤡

首先是分区，物理上一块盘分成4个区，先后分别为：ESP，BOOT，SWAP，LUKS

结果第一步就出现问题，Gemini直接搞忘记了/boot放进LUKS会直接引导不起来，难绷(真的有人会把sudo权限给AI吗？)，还是自己参考现在用着的老SSD手动来吧：
``` bash
能正常启动的老SDD结构：
nvme0n1
│    zfs_me 5000  rpool     13079977825321431038                                  
├─nvme0n1p1
│    vfat   FAT32           2136-E3A9                               387.8M    24% /boot/efi
├─nvme0n1p2
│    ext2   1.0             012c4051-433c-4973-ae25-dfdb18d0ef65    268.6M    36% /boot
└─nvme0n1p3
  │  crypto 2               ae4f35d5-1afc-4a76-9fac-450a85aa2079                  
  └─nvme0n1p3_crypt
    │  LVM2_m LVM2            3T56uZ-sI5j-Skbm-Y6uH-HkYd-ICdF-gYFCnl                
    ├─hairenjun--vg-root
    │  ext4   1.0             d118e993-ed0e-42d8-8324-6cf9f6bb689e       67G    81% /
    └─hairenjun--vg-swap_1
       swap   1               2f507dd9-f299-4660-b1dc-1b17b9eded43                  [SWAP]
这ZFS哪来的我也不知道说实话，难道这么长时间没爆炸就是他在给我保驾护航？
``` 

分区没什么好说的，基本就是拿Disks分区，比较坑的点是不能一块儿办了：ESP得先分成没格式的EFI分区再手动mkfs.vfat -F 32，分完了记得挂载一下
```bash
新的SSD结构
NAME                                          FSTYPE      FSVER    LABEL     UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                                                                
├─sda1                                        vfat        FAT32              7A57-DF8C                                 511M     0% /mnt/kali/boot/efi
├─sda2                                        ext2        1.0                4c78fc7d-5505-427a-918a-e11ed9e7cce9        2G     0% /mnt/kali/boot
├─sda3                                        swap        1        SWAP      2d43200d-c869-488d-aa9e-2a0be3850891                  
└─sda4                                        crypto_LUKS 2                  6f36da02-44bc-441e-ae48-f225a66b3575                  
  └─luks-6f36da02-44bc-441e-ae48-f225a66b3575 btrfs                Btrfs     f4286e0b-1ebe-492c-9eab-00b184d461f0      3.7T     0% /mnt/kali/home
                                                                                                                                   /mnt/kali

临时挂载到mnt目录下，待会儿直接一个debootstrap+chroot
sudo mount -o defaults,noatime,compress=zstd,subvol=@ /dev/mapper/kali_crypt /mnt/kali
还有home目录和/boot、/boot/efi别忘了
```

然后就可以用debootstrap安装新系统的文件了：
``` bash
sudo debootstrap kali-rolling /mnt/kali http://http.kali.org/kali
```

然后为新系统共享现在运行的kali的环境然后一个chroot
``` bash
sudo mount --bind /dev /mnt/kali/dev
sudo mount --bind /proc /mnt/kali/proc
sudo mount --bind /sys /mnt/kali/sys
sudo chroot /mnt/kali /bin/bash
```
这里直接chroot的话会尝试用zsh，然而新系统是没有的所以会报错，需要显式指定bash

chroot 之后为apt添加仓库，然后update接upgrade，记得装kernel和btrfs，桌面环境我推荐Gnome😋

接着就是紧张刺激的安装启动引导器的环节，由于我们是在chroot环境下，所以不会影响到本机的grub；而且是手动安装可以指定--removable，不会在主板上拉托大顺便把原系统EFI搞不见（别问我是怎么知道的）

首先要修改fstab和crypttab，使其能够正确挂载文件系统
``` bash
nano /etc/fstab #喜欢用vim的dalao们随意

# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# 1. ROOT (The Encrypted Btrfs) - Mapped from sda4
/dev/mapper/cryptroot /               btrfs   defaults,subvol=@ 0       0
/dev/mapper/cryptroot /home           btrfs   defaults,subvol=@home 0       0
/dev/mapper/cryptroot /.snapshots     btrfs   defaults,subvol=@snapshots 0       0
# 2. BOOT (The Ext2 partition on sda2)
UUID=4c78fc7d-5505-427a-918a-e11ed9e7cce9  /boot  ext2  defaults  0  2
# 3. EFI (The FAT32 partition on sda1)
UUID=7A57-DFz8C  /boot/efi       vfat    umask=0077      0       1
# 4. SWAP (The swap partition on sda3)
UUID=2d43200d-c869-488d-aa9e-2a0be3850891  none   swap  sw        0  0


nano /etc/crypttab

cryptroot UUID=6f36da02-44bc-441e-ae48-f225a66b3575 none luks

#自己分区的UUID记得换了
```

然后就是initramfs和grub
``` bash
update-initramfs -u -k all
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Kali --removable --recheck
update-grub
```

最后添加一个常规用户,我还换成了zsh，各位随意
``` bash
useradd -m -G sudo -s /usr/bin/zsh USERNAME
chown -R USERNAME:USERNAME /home/USERNAME/

chsh -s /usr/bin/zsh root
```

大功告成，可以重启试试看效果了～！


## 喜闻乐见的Debug阶段🤡🤡
好消息是UEFI确实识别到了新的启动设备，能直接从USB起起来，坏消息是没能进系统...最后在initramfs弹了个shell出来

好在是有报错信息，提示找不到btrfs的UUID；但是诡异的是报错的这个ID并不在硬盘分区上也不再我给的fstab里，关键时刻Gemini的超长上下文找到了问题的关键：这个UUID是解锁后的btrfs，在解锁之前kernel当然读不到，所以还得对grub做出修改，让他能等decrypt完成后再去读东西。

但是问题就卡在了这一步让initramfs解锁LUKS这个环节，不知道是不是缺了什么关键步骤，反正就是不解锁......

在手动搜集资料后，发现initramfs存在一个"HOOK"机制，别的不说了可以去这两个目录下，看一眼就明白怎么个事儿了：
``` bash
/etc/initramfs-tools/ #用户定义hook
/usr/share/initramfs-tools/ #sys自己的hook
```

信不过AI了，我们直接基本法，看看现在这个能正常启动的initramfs是什么情况:
``` bash
└─$ tree .
.
├── conf.d
├── conf-hooks.d
│   ├── busybox
│   └── cryptsetup
├── hook-functions
├── hooks
│   ├── amd64_microcode
│   ├── btrfs
│   ├── cryptgnupg
│   ├── cryptgnupg-sc
│   ├── cryptkeyctl
│   ├── cryptopensc
│   ├── cryptpassdev
│   ├── cryptroot
│   ├── cryptroot-unlock
│   ├── cryptsetup-nuke-password
│   ├── dmsetup
│   ├── fsck
│   ├── fuse
│   ├── intel_microcode
│   ├── iscsi
│   ├── keymap
│   ├── klibc-utils
│   ├── kmod
│   ├── lvm2
│   ├── mdadm
│   ├── ntfs_3g
│   ├── plymouth
│   ├── reiserfsprogs
│   ├── resume
│   ├── thermal
│   ├── udev
│   ├── xfs
│   ├── zz-busybox
│   └── zz_nvidia-blacklists-nouveau
├── init
├── modules
├── modules.d
└── scripts
    ├── functions
    ├── init-bottom
    │   ├── plymouth
    │   └── udev
    ├── init-premount
    │   └── plymouth
    ├── init-top
    │   ├── 00_mount_efivarfs
    │   ├── all_generic_ide
    │   ├── blacklist
    │   ├── keymap
    │   ├── simple-framebuffer
    │   └── udev
    ├── local
    ├── local-block
    │   ├── cryptroot
    │   └── mdadm
    ├── local-bottom
    │   ├── cryptgnupg-sc
    │   ├── cryptopensc
    │   ├── cryptroot
    │   ├── iscsi
    │   ├── mdadm
    │   └── ntfs_3g
    ├── local-premount
    │   ├── btrfs
    │   ├── ntfs_3g
    │   └── resume
    ├── local-top
    │   ├── cryptopensc
    │   ├── cryptroot
    │   └── iscsi
    ├── nfs
    └── panic
        └── plymouth

14 directories, 61 files
```
再看看没起起来的新盘:
``` bash
└─# tree .
.
|-- functions
|-- init-bottom
|   |-- plymouth
|   `-- udev
|-- init-premount
|   `-- plymouth
|-- init-top
|   |-- all_generic_ide
|   |-- blacklist
|   |-- keymap
|   |-- simple-framebuffer
|   `-- udev
|-- local
|-- local-block
|   `-- cryptroot
|-- local-bottom
|   |-- cryptgnupg-sc
|   |-- cryptopensc
|   |-- cryptroot
|   `-- ntfs_3g
|-- local-premount
|   |-- btrfs
|   |-- ntfs_3g
|   `-- resume
|-- local-top
|   |-- cryptopensc
|   `-- cryptroot
|-- nfs
`-- panic
    `-- plymouth

9 directories, 22 files
```

然后问问Gemini是怎么回事，果然，差了一个非常关键的组件：
``` bash
apt install cryptsetup-initramfs
```
## 走捷徑了bro🤡🤡🤡

在反复重装之中，我突然发现安装器（Installer）在磁盘分区这一步的时候是可以手动分区的，不过就是有一些操作不然安装器不知道往哪装

首先是分区，这个没什么好说的，必要的分区只有三个其实：ESP，BOOT，ROOT

在分区分完大小后，使用"use as"这个选项卡选好文件系统，ESP就是EFI，兼容性考虑BOOT使用ext2，这里要选择mount point为/boot不然后面grub不知道往哪装

然后就是这沟槽的ROOT，要是没有安全需求的话单纯的btrfs或者ext4就好了，如果要上LUKS2的话，还是有点坑的，必须按操作来:
  + 首先是将分区大小分配好，例如我给/分了1TB空间，给/home分了2TB，我还是推荐分开的，这样的话system本身炸了也可以单纯重装而不严重影响工作的数据
  + 差点忘了SWAP还得分64G，我还是推荐多分点，毕竟btrfs不能后期dd出一个块当SWAP用
  + 然后，set up crypted volumes, 选定那俩分区，下一步的时候记得取消Erase Data，咱QLC经不起全盘写一遍的折腾，反正新盘没东西，挂在就用mapper就行
  + 配置LUKS的Passphrase
  + 然后就会回到分区界面，会发现多了仨mapper出来的块存储，再去选择文件系统（这次我肯定选btrfs了）
  + 选择挂载点，/ 和/home，其中/挂载点必须要有不然安装器不知道往哪塞文件，SWAP不用选挂载点

至此，分区工作完成，此时应该至少有三个分区和对应的挂载点（我有五个分区和三个挂载点）：ESP-->EFI, Ext2/Ext3/Ext4-->/boot, Ext4/Btrfs/... -->/

然后就可以点Finish Partition进入安装工作了，不然安装器会报错，要么装不上grub要不干脆找不到root不能进安装进程

/boot暴露在LUKS之外确实不对劲，但是虽然安装器会说systemd能够保证boot放LUKS里面也没问题，但是就没成功过，全是装不上或者起不来，不知道是bug还是操作有问题

选一下软件包和桌面环境等装完，我推荐Gnome

## 坑，又是大坑🤡🤡🤡🤡
这里不得不说一个大坑了，就是这个Gnome

务必下载2025.4版本的Installer，因为这个版本的会默认装Gnome49,和6.16内核；如果是老版本的话，一个 apt upgrade就会升级到6.17内核和Gnome49

这有什么问题呢？问题就是Gnome49彻底没有了X11而只支持Wayland/Xwayland， 容易出现老配置还是X11导致Gnome跑不起来；第二个就是FK Nvidia环节，加上6.17和Wayland Gnome会神奇的跑不起来桌面（我🥬）

试过在Rescure下重装Gnome都不行，于是发现事情不简单，直到我看到了这个，手动修有点困难了，反正是白板直接重装吧:
``` bash
DMESG：
[   54.488912] r8169 0000:05:00.0 eth0: Link is Down
[   59.531035] traps: gnome-session-i[1807] trap int3 ip:7fd6841c974b sp:7fff9fb0b820 error:0 in libglib-2.0.so.0.8600.2[6874b,7fd684180000+a5000]
[   65.971445] traps: gnome-session-i[1958] trap int3 ip:7f4c8e9f474b sp:7ffd92991160 error:0 in libglib-2.0.so.0.8600.2[6874b,7f4c8e9ab000+a5000]
[   71.151349] traps: gnome-session-i[2083] trap int3 ip:7f3c2a09874b sp:7ffcea7b0f40 error:0 in libglib-2.0.so.0.8600.2[6874b,7f3c2a04f000+a5000]
[   76.302550] traps: gnome-session-i[2228] trap int3 ip:7f32a616774b sp:7fff90c688d0 error:0 in libglib-2.0.so.0.8600.2[6874b,7f32a611e000+a5000]
[   81.518601] traps: gnome-session-i[2375] trap int3 ip:7f2b8df7774b sp:7ffe8c71e590 error:0 in libglib-2.0.so.0.8600.2[6874b,7f2b8df2e000+a5000]
[   86.702936] traps: gnome-session-i[2524] trap int3 ip:7f80a768474b sp:7ffefe07b4a0 error:0 in libglib-2.0.so.0.8600.2[6874b,7f80a763b000+a5000]

Journalctl:
 14 19:57:37 Sherrys gnome-session[1808]: Failed to start unit gnome-session-x11@gnome-login.target: GDBus.Error:org.freedesktop.systemd1.NoSuchUnit: Unit gnome-session-x11@gnome-login.target not found.
Dec 14 19:57:38 Sherrys systemd-coredump[1817]: Process 1808 (gnome-session-i) of user 60578 dumped core.
                                                Module libblkid.so.1 from deb util-linux-2.41.2-4.amd64
                                                Module libatomic.so.1 from deb gcc-15-15.2.0-9.amd64
                                                Module libmount.so.1 from deb util-linux-2.41.2-4.amd64
                                                Stack trace of thread 1808:
                                                #0  0x00007fe3ebef674b g_log_structured_array (libglib-2.0.so.0 + 0x6874b)
                                                #1  0x00007fe3ebef6bc4 g_log_default_handler (libglib-2.0.so.0 + 0x68bc4)
                                                #2  0x00007fe3ebef6e29 g_logv (libglib-2.0.so.0 + 0x68e29)
                                                #3  0x00007fe3ebef7193 g_log (libglib-2.0.so.0 + 0x69193)
                                                #4  0x000055ee866427a8 n/a (/usr/libexec/gnome-session-init-worker + 0x27a8)
                                                #5  0x00007fe3ebcc1ca8 n/a (libc.so.6 + 0x29ca8)
                                                #6  0x00007fe3ebcc1d65 __libc_start_main (libc.so.6 + 0x29d65)
                                                #7  0x000055ee86642a51 n/a (/usr/libexec/gnome-session-init-worker + 0x2a51)
                                                
                                                Stack trace of thread 1815:
                                                #0  0x00007fe3ebd329ee n/a (libc.so.6 + 0x9a9ee)
                                                #1  0x00007fe3ebd27668 n/a (libc.so.6 + 0x8f668)
                                                #2  0x00007fe3ebd276ad n/a (libc.so.6 + 0x8f6ad)
                                                #3  0x00007fe3ebd9be6e ppoll (libc.so.6 + 0x103e6e)
                                                #4  0x00007fe3ebeedaf4 n/a (libglib-2.0.so.0 + 0x5faf4)
                                                #5  0x00007fe3ebeee4cf g_main_loop_run (libglib-2.0.so.0 + 0x604cf)
                                                #6  0x00007fe3ec17c72a n/a (libgio-2.0.so.0 + 0x12e72a)
                                                #7  0x00007fe3ebf200e6 n/a (libglib-2.0.so.0 + 0x920e6)
                                                #8  0x00007fe3ebd2ab7b n/a (libc.so.6 + 0x92b7b)
                                                #9  0x00007fe3ebda87b8 n/a (libc.so.6 + 0x1107b8)
                                                
                                                Stack trace of thread 1814:
                                                #0  0x00007fe3ebd329ee n/a (libc.so.6 + 0x9a9ee)
                                                #1  0x00007fe3ebd27668 n/a (libc.so.6 + 0x8f668)
                                                #2  0x00007fe3ebd276ad n/a (libc.so.6 + 0x8f6ad)
                                                #3  0x00007fe3ebd9be6e ppoll (libc.so.6 + 0x103e6e)
                                                #4  0x00007fe3ebeedaf4 n/a (libglib-2.0.so.0 + 0x5faf4)
                                                #5  0x00007fe3ebeee1d0 g_main_context_iteration (libglib-2.0.so.0 + 0x601d0)
                                                #6  0x00007fe3ebeee221 n/a (libglib-2.0.so.0 + 0x60221)
                                                #7  0x00007fe3ebf200e6 n/a (libglib-2.0.so.0 + 0x920e6)
                                                #8  0x00007fe3ebd2ab7b n/a (libc.so.6 + 0x92b7b)
                                                #9  0x00007fe3ebda87b8 n/a (libc.so.6 + 0x1107b8)
                                                
                                                Stack trace of thread 1813:
                                                #0  0x00007fe3ebda6779 syscall (libc.so.6 + 0x10e779)
                                                #1  0x00007fe3ebf1f962 g_cond_wait (libglib-2.0.so.0 + 0x91962)
                                                #2  0x00007fe3ebeb3b04 n/a (libglib-2.0.so.0 + 0x25b04)
                                                #3  0x00007fe3ebf203a4 n/a (libglib-2.0.so.0 + 0x923a4)
                                                #4  0x00007fe3ebf200e6 n/a (libglib-2.0.so.0 + 0x920e6)
                                                #5  0x00007fe3ebd2ab7b n/a (libc.so.6 + 0x92b7b)
                                                #6  0x00007fe3ebda87b8 n/a (libc.so.6 + 0x1107b8)
                                                ELF object binary architecture: AMD x86-64
```

第二个坑来自Btrfs，首先是记得装btrfs-progs不然update-grub还是update-initramfs会有点问题，导致无法完成升级，表现就是apt upgrade会Fail；第二个就是Installer会在/下创建一个.snapshot的subvolume,如果要使用btrfs-assistant这样的工具创建根目录快照防炸机的话会提示subvolume存在，我听信了Gemini的话手动卸载并删除了.snapshots然后btrfs确实OK的创建好了，但是下次开机直接进维护模式了...multiuser.target都进不去...这里又要提醒一下，进系统第一件事就是创建root用户密码，不然维护模式都进不去，没权限拿TTY

没办法只能再拿安装器，进rescure mode，挂载root，改root密码；直接journalctl，发现是无法挂载.snapshots致的,那好说了，直接去/etc/fstab把这个挂载去掉就行了，毕竟snapshot是btrfs-assistant在管，改完了update-initramfs然后update-grub2, OK

有一说一，用了btrfs后可以每一个小时保存一次快照，每天保存一个快照，再也不怕rm -rf了

## 参杂私货😋
{% raw %}
<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=637549597&bvid=BV19Y4y1i7or&cid=556747043&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>
{% endraw %}
听着就像是回到了胡思乱想的初中时代,封面大图
![大河Prpr](https://image.hairenjun.link/2025/Cover_%E6%A9%99%E8%89%B2%E5%B0%91%E5%A5%B3%E5%BF%83.webp)