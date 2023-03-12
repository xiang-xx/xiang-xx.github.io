---
date: 2023-03-01T19:26:36+08:00
title: "Raft 协议核心内容"
description: "Raft 论文核心内容摘要"
tags: ["阅读"]
series: []
---

## 关于 raft

Raft[^1] 是一种共识算法，它能够使得系统成员部分失效后，系统仍然能够工作。本质上是一个备份状态机，它具有以下特性：
- 安全性：永远不会返回错误数据（尽管存在拜占庭故障、网络分区、延迟、丢包等）
- 高可用：只要大多数节点存活（互相之间能通讯，能被客户端访问到），系统就是可用的
- 共识日志不依赖时间（因为系统可能存在时间故障、严重延迟等）
- 一般情况下，只要集群大多数响应命令完成，则此命令就会完成，避免少量慢节点影响整体服务性能

Raft 部分思想受 Paxos、zab 等协议启发，但 Raft 协议简单很多：作者在论文中多次强调 Paxos 过于复杂，他们花了一年时间才真正搞懂 Paxos 究竟是如何工作的。而 Raft 协议的初衷就是创建一个更简单的共识算法，更简单意味着可以被更多人学习、理解，能更好的在生产中实现、使用。

[这个网站](http://thesecretlivesofdata.com/raft/)[^2]使用分步动画形式很好的解释了 raft 的运行方式，通过它可以对 raft 协议有大致的认识。

## Raft 共识算法概要

Raft 算法要求先选出 leader，然后 leader 接收客户端的 log entries，复制给其他 servers，并告诉它们什么时候可以把 log entries 应用到它们的状态机。

Raft 协议把共识算法分解成三个互相无依赖的子问题：
- Leader election：当现有的 leader 失效后，必须选举出新 leader
- Log replication：leader 从客户端接收 log entries，并复制到集群的其他 servers 上
- Safety：如果一个 server 把某个 log entry 应用到它的状态机，则集群其他 server 与此 log entry 相同的 log index 上，使用的都是同一个 log entry.

## Raft 详细内容

### 所有服务器的持久状态（persistent state）：
- currentTerm: 服务器上次看到的 term
- votedFor: 当前 term 收到投票的服务器ID，如果没有，是 null
- log[]: log entries，从 leader 接收到的每个 entry 都包含状态机的命令，和当时的 term

### 所有服务器的不稳定状态 （volatile state）：
- commitIndex: 已知需要提交的最高 log 的 index
- lastApplied: 已经应用到状态机的最高 log 的 index

### leader 服务器的不稳定状态（选举后重新初始化）
- nextIndex[] 需要发送给特定 server 的下一个 entry 的 index（初始化为 leader last log index + 1）
- matchIndex[] 已知在每个 server 上已复制的 index

### AppendEntries Rpc
leader 调用用来备份 entries；同时也是用来作为心跳的 rpc
参数：
- term: leader 的 term
- leaderId: follower 可以据此重定向 client 的请求
- entries[]: 同步的命令，可以为空
- prevLogIndex: entries 前一个 entry 的索引
- prevLogTerm: prevLogIndex 的 term
- leaderCommit  leader 的提交索引

响应：
- term：接收端 currentTerm，leader 可以根据响应更新自己的状态。（比如 leader 落后 term，变为 follower）
- success: 如果 follower 包含对应的 prevLogIndex 和 prevLogTerm，则是 true

接收端实现：
1. 如果 term < currentTerm，return false
2. 如果 prevLogIndex， prevLogTerm 不存在，则 return false
3. 如果已存在的 entry 与 new entries 冲突（相同 index，不同 term），删除现有的 entry，使用 leader 的 entries
4. 新的 entries 全部 append 到 log 中
5. 如果 leaderCommit > commitIndex, 把 commitIndex 设置为 min(leaderCommit, index of last new entry)

### RequestVote Rpc
候选人调用，获取选票。

参数：
- term: 候选人的 term
- candidateId: 候选人id
- lastLogIndex: 候选人最后一条 log entry 的 index
- lastLogTerm: 候选人最后一条 log entry 的 term

响应：
- term: currentTerm，候选人根据此 term 更新自己
- voteGanted: 为 true 表示候选人得到了选票

接收端实现：
1. 如果 term < currentTerm，则 return false
2. 如果 votedFor 是 null 或者 candidateId，且候选人日志至少是最新的（比投票人的 lastLogIndex 更新或相同），投票为 true

### Servers 的规则
All Servers:
- 如果 commitIndex > lastApplied，增加 lastApplied，并把 `log[lastApplied]` 应用到状态机
- 如果 RPC 请求或响应存在 term T > currentTerm，设置 currentTerm = T，并设置为 follower

Followers:
- 响应 candidate 和 leader 的 rpc 调用
- 如果选举超时结束，且没有收到 leader 的 AppendEntries RPC 和 candidate 的 RequestVote RPC，则变为 candidate

Candidate:
- 当变为 candidate 后，开始选举
  - 增加 currentTerm
  - 投票给自己
  - 重设 election timer
  - 向其他 server 发送 RequestVoteRpc
- 如果收到大多数 server 的选票，变为 leader
- 如果收到新 leader 的 AppendEntries RPC，则变为 follower
- election timer 超时，重新开启选举

Leader:
- 选举时，发送空 AppendEntriesRPC 到每个 server；空闲期间重复发送，避免选举超时
- 收到 client 的命令后：把 entry 追加到本地 log，当 entry 被应用状态机后返回
- 如果 follower 的 last log index >= nextIndex，AppendEntryRpc 的 entries 从 nextIndex 开始发送
  - 如果 success，更新记录 follower 的 nextIndex 和 matchIndex
  - 如果 fail，减小 nextIndex，并重试
- 如果存在 N， N > commitIndex，且大多数的 matchIndex[i] >= N, 且 log[N].term == currentTerm，set commitIndex = N

### 安全性保证

通过以上逻辑，Raft 可以实现下面的安全性保证。

选举安全：每个 term 只能有一个 leader 被选举出来。

Leader 日志仅追加（Append-Only）: leader 不会修改或删除它的本地 Log，仅追加。

日志匹配（Log Matching）: 如果两个 logs 有相同的 index 和 term，则这两个 logs 里所有的 entry 都是相同的。

Leader 完备（Leader Completeness）: 如果在特定 term 提交了一个 entry，则在所有更高 term 的 leader 里，它都是存在的。

状态机安全：如果一个 server 已经 apply a entry 到状态机，则相同的 index 下，其他 server apply 的是同一个 entry。

Leader 只会通过计数本任期 term 下产生的 entry 的备份数量，来提交 entry（设置 commitIndex）。


## 其他

时间与可用性：广播时间 << 选举超时时间 << 崩溃间隔。内网广播时间（请求响应一来一回的时间）通常是几毫秒内；崩溃间隔通常几天、几月甚至更长。选举超时时间一般随机设置在 150ms - 300ms，太长会影响 leader 崩溃后系统的恢复速度，太短则可能频发引发选举。

集群成员改变（增减）：使用 config log entry 同步成员信息，保证配置信息共识。成员初次加入需要同步数据，同步数据时没有投票权，不占大多数计算逻辑；直到追上日志后才能投票。

日志压缩：快照模式，各个 server 管理自己的快照。快照后，前面的 log entries 可以删除。如果 leader 给 follower 需要同步的日志已删除，需要接口先同步快照，再同步后面的日志。同步快照使用 InstallSnapshot RPC。

客户端交互：客户端随机选择节点发起请求，如果是 follower 节点，则节点拒绝请求，并返回 leader 信息。客户端向 leader 发起请求。

为了避免 **leader 已提交但未响应 client 后崩溃，client 重新请求新 leader 节点会导致命令多次执行**，每个 command 需要带一个唯一标识，新 leader 包含所有已提交 entry，遇到相同的唯一标识的 command，直接返回已执行即可。

[^1]: [Diego Ongaro and John Ousterhout Stanford University](https://raft.github.io/raft.pdf)

[^2]: [Raft: Understandable Distributed Consensus](http://thesecretlivesofdata.com/raft/)

