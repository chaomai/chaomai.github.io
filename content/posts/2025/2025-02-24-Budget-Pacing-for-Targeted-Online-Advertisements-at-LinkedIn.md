---
title: Bid Optimization for Offsite Display Ad Campaigns on eCommerce笔记
date: 2025-02-24 13:23:57
categories:
    - paper-notes
tags:
    - computational-advertising
    - budget-pacing
    - dsp
---

[📕Paper Notes](/2024/2024-11-02-paper-notes)

# Background

广告投放中，对每次展示机会都竞价的方式对广告主和平台并不是最优的，这个方式会快速消耗完表现好的广告主的预算，影响广告主的体验；同时由于二价的机制，有竞争力的广告主预算很早消耗完了，降低了竞争的环境，影响平台收入。

改出价的方式受限于 min mrp 的影响，因此可能也没法有效的延长 campaign 的时间。

这篇文章提出了 probabilistic throttling，从一些随机选出来的 req，把效果好的推广活动给筛掉。

# Pacing algorithm

假设$S$是用于的全集，campaign $i$的当日预算为$d_i$，出价为$b_i$，对子集$S_i$定向。每日划分为$T$个等长的时间窗口，$s_{i,t}(0\le t \lt T)$表示 campaign $i$当日截止时间窗$t$的累积消耗。$f{i,t}$是$s_{i,t}$对应的预测 imp，$f_{i,T}$是当日预测总 imp。$a_{i,t}$表示 campaign $i$当日截止时间窗$t$的预计消耗。

为了实现计划消耗速度和曝光曲线保持一致，

$$a_{i,t}=\frac{f_{i,t}}{f_{i,T}}d_i$$
对于时间窗内的每次竞价，按$p_{i,t}$（pass through rate，PTR）判断是否参竞，这个值在每个时间窗口起始时更新，$0 \le r_t \le 1$时一个调节系数。

$$
p_{i,t} = \begin{cases} p_{i,t-1} * (1 + r_t) \land 1 & \text{if } s_{i,t} \le a_{i,t} \\ p_{i,t-1} * (1 - r_t) \lor 0 & \text{if } s_{i,t} > a_{i,t} \end{cases}
$$

campaign $i$的预算曲线$a_{i,t}$在每日开始时已经确定，消耗曲线$s_{i,t}$通过修改 PTR 来控制，目标是使得$s_{i,t}$接近$a_{i,t}$。

另一篇 paper 可以通过 bid modification 实现 pacing，

$$
b^*_i = b_i \psi(s_{i,t} / d_i) \text{ where, } \psi(x) = 1 - e^{x-1}
$$

前面已经讨论过为什么使用 probabilistic throttling 而非 bid modification。这里随着$s_{i,t}$的增加，$b^*_i$减少至 0，起到 pacing 的作用。但是实际受限于 min mrp 的限制，且导致出价调控和预算平滑耦合。

这里如果使用$p^*_{i,t} = \psi(s_{i,t}/d_{i})$作为 PTR，论文实验结果是消耗提升了，但并未显著延长 campaign 的投放时长。

# 实现细节

## 更新频率

每个 campaign 的 PTR 1 分钟更新一次，高频的更新可以保证系统的灵敏度，并快速达到稳定状态。

## Imp 预测

imp 曝光预测的准确性对该算法的有效性至关重要。论文讲述了方法。

1. 维护了一个用户集合 list，叫 base profiles。
2. 对于每个 base profile，统计 ad 的历史 imp 时间序列数据。
3. 用组合统计时序模型，每天都去拟合每个 base profile 最新的时间序列数据。
4. 使用模型，来得到每个定向用户群的 imp 预测结果。

## 调节系数

论文中采用的调节系数为固定值 10%，实现较为简单且鲁棒性强。理论上应该使用消耗的变化率$\partial s_{i,t} / \partial t$。消耗快的 campaign 需要更多的调节参竞率，消耗慢的则需要较小的调节。

没使用使用这个复杂方法的原因有两点，

1. $\partial s_{i,t} / \partial t$可能是噪声。预算消耗曲线并不平滑，尤其是对于 cpc 计费的 campaign，clk 远少于 imp，而曲线仅在有 clk 时才变化。
2. PTR 的更新频率高，就算当前 PTR 不是理想值也可以快速的调节。

## Slow start

论文中对于所有 cqmpaign 的 PTR 初始值设置为 10%，即 slow start。可以让系统有足够的时间调节每个计划到理想值。对于消耗过快的 campaign，可能在达到理想值之前就把预算消耗完。

论文提到最好是根据每个 campaign 所需的 imp（花完推广预算需要的曝光次数）和能提供的 imp 来计算（campaign 能用的总曝光次数）来设置 PTR 的初始值。

## Fast finish

由于每个imp的曝光量是使用模型预估出来的，可能存在统计错误。进而导致预算无法消耗完。解决方案是将每天最后两个小时的imp设置为0，让算法尽可能在22h内华完预算。
