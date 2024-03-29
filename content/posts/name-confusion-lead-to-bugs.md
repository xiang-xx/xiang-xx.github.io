---
date: 2023-05-22T20:19:19+08:00
title: "术语命名混乱问题及解决方案"
description: ""
tags: ["code"]
series: []
---

## 前言

在日常开发中，当你看到代码里的 `userId` 变量时，你无需检查代码的上下文，就猜到它代表的是 `user` 表的主键 `id`。但有时候出于各种原因，导致变量实际代表的意义并非我们所想的那样，如果我们按照自己认为的那样进行编码，则可能造成严重问题。

## 我遇到的典型案例

### 1.

某教育公司的 `teacher` 表里有三个数字字段分别是：`id`，`teacher_id`，`teacher_uid`；其中，`id` 是自增主键，`teacher_id` 是发号器生成的全局唯一 `id`；`teacher_uid` 是绑定的用户 `id`。

当代码里、或其他表字段里出现一个 `teacher_id` 时，如果不查看上下文，你很难知道它代表的是 `teacher` 表的哪个字段。曾经就有一位研发在建表时使用 `teacher_id` 代表了 `teacher_uid`，导致某些功能查询 `teacher` 信息失败。

在设计表时，如果决定使用递增全局唯一 `id` 时，就没必要再保留主键自增了，把从发号器拿到的全局唯一 `id` 作为主键存储即可，这样就避免了 `teacher_id` 的语义问题。再规范使用 `teacher_id` 与 `teacher_uid`，则可以避免此类字段引起问题。

### 2.

某电子商务公司的支付系统的各个服务中，`customer_id` 经常用来表示企业用户 `id`，企业信息、账单、合规认证等各个系统表里及代码里都是用 `customer_id` 表示企业用户 `id`。

后来在开发一个数据统计功能时，某新人研发要根据**订单概要表**统计企业销售数据，代码逻辑是根据此表的 `customer_id` 进行聚合查询等操作。临上线时发现聚合得到的数据跟其他系统的数据对不上，后来调查发现此处的 `customer_id` 是消费者用户 `id`，并非企业 `id`。

同一系统中，同一个单词只能代表一种主体，必须确保术语的语义唯一。

### 3.

某互联网公司的系统中有如下三张表： `swap_result(id, ...)`， `swap_record(id, ..., swap_result_id)`，`swap_extrinsic(id, ..., swap_result_id)`。

后面两张表里都有相同数据类型的 `swap_result_id`。使用其中的 `swap_result_id` 去 `swap_result` 表中按主键查询，什么也查不到，就算有能查到的数据，但各种场景都对不上。后来翻代码才发现，`swap_record.swap_result_id` 是 `swap_extrinsic.id`，而 `swap_extrinsic.swap_result_id` 是 `swap_record.id`，它们跟 `swap_result` 表毫无关联！

通常在设计外键关联时，只需要根据业务逻辑把一张表的外键关联另一张表即可。就算是一对一的关系，也无需两张表互相关联。且这里的字段命名存在很大的词不达意问题。


## 避免术语不一致

术语混淆问题带来的 bug 通常很隐晦，难以在编码阶段发现。而当测试场景不充分时，在测试阶段也可能无法发现这些 bug。一旦上到生产环境，可能造成严重问题：比如使用了错误的 id 导致数据被覆盖，被误删，新增了错误数据等。

避免术语混淆应从最早的时间开始。

### 从立项开始

在立项初期，项目负责人就应该根据项目涉及到的主体，分别确定其术语，最好附带其英文翻译，这样可以做到多方（前端、后端、客户端等）的命名一致。在立项会或产品评审阶段，各方会对术语达成一致，在各自的文档和代码中使用相同的命名，便于代码阅读、接口理解。

### 技术文档与技术评审

服务端的技术文档通常要包含以下核心要素：
- 数据库表设计，建表语句
- 缓存字段命名、值类型
- 接口详细定义

有的系统对代码质量要求较高，可能还需要再技术文档里说明大致的代码框架：
- 接口定义
- 关键类、字段

在评审这些与编码直接相关的内容时，应严格把控字段命名：
- 与立项定义的关键术语命名一致
- 外键命名规范：主键表_id
- 缓存字段名确保见名知意；缓存 key 前缀，避免多项目 key 冲突
- 关键接口、类、字段术语一致；不要过于简写

### 编码

长一点的名字更能准确表达意思。应少用 tmpXXX 作为作用域较广的变量，而是根据其场景，使用更能体现其意义的名字。

函数通常是与其他工程师的代码进行交互的入口，所以函数参数的命名更加重要。重要的入口函数加上完整的注释，能够避免很多沟通上的问题；重构函数时也要记得修改注释。

接口、类的 public 字段等，同样如此。

### 代码审查

代码审查是把控代码质量的最后一道关口，需要认真执行，不能流水账。

通常提测时就应该开始安排代码审查，上线前再根据上次审查后的变更记录进行增量的审查。

这样可以给审查留出足够的时间，而不是临上线时，着急上线导致流水账式的审查，导致漏掉一些隐晦的问题。提前审查出的不规范、代码结构不合理等问题，也可以给研发留出充足的时间进行重构，而不是临上线时重构，导致需要重新走一遍测试流程（有些着急上线不重新测试，经常会遇到重构出 bug 的情况）。

## 总结

术语混淆使用导致的问题通常隐晦且对系统有破坏性，在日常工作中需要重视。

应从立项开始，就对系统的术语进行规范；在项目进行的关键环节，对关键命名进行评审把控，同时提升研发人员素质，才能减少此类故障发生的可能性。
