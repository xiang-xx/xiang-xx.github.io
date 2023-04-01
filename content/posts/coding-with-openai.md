---
date: 2023-03-28T10:08:48+08:00
title: "使用 OpenAI 协作完成项目，好用到离不开"
description: ""
tags: ["AI", "ChatGPT"]
series: []
---

## 需求案例

最近有个需求，使用 ChatGPT 的 Embedding 技术实现自定义 AI 助手。

总体功能：用户上传 pdf 或者其他知识库，程序把用户上传的知识库调用 ChatGPT 的 Embedding api 进行向量化并存储到向量数据库。用户跟自己创建的 AI 助手对话时，根据用户的问题匹配向量数据库的内容，把匹配到的内容作为上下文知识，调用 ChatGPT 的 Completions api 完成对话补全。

总体功能类似此项目：[gpt4-pdf-chatbot-langchain](https://github.com/mayooear/gpt4-pdf-chatbot-langchain)

## 背景信息

ChatGPT 版本：ChatGPT-4

此项目要求使用 python 开发，基本的技术栈为：
- web 框架：fastapi
- 核心库：langchain
- python 虚拟环境包管理工具：poetry
- 向量存储库：chromadb/redisvector/pinecone

我对 python 不太熟，几年前自己接触过一点，写了一些 demo。那时候连 virtual env 都很少用。所以为了快速上手，很多知识点都需要问 ChatGPT。

## 协作开发过程

在选择向量存储工具时，我让 ChatGPT 对三种备选方案进行对比，分析其优劣势。ChatGPT 给了比较准确的回复，它还给出了一个简短的总结：

>总之，选择哪种技术取决于你的具体需求和应用场景。如果你需要一个简单的、轻量级的解决方案，Redis 可能足够了。然而，对于大规模的矢量搜索和相似度查询，Pinecone 可能是一个更好的选择。Chroma 可能适合某些特定需求，但可能需要更多的开发工作。

&nbsp;

在使用 pinecone 时，对 index 的两个参数 `dimensions` 和 `metrics` 的作用不太清楚，也不知道如何设置。通过咨询 ChatGPT，能很快地得到答案（这里截取部分内容）：

>- Dimensions 指的是矢量的维度。在创建 Pinecone 索引时，需要指定矢量的维度。这个维度需要与您的矢量数据集的实际维度保持一致。矢量维度的选择取决于特征提取和表示方法。对于某些场景，较低的维度可能足以捕捉到数据的重要特征；而对于其他场景，较高的维度可能有助于提高搜索准确性。在实际应用中，可以通过实验和评估来确定合适的矢量维度。
>
>- Metrics 是用于度量矢量之间相似度的方法。Pinecone 支持多种相似度度量，如欧几里得距离（euclidean）、余弦相似度（cosine）和内积（dot-product）等。相似度度量的选择取决于您的应用场景和数据类型。例如，在处理文本数据时，余弦相似度通常是一个不错的选择；而在处理图像或其他连续数据时，欧几里得距离可能更合适。在实际应用中，可以通过实验和评估来确定合适的相似度度量。

后来我查询了 pinecone 的文档，发现 ChatGPT 总结的很到位。

&nbsp;

在编写 python 代码时，因为我不太熟悉相关语法和库，所以基本是边写边 ChatGPT，比如：
- 如何创建、初始化对象
- 什么是 pydantic 模型（一个用于数据验证和设置管理的 Python 库）
- 如何使用 sqlalchemy 进行增删改查，如何设置基类
- vscode 如何设置 poetry 的 venv
- python 类型互转、hex encode 等
- python 如何使用 Ed25519
- pip wheel 有什么用
- fastapi 设置可选路径参数、可选 query 参数、接收上传文件
- 如何递归创建目录
- 把中文错误提示翻译成英文错误提示
- docker 相关命令
- k8s volumes 映射单个 config 文件

ChatGPT 给的代码相当准确，复制到项目里基本就能跑。

&nbsp;

在使用过程中，ChatGPT 也出现过一些小问题，通常是因为知识不足，或知识歧义。

我为了测试 Pinecone 的向量存储服务，开通了标准账户。测试完毕后我打算取消掉 Pinecone 的付款计划（billing plan）。

Pinecone 如何取消付款计划？最初我在 pinecone 上没有找到取消付款计划的入口。ChatGPT 说：需要联系 Pinecone 的支持团队。发送一封电子邮件至 support@pinecone.io....

于是我让 ChatGPT 帮我写了封邮件（比高中英语小作文的范文还要好）:

>Subject: Request to Cancel Subscription Plan
>
>Dear Pinecone Support Team,
>
>I hope this email finds you well. I recently subscribed to one of your payment plans for the purpose of evaluating Pinecone for a company project. Unfortunately, I mistakenly used my personal credit card for the subscription.
>
>As a result, I kindly request that you cancel my current subscription plan. I understand that I may need to provide additional information to verify my identity and confirm the cancellation. Please let me know if there are any further steps I need to take to complete this process.
>
>I appreciate your assistance in this matter, and I look forward to hearing from you soon.
>
>Best regards,
>
>[Your Full Name]
>
>[Your Email Address]
>
>[Your Pinecone Account Details, if applicable]

但后来我在 Pinecone 的网站上找到了降级付款计划的按钮... 因为我眼拙没看到此按钮，再加上 ChatGPT 的误导，导致我以为 Pinecone 账号降级只能找官方支持团队，这期间扣了我十几块钱。

&nbsp;

我最开始问 chroma 的 index 相关信息时，它说 chroma 是一个 Python 颜色库，没有索引文件。后来我查了下，确实有这样一个 python 库。当我问 **chroma 向量数据库的 index** 相关信息时，它就回答准确了。

## 总结

ChatGPT 的知识内容非常丰富，特别是 ChatGPT4 已经包含了非常新的内容。它能够在概念性问题上提供快速且准确的帮助，在进行辅助决策时，可以针对自己关心的方面多深入问问，比如上述例子中，深入对比各个向量数据库的准确性、性能、存储引擎实现逻辑、扩展性等。

它非常善于编程，它生成的代码复制到项目里基本都能运行。在开发过程中，可以把复杂任务拆分出子流程，给 ChatGPT 提供上下文及实现要求（函数签名及注释、参数注释等），让它实现后在项目里微调即可；已有的代码可以让它润色，或添加一些额外功能，比如给 go struct fields 添加各种 tag。

交流的时候，尽可能带有明确的上下文。因为 ChatGPT 的知识面太广了，就像我直接问他 chroma 时，我并不知道有一个颜色库也叫 chroma，而它以为我问的是这个颜色库；当我以“向量存储 chroma”进行询问时，它就知道我问的真正是什么了。


## 后记

在没有 ChatGPT 的年代，遇到问题时，通常使用搜索引擎，或查询对应的开发文档；而如今，使用 ChatGPT 能**更快速精准**的找到想要的信息。ChatGPT 在信息咨询、辅助编程、邮件、周报等各种场景下都能表现得很好，从面向 google 编程到面向 ChatGPT 编程的开发流程转变确实已经到来。

但 chat 无法掌握知识，copy 不能提升技能。工具只能用来提高效率，人不能被工具掌控。日常工作所需的技术栈依然需要通过文档、源码进行知识补充，使用起来才能得心应手，否则完全依赖 ChatGPT 的代码无法逃出踩坑的命运。
