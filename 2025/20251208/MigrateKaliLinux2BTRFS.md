---
title: 不使用LVM而是subvolume安装全新的LUKS+btrfs的KaliLinux
date: 2025-12-08 19:00:00

cover: https://tc.z.wiki/autoupload/f/s53jY0hzeXWnA2DBfRnx7Z_Kv4b_7Q93KIuY3QIXybyyl5f0KlZfm6UsKj-HyTuv/20250715/Qd5M/840X1200/60168840_p0.jpg

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

## 前情提要
由于历史遗留问题，系统装在一块原厂三星的SSD上，只有500G空间。不过用了四年了，装了VS Code还有各种套件都还120GB空间，搞搞低强度Coding完全够了。但是啊但是，很快强度就上来了，第一个大头就是自己编译手机的Kernel和LineageOS，一下就给我吃完了，还不得不分60G作SWAP不然就要当场爆内存。然后Proton兼容层让我直接把Gaming平台也迁移到Linux上了，这下Windows彻底成Adobe全家桶和国产毒瘤的垃圾场了（谁会放心用一个上个厕所就自己重启更新的系统呢？）。于是乎被迫在存储暴涨的时间段花重金买入新SSD，计划来个赛博大搬家。

## 脱离舒适圈，挑战新架构
既然花了这么大功夫，那就必须一步到位了，搞点狠活😋

首先就是文件系统这一块，Ext4确实很稳，但是我稳了四年了，说不准啥时候误操作rf -rm了，而且在单位确实出现了多起Ext4导致文件系统爆炸机子起不来的情况（突然断电这样的），也误删除过一些文件，毕竟rm没有回收站。再就是安全方面的问题，Linux没有系统还原点，如果被干了直接GG，内核被换了直接双手离开键盘没法玩。

这下Btrfs的优势就体现出来了，有快照，自纠错，有压缩，性能也不太差。除了不能方便的dd出一个块然后mkswap之外还没看到特别要命的问题。你可能会问为啥不用ZFS，ZFS确实强，但是我就这一块盘，ZFS太沉重了属实没必要。

然后你又要问，Kali的Live USB安装器可以直接自动Btrfs，手动装是何意味？这就不得不说自动装的问题了，这是早先时候和Gemini探讨可行性的时候发现的问题：如果要使用Luks，则自动安装会在Luks上用LVM而非直接使用btrfs的subvolume，变成了风味儿btrfs不说，还有额外的性能损失；而且还有必要规划一个更大的boot分区，现在装的东西多了，四年前的700MB可以放四个版本的kernel，现在不把前一个kernel删掉apt upgrade都跑不下来；自动LUKS会全盘写随机数，我这QLC真顶不住怕英年早逝......

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


## 喜闻乐见的Debug阶段🤡🤡🤡
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


