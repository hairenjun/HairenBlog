---
title: 使用POWERSHELL对抗数据恢复
date: 2024-03-12 17:27:43
cover: https://s2.loli.net/2024/03/13/5OCLuMwKsbgTor4.png
tags:
  - Powershell
  - SSD
  - TRIM
  - 数字取证对抗
categories:
  - 踩坑实况
keywords: Powershell,TRIM,Optimize-Volume
description: Use TRIM via MS Powershell to destroy file on SSD
---

# 数据恢复？那必不能！

## Chapter 1 前情提要👀
　　数据安全一直是困扰每一个男人的心头之痛，谁都不想自己的电子设备因为丢失或是其他原因而被扒出来里面的陈年黑历史还有深夜的万华镜。<br>
　　事实上，相对于物理世界的实物证据，存在硬盘及网络上的数字痕迹在某种意义上更难以消除，如何真真正正的删除电脑上的一个文件，其实是每个男人都应该会的操作。<br>
　　从一块嫌疑人的硬盘上完整恢复需要的数字证据的技术，是数字取证这个科技树的一个小分支；那为了防止自己的个人信息被技术手段取走看光光，我们姑且称其为数字取证对抗的一小支。我的认识里，对于硬盘信息，有两条科技树可以爬，一个是加密，一个是销毁——我们今天来谈谈销毁。<br>
　　为什么堂堂君子需要这种操作？别问，有的倒霉蛋就是™需要,天天搁这赛博猎女巫，我还以为是带着笔电回中世纪的欧洲了呢😅

***

## Chapter 2 从回收站里删除也不行吗？🤔
　　**不行！**（即答）<br>
　　当你在Windows回收站“彻底删除”一个文件的时候，Windows只是在[MFT](https://baike.baidu.com/item/%E4%B8%BB%E6%8E%A7%E6%96%87%E4%BB%B6%E8%A1%A8/8971573)表中将该文件标记为“已删除”，而真实存放文件数据的磁盘扇区并不会第一时间被其他数据替换或填充，从而给绕过Windows系统直接从磁盘扇区恢复文件提供了可能性。就好比是你跟你妈说作业写完了但实际上第二天早上到教室了开始疯狂抄一样——你妈半夜偷偷看你作业本你就露馅了。同样的，有坏B来直接读你的磁盘扇区，你的“已删除”的文件也会回来，GG。<br>
　　如果你直接对磁盘分区进行格式化，里面的数据也还嫩找回来！<br>
　　只有当Window将新的文件的数据存入这些扇区的时候，被删除的文件才算真的被破坏了。
***

## Chapter 3 做到万无一失👌
　　既然知道了，文件数据存放在对应的扇区。如果你的硬盘真的脏的不行了，最好的方法就是将整个磁盘的所有扇区全部填入无效的数据（咱就说富哥直接把盘烧掉也不是不行）。<br>
　　但是大部分时候吧，是少量脏东西混在大量正经文件里面，全部白给显得不太合适。这时候，最好的选择就是找到文件数据存放的对应的扇区，将无用的数据（比如00，FF，随机数）替换掉其中的数据，于是这份文件就永久损毁了；再去MFT表中找到该文件的记录，删掉，OJBK，就好像这个文件不曾存在过一样。唯一可能的恢复手段，就是沉SSD主控还没来得及搞操作，就绕过主控直接去读闪存颗粒，寄希望于SSD的保留区块内还残存的文件的原始数据，名言：“碰运气，怎么可能的麻”。<br>
　　事实上市面上已经有相当专业的软件可以干这个事情了，比如[DiskGenius](https://www.diskgenius.com/)，这套方案已经完全能够应对绝大部分的数据恢复了，无论是机械硬盘还是SSD，读不出来就是读不出来，菜，就多练（）。

***

## Chapter 4 软件洁癖主义者🥰
　　yysy，仅仅为了一个小小需求就安装一个专业软件，实在是有点不好接受。那有没有不装软件就能实现类似功能的方法呢？<br>
　　**有！** 这就是利用[TRIM](https://zhuanlan.zhihu.com/p/612220472)，一个主流SSD都支持的特性。通过Powershell，我们能手动触发操作系统向SSD发送优化的TRIM指令，使其释放不用的扇区，这样，文件就不见力！<br>
### How to 
1. 删除你想销毁的文件，对，就是你经常用的Windows的那个删除
2. 清空回收站
3. 打开一个有管理员权限的Powershell窗口（[不会？](https://mwell.tech/archives/473#:~:text=%E4%BB%A5%E4%B8%8B%E6%98%AF%E5%9C%A8%20Windows%2011%20%E4%B8%8A%E4%BB%A5%E7%AE%A1%E7%90%86%E5%91%98%E8%BA%AB%E4%BB%BD%E6%89%93%E5%BC%80%20Windows%20PowerShell%20%E7%9A%84%204,%E5%88%87%E6%8D%A2%E5%88%B0%20PowerShell%20Admin%EF%BC%9A%E5%9C%A8%E6%99%AE%E9%80%9A%E7%9A%84%20PowerShell%20%E4%B8%AD%EF%BC%8C%E5%A4%8D%E5%88%B6%E5%B9%B6%E7%B2%98%E8%B4%B4%E4%BB%A5%E4%B8%8B%E4%BB%A3%E7%A0%81%EF%BC%8C%E7%84%B6%E5%90%8E%E6%8C%89Enter%EF%BC%9Astart-process%20powershell%20-verb%20runas)）
4. 使用下面的命令手动触发SSD TRIM
```
Optimize-Volume -DriveLetter <替换为已删除文件所在分区的盘符> -ReTrim -Verbose
```
5. 重启电脑（主打一个心理安慰）

　　可以看看微软的[官方文档](https://learn.microsoft.com/en-us/powershell/module/storage/optimize-volume?view=windowsserver2019-ps)。效果好滴很！对于绝大多数非破坏性数字取证方案完全够用。不过**数据无价，谨慎操作！！！**


***
# 封面图
[![虎-UHT](https://s2.loli.net/2024/03/13/5OCLuMwKsbgTor4.png "我的XP是：武装直升机！")](https://www.pixiv.net/artworks/104670190)
        PID：104670190





