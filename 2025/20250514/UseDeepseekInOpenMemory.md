---
title: 在Android设备上使用BBR拥塞控制算法

date: 2025-5-15 19:00:00

cover: https://s2.loli.net/2024/10/05/D8cHzbgr2Ej4y5p.png

tags:

  - Mem0

  - DeepSeek

  - OpenMemory MCP

  - LLM

  - MCP

  - CherryStudio
categories:

  - 整活

keywords: Mem0,DeepSeek,OpenMemory MCP, LLM ,MCP,CherryStudio
description: Use Deepseek R1/V3 in OpenMemory MCP, CherryStudio as front
---

# OpenMemory MCP中使用DeepSeek替代OpenAI

## Chapter 1 前情提要👀

好久没整好活了，最近的工作非常饱满（Nethunter Kernel鸽了...），但是有很多明显可以用AI完成的部分，于是乎在各种等设备IO的water时间，狠狠地技术储备，准备手搓一个AI平台来分流water成分大的体力劳动，把精力投入到更有创造力的部分。毕业以来很久没正经的摸过AI相关的应用了，这个领域半年猪突猛进看来还是有不少狠活，特别是本期主角，[Mem0](https://github.com/mem0ai/mem0/tree/main)和他的[OpenMemory MCP](https://github.com/mem0ai/mem0/tree/main/openmemory) （后者刚上线没几天就给我碰到了^_^）

## Chapter 2 补习功课：MCP✍️
很早之前，我看到过一个demo，就是openai展示新GPT模型通过格式化json数据调用外部工具的能力；这种操作拉高了LLM的使用场景上限，解决了很多痛点，就比如说LLM数学口算稀烂，现在可以直接调用计算器了，再有甚者去调用[Wolframalpha](https://www.wolframalpha.com/),当年学高数的时候体验过一把（用来水作业），现在多模态AI后更是直接薄纱了。再往后出现了更多的开源方案，比如当年群发邮件也给我发了一份的[Llama3.3](https://ollama.com/blog/structured-outputs)。显然，这个用“教会速读大傻逼用计算器”的方案有很大的潜力。

[MCP](https://modelcontextprotocol.io/introduction)就是在这种美好憧憬下诞生的，不多废话直接放原文：
```
MCP is an open protocol that standardizes how applications provide context to LLMs. Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect your devices to various peripherals and accessories, MCP provides a standardized way to connect AI models to different data sources and tools.
```
很快啊很快，MCP就让众多开发者达成共识，由于接口标准统一，业务逻辑清晰，SDK好使，很快就占据了半壁江山。特别是有了SDK之后，为特定的场景定制“计算器”给LLM用方便了非常多，复用/分享的时间精力成本相比于之前json数据还有后端一把抓要打骨折，香啊很香啊。

这让定制化LLM使用场景从以前的只能花难以承受的时间精力成本做Fine-tune写code处理输入输出变成只需要prompt写好一点会用MCP调用工具就好了（当然带chain of thought的LLM也是立大功）。

## Chapter 3 让LLM突破Content Window的限制：Vector DB
众所周知，所有Transformer都有一个content window，可以简单粗暴的理解为最大上下文长度。就算是Claude这样一骑绝尘的长对话强者，事实上也是记忆力捉急；并且“记忆力”只存在于当前对话之中，并不能跨对话（手动复制不算）。为了解决这个问题，特别是针对多个文件多轮对话的问题，大佬们想到了使用Vector DB，即向量数据库：通过将上下文使用tokenizer向量化然后整理总结存入Vector DB中，下次启动新的对话就从Vector DB中先搜索一遍相关内容，再将输出写入当前对话，这样LLM就假装有了长期的通用记忆。

注意到三个重要环节，总结归纳向量化/Vector DB/Search Memory，这三者的有机结合决定了最后LLM记忆的水平。这方面成熟的产品貌似挺多的，但是本期的mem0最近是非常的火爆。Paper是看不懂的，star连接大脑，clone代替思考😋。

## Chapter 4 Mem0+MCP = OpenMemory MCP
在长期记忆的基础上，我们又很容易根据实际使用场景提出一个现实需求：共享记忆。毕竟MCP的工具类不能真的老是停留在计算器水平，肯定有高级应用还是要根据场景来的，比如说一些数据分析，对联想要求高的人物场景；AI的另一条赛道，智能体（Agent），共享记忆的需求肯定也是有的吧，赛博黑奴共享大脑协作打工，想想就是生产力大解放。

好，持久化记忆池有了，通信有了，二合一，OpenMemory MCP，启动！

首先还是clone一下repo，然后进入到repo根目录下的openmemory文件夹下，然后根据readme的指引一波make三连
``` bash
make build
make env
make up
```
理论上就跑起来了，还有一个不错的web控制台😋
但是啊但是，openai的token那是真的金贵，肯定是不能放在这种巨大消耗的地方用的。还是看看code：
``` python
MEMORY_CATEGORIZATION_PROMPT = """Your task is to assign each piece of information (or “memory”) to one or more of the following categories. Feel free to use multiple categories per item when appropriate.

- Personal: family, friends, home, hobbies, lifestyle
.........
- Goals: ambitions, KPIs, long‑term objectives

Guidelines:
- Return only the categories under 'categories' key in the JSON format.
- If you cannot categorize the memory, return an empty list with key 'categories'.
- Don't limit yourself to the categories listed above only. Feel free to create new categories based on the memory. Make sure that it is a single phrase.
"""
```
还有的code就不放了，code大意是每次memory的增删都会调用LLM原汤化原原食，那是用不起一点

好在code是开源的，我们还有更经济的选择：Deepseek-R1  （别跟我说本机Ollama跑蒸馏小模型，模型小了效果差，大一点的模型要是能跑起来显卡钱购买几年的token了）

## Chapter 5 修改代码使Openmemory后端使用deepseek(Part1)
[Deepseek的API](https://api-docs.deepseek.com/)完全与openai兼容,理论可行，实践开始！

我也不知道为啥没有人做这个工作，太简单了？我来太早了？不管怎么说，自己动手丰衣足食！

火速打开VS Code然后查看Openai的调用部分(写的时候是[这个版本](https://github.com/mem0ai/mem0/commit/a22287a3bac2648b832821ffaa095a106a902e98))

在/api/app/utils/categorization.py之中
``` python
openai_client = OpenAI()
class MemoryCategories(BaseModel):
    categories: List[str]
@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=15))
def get_categories_for_memory(memory: str) -> List[str]:
    """Get categories for a memory."""
    try:
        response = openai_client.responses.parse(
            model="gpt-4o-mini",
            instructions=MEMORY_CATEGORIZATION_PROMPT,
            input=memory,
            temperature=0,
            text_format=MemoryCategories,
        )
  ...
```
怪诶，他这个openai_client在init的时候为啥没传key进去呢？看了看openai python SDK的[README](https://github.com/openai/openai-python?tab=readme-ov-file)找到了答案
``` python
import os
from openai import OpenAI

client = OpenAI(
    # This is the default and can be omitted
    api_key=os.environ.get("OPENAI_API_KEY"),
)

response = client.responses.create(
    model="gpt-4o",
    instructions="You are a coding assistant that talks like a pirate.",
    input="How do I check if a Python object is an instance of a class?",
)

print(response.output_text)
```
其实就是直接从环境变量里面读

那后面的简单了，修改环境变量和接口地址，剩下的就看DeepSeek-R1能不能支持这些API的功能了！

``` bash
echo 'OPENAI_API_KEY=<DEEPSEEK token>' > ./app/.env
```

然后用Code的全局搜索，找到所有openai client实例init的时候，传入参数
``` python
###其实就这一个./api/app/utils/categorization.py
openai_client = OpenAI(base_url="https://api.deepseek.com")
```
接下来跟openai_client的使用，把参数的model换成deepseek的,比如：

``` python
response = openai_client.responses.parse(
    model="deepseek-reasoner",
    instructions=MEMORY_CATEGORIZATION_PROMPT,
    input=memory,
    temperature=0,
    text_format=MemoryCategories,
)
```
CoT模型合不合适我还不知道，但是doc上说是支持格式化输出的，理解不深，abaaba

## docker-compose 启动！
还有个准备项，根据[官方readme](https://github.com/mem0ai/mem0/tree/main/openmemory)，先make，然后还有一个env要填:./ui/.env,不过我不知道这个UI是干啥的，不管了不关键😋

直接一个docker compose up,冲刺！冲刺！打开lazydocker可以发现mcp,storage和ui都跑起来了,我就不贴图了

## 在CherryStudio中使用OpenMemory MCP
本次选择的测试方案是[CherryStudio](https://github.com/CherryHQ/cherry-studio),有原生的MCP支持

我这里的应用场景：学/逆一个大型的项目，人脑分析很吃力，多个code文件塞给AI会超token;但是如果AI的记忆力强大的话，能解决很多问题，可以跨很长的时间跨度链接线索

还有一些参数得往里面写，但是不知道为什么，反正网上没有相关的资料，有几个参数折腾半天才知道该填什么

最后还是在Web Dashboard里面找到了答案：使用SSE（Server-Sent-Events）

最后在lazydocker的log里面找到了接口(docker 部署的)，拼接起来就是
``` bash
http://127.0.0.1:8765/mcp/openmemory/sse/hairenjun
```

点击保存，顺利通过了CherryStudio添加MCP的可用性检查！现在随便问个问题来看看效果！
![实际测试](https://tc.z.wiki/autoupload/7VYis3AlxQoq81hqTtnIGOgPd76tONEmzmEaoJ60APayl5f0KlZfm6UsKj-HyTuv/20250614/o32X/2458X1514/image.png)

可以看到确实调用了MCP工具，但是在MCP工具内部，出现了一些连接问题，赶紧查看一下lazydocker的log

![容器日志](https://tc.z.wiki/autoupload/7VYis3AlxQoq81hqTtnIGOgPd76tONEmzmEaoJ60APayl5f0KlZfm6UsKj-HyTuv/20250614/d7VD/2508X1294/image.png)

看来是个单纯的网络问题，直接exec shell进去看一下怎么个事儿
``` bash
root@81281d05245e:/usr/src/openmemory# ping api.deepseek.com
PING f25089e3f8624be8804c1a64aa4c7043.vip1.huaweicloudwaf.com (116.205.40.113) 56(84) bytes of data.
64 bytes from ecs-116-205-40-113.compute.hwclouds-dns.com (116.205.40.113): icmp_seq=1 ttl=48 time=62.0 ms
64 bytes from ecs-116-205-40-113.compute.hwclouds-dns.com (116.205.40.113): icmp_seq=2 ttl=48 time=73.6 ms
64 bytes from ecs-116-205-40-113.compute.hwclouds-dns.com (116.205.40.113): icmp_seq=3 ttl=48 time=68.2 ms
64 bytes from ecs-116-205-40-113.compute.hwclouds-dns.com (116.205.40.113): icmp_seq=4 ttl=48 time=66.4 ms
^C
--- f25089e3f8624be8804c1a64aa4c7043.vip1.huaweicloudwaf.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 62.040/67.558/73.626/4.157 ms
```

怪诶...不会是仍然在fetch openai然后GFW发力了吧

事实证明是我小丑了，光写了blog没写code

这里再推荐另一种方法，就是.env法，没跑起来compose前可以直接在.env里面添加
``` bash
OPENAI_API_KEY=sk-你的key
OPENAI_API_BASE=
OPENAI_BASE_URL=
```
跑起来了后直接exec shell进去添加
``` bash
nano /usr/src/openmemory

OPENAI_API_BASE=https://api.deepseek.com/
OPENAI_BASE_URL=https://api.deepseek.com/
```
记得把选择模型的那个code也改了，deepseek-reasoner

然后在lazydocker里面重启一下container

结果又出了400的问题...虽然cherrystudio的log里面MCP返回的是Model not found，但是细看docker的log发现是400 Bad Request，那我就不得不怀疑一下本身对deepseek的支持性了，还是回到下面的code
``` python 
response = openai_client.responses.parse(
    model="deepseek-reasoner",
    instructions=MEMORY_CATEGORIZATION_PROMPT,
    input=memory,
    temperature=0,
    text_format=MemoryCategories,
)
```
对比一下官方api示例仅有message的参数，emmm，为了验证我的猜想，我直接在本地创建一个新的python脚本，复现一下这个请求。

``` python
import json
from openai import OpenAI
from typing import List
from pydantic import BaseModel
MEMORY_CATEGORIZATION_PROMPT=MEMORY_CATEGORIZATION_PROMPT = """Your task is to assign each piece of information (or “memory”) to one or more of the following categories. Feel free to use multiple categories per item when appropriate.

- Personal: family, friends, home, hobbies, lifestyle
- Relationships: social network, significant others, colleagues
- Preferences: likes, dislikes, habits, favorite media
- Health: physical fitness, mental health, diet, sleep
- Travel: trips, commutes, favorite places, itineraries
- Work: job roles, companies, projects, promotions
- Education: courses, degrees, certifications, skills development
- Projects: to‑dos, milestones, deadlines, status updates
- AI, ML & Technology: infrastructure, algorithms, tools, research
- Technical Support: bug reports, error logs, fixes
- Finance: income, expenses, investments, billing
- Shopping: purchases, wishlists, returns, deliveries
- Legal: contracts, policies, regulations, privacy
- Entertainment: movies, music, games, books, events
- Messages: emails, SMS, alerts, reminders
- Customer Support: tickets, inquiries, resolutions
- Product Feedback: ratings, bug reports, feature requests
- News: articles, headlines, trending topics
- Organization: meetings, appointments, calendars
- Goals: ambitions, KPIs, long‑term objectives

Guidelines:
- Return only the categories under 'categories' key in the JSON format.
- If you cannot categorize the memory, return an empty list with key 'categories'.
- Don't limit yourself to the categories listed above only. Feel free to create new categories based on the memory. Make sure that it is a single phrase.
"""

openai_client = OpenAI(api_key="sk-XX", base_url="https://api.deepseek.com/v1/")
class MemoryCategories(BaseModel):
    categories: List[str]
print(openai_client.chat.completions.create( 
            model="deepseek-reasoner", 
            messages=[
                {"role": "system", "content": MEMORY_CATEGORIZATION_PROMPT},
                {"role": "user", "content": "你的名字是Murasame"}
            ],
            temperature=0,
            response_format={"type": "json_object"}  # 
        ))

def get_categories_for_memory(memory: str) -> List[str]:
    """Get categories for a memory."""
    try:
        response = openai_client.responses.parse(
            model="deepseek-reasoner",
            instructions=MEMORY_CATEGORIZATION_PROMPT,
            input=memory,
            text_format=MemoryCategories,
        )
        response_json =json.loads(response.output[0].content[0].text)
        categories = response_json['categories']
        categories = [cat.strip().lower() for cat in categories]
        # TODO: Validate categories later may be
        return categories
    except Exception as e:
        raise e
if __name__=="__main__":   
    print(get_categories_for_memory("你的名字是Murasame"))
```
随手一写就是一坨，但是不管怎么说，只有使用openai_client.chat.completions.create才有结果，MCP所使用的openai_client.responses.parse直接404，这就实锤了，这玩意在代码上就没有支持Deepseek，或者应该说是deepseek没做responses的API适配？

欸我测怎么这么难搞，我还以为就是换个参数的事情

## Chapter 5 修改代码使Openmemory后端使用deepseek(Part2)
没办法，这里的方法只能自己实现了。直接上code
``` python

def get_categories_for_memory(memory: str) -> List[str]:
    try:
        response = openai_client.chat.completions.create(
            model="deepseek-chat",
            messages=[
                {
                    "role": "system",
                    "content":MEMORY_CATEGORIZATION_PROMPT
                },
                {
                    "role": "user",
                    "content": f"记忆内容: {memory}"
                }
            ],
            temperature=0,
            response_format={"type": "json_object"},  
            max_tokens=500
        )

        response_content = response.choices[0].message.content

        try:
            response_json = json.loads(response_content)
            categories = response_json.get('categories', [])

            if not isinstance(categories, list) or not all(isinstance(c, str) for c in categories):
                raise ValueError("Invalid categories format")

            return [cat.strip().lower() for cat in categories]
            
        except (json.JSONDecodeError, ValueError) as e:
            print(f"JSON解析错误: {e}\n原始响应: {response_content}")
            return []
            
    except Exception as e:
        print(f"API调用失败: {str(e)}")
        return []  

```

**（2025-6-12更小丑的是，我在写这篇markdown的时候有人commit把这个问题解决了）**

**结果还是踏马的400**，真的吐了，不过也不是一点没收获，用复现的脚本发现，模型名称不对会返回400，但响应体是


``` json
{'error': {'message': 'Model Not Exist', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_request_error'}}
```
为什么会出这个BYD问题我真不知道了,想了一下，还是老老实实的看一下deepseek官方对这个400 code是怎么定义的
```
400 - Invalid Format	Cause: Invalid request body format.
Solution: Please modify your request body according to the hints in the error message. For more API format details, please refer to DeepSeek API Docs.
```
此外，还有一个沟槽的问题：
``` bash
https://api.deepseek.com/embeddings - 404
``` 
embedding没用的话，那我VectorDB直接嘻嘻开摆,蛋疼的是deepseek的Doc对这部分内容一点没说，我测，怪不得这帮人宁愿用token比金子贵的openai也不来做deepseek的适配，直接copy的样例name也400，终成小丑，妈的受不了了

## Chapter 6 曲线救国，改用ollama上跑的deepseek-r1:8b
刚好这两天deepseek发布了 DeepSeek-R1-0528，据说8B的蒸馏版本有巨大的提升，也原生支持Tools，不妨试一试，而且这下数据全部不上网，安全隐私还有保证，四舍五入就是赢（）

4060的速度非常的快，比官方满血还快，token性能上反正不是问题了

结果没想到，还是400，这就非常令人怀疑了，因为invoke openai class时候可是传base url进去了的，那如果我把ollama关掉，理应直接超时，试一试：居然还是400！这说明这个endpoint根本就没配置上！捏麻麻的,重新docker compose up之后确实不是400了，直接time out了。。。。。

原因有一，ollama默认监听localhost，没监听docker的虚拟网卡，拿curl测了一下，很容易就debug出来了，解决的话只要在service里面写个env就好

第二个原因则非常非常的坑，看一下下面这段代码：
``` python
try:
    db = SessionLocal()
    db_config = db.query(ConfigModel).filter(ConfigModel.key == "main").first()
    
    if db_config:
        json_config = db_config.value
        
        # Extract custom instructions from openmemory settings
        if "openmemory" in json_config and "custom_instructions" in json_config["openmemory"]:
            db_custom_instructions = json_config["openmemory"]["custom_instructions"]
        
        # Override defaults with configurations from the database
        if "mem0" in json_config:
            mem0_config = json_config["mem0"]
            
            # Update LLM configuration if available
            if "llm" in mem0_config and mem0_config["llm"] is not None:
                config["llm"] = mem0_config["llm"]
                
                # Fix Ollama URLs for Docker if needed
                if config["llm"].get("provider") == "ollama":
                    config["llm"] = _fix_ollama_urls(config["llm"])
            
            # Update Embedder configuration if available
            if "embedder" in mem0_config and mem0_config["embedder"] is not None:
                config["embedder"] = mem0_config["embedder"]
                
                # Fix Ollama URLs for Docker if needed
                if config["embedder"].get("provider") == "ollama":
                    config["embedder"] = _fix_ollama_urls(config["embedder"])
    else:
        print("No configuration found in database, using defaults")
            
    db.close()
```

注意到一个函数_fix_ollama_urls，咱们跟进去看看
``` python
def _fix_ollama_urls(config_section):
    """
    Fix Ollama URLs for Docker environment.
    Replaces localhost URLs with appropriate Docker host URLs.
    Sets default ollama_base_url if not provided.
    """
    if not config_section or "config" not in config_section:
        return config_section
    
    ollama_config = config_section["config"]
    
    # Set default ollama_base_url if not provided
    if "ollama_base_url" not in ollama_config:
        ollama_config["ollama_base_url"] = "http://host.docker.internal:11434"
    else:
        # Check for ollama_base_url and fix if it's localhost
        url = ollama_config["ollama_base_url"]
        if "localhost" in url or "127.0.0.1" in url:
            docker_host = _get_docker_host_url()
            if docker_host != "localhost":
                new_url = url.replace("localhost", docker_host).replace("127.0.0.1", docker_host)
                ollama_config["ollama_base_url"] = new_url
                print(f"Adjusted Ollama URL from {url} to {new_url}")
    
    return config_section
```
看到什么了？就是这个
``` python
 # Set default ollama_base_url if not provided
    if "ollama_base_url" not in ollama_config:
        ollama_config["ollama_base_url"] = "http://host.docker.internal:11434"
```
在Linux的docker中，这个host.docker.internal并不存在！只有win和mac用户的docker才会起效！导致docker直接找不到host，直接timeout

还有就是这个，那我问你，我host哪去了（）
```
[06/14/25 08:58:00] INFO     Retrying request to            _base_client.py:1058
                             /chat/completions in 0.458712                      
                             seconds                                            
[06/14/25 08:58:05] INFO     Retrying request to            _base_client.py:1058
                             /chat/completions in 0.974097                      
                             seconds                               
```
来传统艺能，一行行跟栈
``` python
response = memory_client.add(text,
                                user_id=uid,
                                metadata={
                                "source_app": "openmemory",
                                "mcp_client": client_name,
                            })
...
_memory_client = Memory.from_config(config_dict=config)
...
config = _parse_environment_variables(config)
...
config = get_default_memory_config()

# Variable to track custom instructions
db_custom_instructions = None

# Load configuration from database
try:
    db = SessionLocal()
    db_config = db.query(ConfigModel).filter(ConfigModel.key == "main").first()
    
    if db_config:
        json_config = db_config.value
```
其中 get_default_memory_config()是写死的，实话实说我搞不明白直接从.config读就行的事情搞这么麻烦干嘛，应该确保这玩意按doc能跑得起来，代码精简好二开，再去给README都看不懂的哥们作冗余。

所以关键从来不是那个Openai实例，出问题的一直是这个沟槽的db config，怪不得之前怎么改config文件都出问题。这应该是大佬的思维超出我的水平了我理解不了，为了性能什么的（）

所以多半是哪个本应该是用来兜底的函数一不小心弄巧成拙把host搞没了

直接在main里面加上下面的代码启动uvicorn服务器，方便debug
``` python
import uvicorn
if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8765, log_level="info")
```