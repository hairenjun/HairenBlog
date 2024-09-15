---
title: X64DBG+Cutter动静结合三板斧玩转微信

date: 2024-09-7 19:00:00

cover: https://s2.loli.net/2024/03/13/5OCLuMwKsbgTor4.png

tags:

  - WeChat

  - 微信

  - 逆向工程

  - Cutter

  - X64DBG

  - Reverse Engineering

categories:

  - 逆向实况

keywords: WeChat,Reverse Engineering,Cutter
description: Use Cutter/X64DBG reverse Wechat and make a dll patch to disable "recall" function
---
# X64DBG+Cutter+CE动静结合三板斧逆向微信推出防撤回补丁

## Chapter 1 前情提要👀

微信群里有爆典高手，但是每次看消息都很不及时，于是看着recalled message无能为力...那必不可能！于是萌发了想办法做一个防撤回补丁的想法，Just Do It！

本blog写的时候距离第一次成功操作已经过去了快三四个月了，微信版本还是老的V3.9.7.29，现在忘得基本上差不多了。刚好趁这个写Blog的机会在重温经典一下，练一练二进制分析的本领，所以说实际结果很可能与Github上的操作并不相符。

## Chapter 2 思路整理😋

主要思路还是用去抓一下到底是哪个Function在Recall这个message。一个message被Recall的流程，我猜是从接收到网络包开始，处理网络消息，写DB，最后在用户界面上把消息撤回去

我们希望的是，在处理网络消息与写DB这个流程之间找到个关键Function，禁用它，使其无法处理后面的流程，这样就实现了防撤回的功能

## Chapter 3 开始操作

