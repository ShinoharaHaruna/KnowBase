---
title: Raft
date created: 2025-02-19 17:28:40
tags:
  - "#分布式系统"
  - "#Raft"
  - "#共识算法"
  - "#日志复制"
category: 计算机科学
---

# Raft 分布式共识协议详解

## 概述

Raft 是一种易于理解的分布式共识算法，旨在确保分布式系统中的数据一致性。它将共识问题分解为 Leader 选举、日志复制和安全性三个核心机制，广泛应用于分布式数据库和键值存储系统。

## 核心机制

### 1. Leader 选举

- **目标**：确保集群中始终有一个 Leader 负责处理客户端请求。
- **过程**：
  - 当 Follower 超时未收到 Leader 的心跳时，转变为 Candidate 并发起选举。
  - Candidate 向其他节点发送投票请求，获得大多数投票后成为 Leader。
  - 如果选举失败，Candidate 等待一段时间后重新发起选举。

### 2. 日志复制

- **目标**：确保所有节点拥有相同的日志副本，从而实现一致性。
- **过程**：
  - Leader 将客户端请求追加到自己的日志中，并将日志条目复制到所有 Follower。
  - 当日志条目被大多数节点复制后，Leader 将其标记为已提交，并应用到状态机中。

### 3. 安全性

- **目标**：通过 Term 和日志匹配特性，确保数据的一致性和正确性。
- **机制**：
  - 每个日志条目包含 Term 和 Index，用于标识其唯一性。
  - 如果两个日志条目具有相同的 Index 和 Term，则它们的内容相同。
  - 如果某个日志条目被提交，则所有之前的日志条目也必须被提交。

## 高级特性

### 1. Joint Consensus

- **目标**：处理动态成员变更（如节点加入或退出），避免脑裂和数据丢失。
- **过程**：
  1. Leader 将旧配置和新配置合并为一个联合配置，并将其复制到大多数节点。
  2. 在联合配置中，集群同时使用旧配置和新配置。
  3. 当联合配置被大多数节点接受后，Leader 切换到新配置。

### 2. 线性一致性读

- **目标**：确保读操作的一致性。
- **实现**：
  - **Read Index**：Leader 确保自己的日志是最新的，然后返回读结果。
  - **Lease Read**：Leader 在租约有效期内直接返回读结果，无需与其他节点通信。

### 3. 日志快照

- **目标**：减少日志存储开销，并加快节点恢复速度。
- **过程**：
  1. Leader 或 Follower 定期生成快照，将当前状态机状态保存到磁盘。
  2. Leader 将快照发送给日志落后的 Follower。
  3. Follower 接收快照后，将其应用到状态机中，并删除快照之前的日志条目。

## 优化策略

### 1. 日志复制优化

- **批量传输**：将多个日志条目打包成批量消息进行传输，减少网络通信次数。
- **Pipeline 复制**：允许 Leader 在等待 ACK 的同时继续发送新的日志条目，提高吞吐量。

### 2. 跨数据中心优化

- **区域 Leader**：在每个数据中心选举一个区域 Leader，负责本数据中心的日志复制。
- **P2P 同步**：允许已经同步的节点帮助其他节点同步日志，减少 Leader 的负载。

### 3. 多租户支持

- **日志隔离**：为每个租户分配独立的日志和状态机，确保数据隔离。
- **资源配额**：为每个租户分配独立的资源配额（如 CPU、内存、存储），避免资源竞争。

## 代码实现

### Leader 选举伪代码

```go
type NodeRole int

const (
    Follower NodeRole = iota
    Candidate
    Leader
)

type RaftNode struct {
    term        int
    role        NodeRole
    timeout     uint32
    votedFor    int // 记录当前 Term 内投票给哪个节点
    electionTimer *Timer
}

func (n *RaftNode) RequestVote() {
    if n.role == Follower && n.electionTimer.Timeout() {
        n.role = Candidate
        n.term++
        n.votedFor = n.id // 投票给自己
        // 向其他节点发送投票请求
        for _, peer := range n.peers {
            peer.Vote(n.term, n.id)
        }
        // 重置选举超时
        n.electionTimer.Reset()
    }
}

func (n *RaftNode) Vote(candidateTerm int, candidateId int) {
    if candidateTerm > n.term {
        n.term = candidateTerm
        n.votedFor = candidateId
        // 返回投票成功
        return true
    }
    // 返回投票失败
    return false
}

func (n *RaftNode) HandleVoteResponse(voteGranted bool) {
    if n.role == Candidate && voteGranted {
        n.votesReceived++
        if n.votesReceived > len(n.peers)/2 {
            n.role = Leader
            // 开始发送心跳消息
            n.startHeartbeat()
        }
    }
}
```

## 常见问题

Q：如果日志到达 Follower 但未持久化会怎样？

A：**日志丢失**：如果 Follower 在持久化之前崩溃，未持久化的日志条目会丢失。Leader 会检测到 Follower 的日志落后，并重新发送缺失的日志条目。

Q：如果 Leader 崩溃了怎么办？

A：**选举新 Leader**：Follower 超时后转变为 Candidate，并发起选举。新 Leader 会检测到其他节点的日志落后，并发送缺失的日志条目。

Q：如何实现多租户支持？

A：**日志隔离、资源配额**：为每个租户分配独立的日志和状态机，为每个租户分配独立的资源配额。

Q：任期（Term）的用处是什么？为什么要设置这个属性？

A：任期（Term）是 Raft 中用于区分不同领导周期的逻辑时间戳，它的作用包括：

- **标识 Leader 的合法性**：每个 Leader 都与一个唯一的任期号绑定，确保集群在同一时间内只有一个合法的 Leader。
- **解决冲突**：如果出现网络分区或节点故障，任期号可以帮助节点识别过期的 Leader 或 Candidate，避免数据不一致。
- **选举仲裁**：在选举过程中，节点只会投票给任期号大于或等于自己当前任期号的 Candidate。

Q：Term 在哪些情况下会被更新？

A：以下是 Term 会被更新的主要情况：

1. 节点收到更高 Term 的消息：当一个节点收到来自其他节点的消息（如心跳、日志复制请求或投票请求），如果消息中的 Term 大于自己的 Current Term，节点会更新自己的 Term 为消息中的 Term。
2. 节点发起选举：当 Follower 超时未收到 Leader 的心跳消息时，它会转变为 Candidate 并发起选举。在发起选举时，Candidate 会将自己的 Term 加 1。
3. 节点收到更高 Term 的选举结果：当 Candidate 收到来自其他节点的选举结果（如投票响应或新 Leader 的心跳消息），如果消息中的 Term 更高，Candidate 会更新自己的 Term 并转变为 Follower。
4. 节点发现日志冲突：当 Leader 发现 Follower 的日志与自己的日志冲突时，如果 Follower 的 Term 更高，Leader 会更新自己的 Term 并转变为 Follower。
5. 节点恢复后同步日志：当节点崩溃后恢复时，如果它发现自己的 Term 落后于集群中的其他节点，它会更新自己的 Term 并同步日志。

## 参考资料

- [Raft 论文](https://raft.github.io/raft.pdf)
- [Raft 可视化演示](https://raft.github.io/)
