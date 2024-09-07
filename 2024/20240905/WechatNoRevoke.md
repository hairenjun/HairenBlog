---
title: X64DBG+Cutter+CE动静结合三板斧玩转微信

date: 2024-09-7 19:00:00

cover: https://s2.loli.net/2024/03/13/5OCLuMwKsbgTor4.png

tags:

  - WeChat

  - 微信

  - 逆向工程

  - Cutter

  - X64DBG

  - CheatEngine

  - Reverse Engineering

categories:

  - 逆向实况

keywords: WeChat,Reverse Engineering,Cutter
description: Use Cutter/CheatEngine/X64DBG reverse Wechat and make a dll patch to disable "recall" function
---


#  X64DBG+Cutter+CE动静结合三板斧逆向微信推出防撤回补丁

##  Chapter 1 前情提要👀

微信群里有爆典高手，但是每次看消息都很不及时，于是看着recalled message无能为力...那必不可能！于是萌发了想办法做一个防撤回补丁的想法，Just Do It！

##  Chapter 2 思路整理😋

主要思路还是用CE去抓一下到底是哪个Function在Recall这个message。一个message被Recall的流程，从接收到网络包开始，处理网络消息，写DB，最后在用户界面上把消息撤回去

我们希望的是，在处理网络消息与写DB之间找到个关键Function，禁用它，使其无法处理后面的流程，这样就实现了防撤回的功能