我还记得突破口是revoke这个关键词（虽然官方英语的显示是recall），不妨先静态分析一下？
![疑似](https://s2.loli.net/2024/09/07/Uh3MzTAiyknjIuc.png)
找到了一些有意思的Strings，然后我们看看Xref到函数的位置，有几个Xref，一个个看看呗...然后就发现啊，他们后面都会call一个在偏移量在0x13674D0的一个function
![可疑的function](https://s2.loli.net/2024/09/08/VGrW7XP9epUFAmx.png)
这不追上去看看？

疑似是个用来写什么东西的函数，看看Xref？
![Xref@13674d0](https://s2.loli.net/2024/09/08/MVmLJaFRPZwGyvT.png)
不太对劲，实在是太多了，他可能确实是个关键函数，但是并不是我们要找的那个部分，很可能别的功能也要靠它工作。只能看看有没有别的Function可以操作。

整理一下思绪，在0x09bd7d0有一个ChatRevokeMgr::revokeMsg，在0x09bccc0有一个ChatRevokeMgr::doRevoke，且根据反编译的结果，revokeMsg在最后恰好会调用doRevoke。这里应该就是执行的部分了。相对于破坏性的去删改成一片nop，我更希望只修改一下流程，使其不能进入到这个处理revoke动作的分支。所以应该往上找，继续找Xref。也可以用X64DBG上个断点看看啥效果。

找到Xref后往上翻，翻到Function为止。好消息不远处就有一个Function，坏消息，这个Function里面压根就没有我们需要的0x09bd7d0的调用。捏麻麻的，突然就没活整了。那不如试试往下走走？毕竟，doRevoke的部分才是真的在revoke对吧

## Chapter 4 没活整了，上断点

X64DBG，启动！进行一个动态调试！先把这个基址搞清楚
![BaseAddr](https://s2.loli.net/2024/09/09/dUylAeP5b9B8Va4.png)
然后用计算器去算func call的位置

```
0x7FFAA11C0000 + 0x9BDA19 = 0x7FFAA1B7DA19
```

然后上断点
![Breakpoints](https://s2.loli.net/2024/09/09/d6re7K81AiY9JmC.png)
断点拉满，用来跟踪行为

然后我们就可以在微信上试试看效果了，比如自己发个消息，然后撤回。你可能要问，不应该是找人发消息然后撤回吗？其实自己发消息一样会写DB后触发撤回的流程，而且这个点也没别人了。

第一个停下的断点毫无疑问是call func，紧接着停下的断点是0x00007FFAA1B7CD07

然后是...触发超时了，难绷

只能说，菜就多练，再试试

最后跟到了断点0x00007FFAA1BACE6A，这里是个jne到0x00007FFAA1BACE6A，事实上就是在这里之后才完成了Revoke的动作，打开Cutter然后跟上去看看。（Byd Cutter分析的好慢）

事实证明我还是太年轻了，根本看不了一点.jpg, 人变菜了，感叹于这西北的风沙一点点填满了我大脑皮层的沟壑，有一种智慧的美🤪。

换个思路，咱还是以动态调试智取（菜）

## Chapter 5 又双叒叕没活整了，看一下Call stack和log

我们注意到，每次撤回一个消息的时候，他都会创建两个线程，有没有可能从这两个地方入手，看看有没有什么有价值的线索呢？
![SuspectThread](https://s2.loli.net/2024/09/09/emlAWpY1POMg7LB.png)
进Cutter看一下....没想法，告辞

## Chapter 6 兜兜转转回到起点

理论上来说应该还是按照最开始的思路，谁调用的就去找谁，还是得看0x09bd7d0

终于，在狠狠地找了一番Xref后找到了上一级的Func: 0x1997710 给我找吐了，真的。说实话，这个过程中很多的call并不在一个Func中，我也不知道为什么，可能是编译器喜欢Goto吧

不要犹豫，犹豫就会败北，果断上断点看看效果....byd居然没在断点停下来，只能说明在撤回的时候并不是从这个入口进入的0x09bd7d0

玛德不看Xref了，捏麻麻的直接上断点追Stack，急急急，菜！

```
下断点：0x7FFAA11C0000 + 0x09BD7D0 = 0x7FFAA1BAD7D0
```

问题是，X64DBG似乎没有查断点前调用的操作...吐了

干了4个小时了，没啥进展...洗个澡先换换脑子.jpg

突然发现在call 0x9bd7d0前必定call 0x9bafe0...... 阿巴阿巴没什么想法。

又试了试直接nop掉call 0x09bccc0,好像确实是没有recall msg，但是蛋疼的问题是：破坏了stack的结构，报个异常就直接闪退了......

## Chapter 7 问问大佬，上上科技，GPT启动！

问了问GPT4o，哪来的GPT4o？找的某宝卖家，6￥换5$的APIKey，而且人家API也对接好了。我怀疑是洗黑U的但是我没有证据，而且确实比原厂便宜😋

没有解决问题，但是提供了一个全新的思路：注意偏移量0x3312490的String:ChatRevokeMgr::revokeMsg，并不在.text段之中，并且是在Runtime下，调用偏移量0x13674D0的Function之前将其地址送到寄存器值中，作为fcn.0x13674D0的一个参数
![细看反汇编](https://s2.loli.net/2024/09/11/1SDRCFt3waEd4nI.png)
![细看反编译](https://s2.loli.net/2024/09/11/tjcbXA9HUCPyDf4.png)

这里不得不吐槽一点：我知道这个Function有这几个参数了，但是这个位置的arg不是确定是rdata.0x3312490了吗？

也就是说，我猜这个字符串的含义并非是一般的Symbols，不是编译器编译的能call的东西。这个Function类似一个解析器，解析字符串指令并执行后面对应的操作，switch这样的。

把所有的call fcn.0x13674D0的部分拿出来看看啥情况
```
fcn.1813674d0(0x183312410, 0x1833124b0, (int64_t)(LPFILETIME)&var_98h, (int64_t)&var_58h, (int64_t)&var_d8h, (int64_t)&var_88h, (int64_t)&var_78h, (int64_t)&var_68h);
fcn.1813674d0(0x183312410, 0x183312460, (int64_t)(LPFILETIME)&var_e8h, (int64_t)&var_68h, (int64_t)&var_78h, (int64_t)&var_88h, (int64_t)&var_c8h, (int64_t)&var_d8h);
fcn.1813674d0(0x183312410, 0x1833124e0, (int64_t)(LPFILETIME)&var_d8h, (int64_t)&var_88h, (int64_t)&var_78h, (int64_t)&var_68h, (int64_t)&var_58h, (int64_t)&var_98h);

0x183312410:ChatRevokeMgr
0x1833124b0:add to be_revoke_local_id_set_, LocalId=%lld
0x1833124e0:add to be_revoke_local_id_set_, LocalId=%lld, svrId=%lld
0x183312460(memory dump):cancelUploadMediaByMsgId ok, LocalId=%lld.......
    00007FFC05502460  63 61 6E 63 65 6C 55 70 6C 6F 61 64 4D 65 64 69  cancelUploadMedi  
    00007FFC05502470  61 42 79 4D 73 67 49 64 20 6F 6B 2C 20 4C 6F 63  aByMsgId ok, Loc  
    00007FFC05502480  61 6C 49 64 3D 25 6C 6C 64 00 00 00 00 00 00 00  alId=%lld.......  

```
注意这里并没有看到目标的rdata.0x3312490，因为地址在别的寄存器之中，反编译器没标出来，只能手动翻翻反汇编，看看var_XXh里面都是些什么好东西

```
var_98h:
        0x1809bdacb      movups xmm0, xmmword [data.1832cb828] ; 0x1832cb828
        0x1809bdaec      movdqa xmmword [var_98h], xmm0
        data.1832cb828->data.0x1832cc0ca  像是大块儿乱码（图片或是表格？），不是ASCII，以UTF-8解析直接给我把Cutter干闪退了





