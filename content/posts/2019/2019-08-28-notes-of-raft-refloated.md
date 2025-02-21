---
title: Raft Refloated Do We Have Consensus?笔记
date: 2019-08-28 16:32:52
categories:
    - "distritubed system"
tags:
    - replication
    - raft
---

# 介绍
文章主要使用实验验证了raft是否如原论文所阐述的易于理解和实现。重新阐述了raft，使用OCaml实现了raft，开发了一个事件驱动的模拟器来进行测试，重现了raft原论文中的测试，并提出了几个优化。

# 性能
对于raft的性能，raft原论文中提到了两点，
1. 典型的情况下，leader复制entry的时间。
2. 最坏情况下，leader失败后，选举出新leader的时间。
    1. 选举能否快速收敛？
    2. 集群能达到的最小下线时间是什么？

## 典型的情况下，leader复制entry的时间
需要耗费一个rtt来复制到多数派server，可以做batching和piplining优化。

## 最坏情况下，leader失败后，选举出新leader的时间
关于这点，raft原论文中使用5个server的集群做了两个测试。为了模拟最坏情况，
* 每次测试server都有不一样长度的log，使得有的candidate不能成为leader。
* 为了制造易于出现split vote的环境，在每次终止leader前，都同步地广播heartbeat，目的是重置election timeout。

![-w302](/images/2019/15674125053563.jpg)

两个测试表明了：
1. election time中加入少量的随机，能够明显的减少选举新leader的时间，减少split vote的出现。
2. 集群的最小下线时间随着election time的减少而减少，但如果少于broadcast time，那么会集群产生不必要选举，降低可用性。

# 优化
本文中提出了几个优化，
1. Different Follower and Candidate Timers
2. Binary Exponential Backoff
3. Combined

![-w293](/images/2019/15674141041720.jpg)

## Different Follower and Candidate Timers
raft原论文建议$\textit{candidate timeout} = \textit{follower timeout} ∼ U(T, 2T), T=150ms$，但在高竞争的情况下，例如：$U(150ms, 151ms)$，将两个时间设置在不同的范围，可以大大减少选举leader的时间。

## Binary Exponential Backoff
类似tcp超时重传的指数回退。
