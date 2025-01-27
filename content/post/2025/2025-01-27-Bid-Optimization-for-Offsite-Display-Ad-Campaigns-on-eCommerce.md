---
title: Bid Optimization for Offsite Display Ad Campaigns on eCommerce笔记
date: 2025-01-27 23:57:52
categories:
    - computational-advertising
tags:
    - roas
    - dsp
---

[📕Paper Notes](/2024/2024-11-02-paper-notes)

## 背景

作者们从广告主视角介绍了一个竞价优化系统，目标是在 DSP 投放广告时， 将 ROAS（Return on Ad Spend）最大化。该系统解决了广告主在尝试优化 ROAS 出价时遇到的几个问题。

-   缺少信息共享
-   手动调价
-   Clicks 和 Revenue 低相关性
    -   由于**转化效果的滞后性**，点击类指标与展示广告系列的收入相关性较低。因此，**以点击量为优化目标并不能有效提升收入**
-   竞价环境带来的预估挑战
-   缺少透明的预算和消费控制

## Dimensional Bidding

为了解决这些问题，作者提出了一种多维度竞价方法，该方法会根据广告请求的不同细分 segment 进行出价调整。

### Segmentation

首先将展现人群划分为子人群，即 dimension。dimension 是指可以定向、屏蔽或调整出价的定向条件。这样便于追踪每个 dimension 的效果也便于探索。

### Overlapping Dimensions

#### Assumptions

系统依赖如下几个假设：

1. CPM 正比于 bid factor
    - base bid \* bid factor 是实际出价 bid price
    - campaign 的 cost 也会随着 bid factor 的增加而增加
2. win 的数目（或者 imp）正比于 bid factor
3. RPM 独立于 bid price，且在短时间内相对稳定
    - 如果我们 RPM 来选择 dimension，那么每个 dimension 的平均收益不应该取决于 cost。

#### Disjoint Bid Dimensions

基于以上假设，可以通过定义多个互斥的 dimension 并为每个 dimension 设置一个 bid adjustment。对于第 i 个 dimension，定义如下参数，

-   Cost per (mille) impression: $CPM_i$
-   Revenue per (mille) impression: $RPM_i$
-   Number of won impressions: $n_i$
-   Total ads spend: $S_i$
-   Total revenue: $R_i$
-   Bid (adjustment) factor: $f_i$

这样需要解决的优化问题如下，

$$
\begin{equation}
\begin{aligned}
    \max_{f_i} \quad & \sum_{i=1}^{m} RPM_i \cdot g(f_i) \\
    \text{s.t.} \quad & \sum_{i=1}^{m} S_i \leq B
\end{aligned}
\end{equation}
$$

其中$B$为总预算，$n_i=g(f_i)$是用于表示维度 𝑖 曝光量的模型。不失一般性，假设基础 bid 为$b_0=1$。使用一个简单的线性模型可以认为$n_i = g(f_i) = \beta_i \cdot f_i$。

|          | Geo1                | Geo2                | Geo3                | Total                        |
| -------- | ------------------- | ------------------- | ------------------- | ---------------------------- |
| website1 | $n_1 = \beta_1 f_1$ | $n_2 = \beta_2 f_2$ | $n_3 = \beta_3 f_3$ | $\sum_{i=1}^{3} \beta_i f_i$ |
| website2 | $n_4 = \beta_4 f_4$ | $n_5 = \beta_5 f_5$ | $n_6 = \beta_6 f_6$ | $\sum_{i=4}^{6} \beta_i f_i$ |
| website3 | $n_7 = \beta_7 f_7$ | $n_8 = \beta_8 f_8$ | $n_9 = \beta_9 f_9$ | $\sum_{i=7}^{9} \beta_i f_i$ |
| ...      | ...                 | ...                 | ...                 | $\sum_{i=1}^{9} \beta_i f_i$ |

又因为 GFP 中，$CPM_i \approx b_0 \cdot f_i = f_i$。综上可以得到如下关系，

$$
\begin{equation}
\begin{aligned}
	& S_i = n_i \cdot CPM_i = \beta_i \cdot f_i^2 \\
	& R_i = RPM_i \cdot n_i = RPM_i \cdot \beta_i \cdot f_i = RPM_i \cdot \sqrt{\beta_i \cdot S_i}
\end{aligned}
\end{equation}
$$

优化问题转化为，

$$
\begin{equation}
\begin{aligned}
    \max_{f_i} \quad & \sum_{i=1}^{m} RPM_i \cdot \beta_i \cdot f_i \\
    \text{s.t.} \quad & \sum_{i=1}^{m} f_i^2 \cdot \beta_i \leq B.
\end{aligned}
\end{equation}
$$

每个$f_i$的解析解如下，

$$
\hat{f}_i = \sqrt{\frac{RPM_i^2}{\sum_{j=1}^{m} RPM_j^2 \cdot \beta_j} \cdot B}.
$$

这个方法有两个弊端，

1. 维度爆炸。
    - 需要为每个 segment 都分配一个系数。
    - 系数的数量为，$\Theta\left(I^K\right)$，其中$K$是维数，$I$是每个 dimension 中组的数量。
2. 如果 segment 的数量很大，那数据将过于系数，将无法有效的计算 $RPM_i$ ，$g(f_i)$。

#### Overlapping Dimensions

这里改为每个 group 分配系数，而非每个 segment，参数的数量减少到$\Theta\left(I \cdot K\right)$。$f_j^i$表示第$i$个维度的第$j$个 group。

|                    | Geo1 ($f_1^2$)                    | Geo2 ($f_2^2$)                    | Geo3 ($f_3^2$)                    | Total                                       |
| ------------------ | --------------------------------- | --------------------------------- | --------------------------------- | ------------------------------------------- |
| website1 ($f_1^1$) | $-$                               | $-$                               | $-$                               | $n_1^1 = g(f_1^1, f_{[1,2,3]}^2)$           |
| website2 ($f_2^1$) | $-$                               | $-$                               | $-$                               | $n_2^1 = g(f_2^1, f_{[1,2,3]}^2)$           |
| website3 ($f_3^1$) | $-$                               | $-$                               | $-$                               | $n_3^1 = g(f_3^1, f_{[1,2,3]}^2)$           |
| Total              | $n_1^2 = g(f_1^2, f_{[1,2,3]}^1)$ | $n_2^2 = g(f_2^2, f_{[1,2,3]}^1)$ | $n_3^2 = g(f_3^2, f_{[1,2,3]}^1)$ | $= \sum_{i=1}^3 n_i^1 = \sum_{i=1}^3 n_i^2$ |

假设 bid factor 是可以分解的$f_l = f_w^1 \cdot f_z^2$，$l$表示两个 dimension 的其中一组不相交的维度，此时每个 segment 的 bid factor 由两个 group 的 bid factor 决定$b = b_0 \cdot f_w^1 \cdot f_z^2$。

每个 dimension 的 cost 和总 cost 如下，其中$K$和$I$分别表示维数和每个 dimension 的 group 数（假设都相同），

$$
\begin{equation}
\begin{aligned}
	S^i & = \sum_{j=1}^{I} S^{i}_j = B \\
	\sum_{i=1}^{K} S^i & = K \cdot B.
\end{aligned}
\end{equation}
$$

由于不同 dimension 的 group 存在相互重叠，因此每个 segment 的数据可以使用更多数据来表示。此时优化问题变为，

$$
\begin{equation}
\begin{aligned}
    \max_{f_i^k} \quad & \frac{1}{K} \sum_{k=1}^{K} \sum_{i=1}^{I_k} n_i^k \cdot RPM_i^k \\
    \text{s.t.} \quad & \frac{1}{K} \sum_{k=1}^{K} \sum_{i=1}^{I_k} n_i^k \cdot CPM_i^k \leq B.
\end{aligned}
\end{equation}
$$

CPM 使用一个简单线性模型表示，$CPM_i^k = a + b \cdot f_i^k$。

展现量使用如下模型表示，最后一个项表示其他 dimension 的 bid factor 的影响。

$$n_i^k = g(f_1^1, \dots, f_I^K) = a^k + \left( \beta_i^k \cdot f_i^k \right) \cdot \left( \prod_{d \neq k} \sum_{j=1}^{I} \beta_j^d \cdot f_j^d \right)$$

## Dimension Selection and Evaluation

简单来说作者选择 dimension 的原则是，能够在不同 group 之间最大化 RPM 差异的 dimension。以便进行有效的 bid 调整。

### Grouping Methods

#### Random

使得展现量均匀，但无法保证 RPM 在各个 group 之间的区分度。

#### Two-step heuristic

1. Sort all $v \in V$ based on $\text{RPM}(v)$ in ascending order, resulting in a sorted list: $V' = \{ v'_1, v'_2, \ldots, v'_n \} \quad \text{such that} \quad \text{RPM}(v'_1) \leq \text{RPM}(v'_2) \leq \ldots \leq \text{RPM}(v'_n)$
2. Assign Group Labels:
    - Initialize: $g = 1 \quad \text{(group label)}, \quad I_{\text{cumulative}} = 0 \quad \text{(cumulative impression volume)}.$
    - For each $v'\_i \in V'$:
        - Assign the current group label $ g $ to $v'\_i$.
        - Update the cumulative impression volume: $I_{\text{cumulative}} \leftarrow I_{\text{cumulative}} + I(v'_i)$
        - If $I_{\text{cumulative}} \geq I_{\text{min}}$
          then:
            - Increment the group label: $g \leftarrow g + 1$
            - Reset the cumulative impression volume: $I_{\text{cumulative}} \leftarrow 0$

#### 其他

-   Hierarchical grouping
-   Categorical grouping
-   Time-based grouping

### Evaluation of Dimensions and Groups

#### RPM Distances

比较不同 group 之间 RPM 的趋势（每日 RPM），选择差异较大的。

$$
r(a, b) = \frac{
    \sum_{t=1}^{T} 1\left( RPM_{\text{grp}}^t a > RPM_{\text{grp}}^t b \right) + 1
}{ \sum_{t=1}^{T} 1\left( RPM_{\text{grp}}^t a \leq RPM_{\text{grp}}^t b \right) + 1 }
$$

$$
pairwise\_distance(a, b) = \left| \sum_{t=1}^{T} \left( RPM_{\text{grp}}^t a - RPM_{\text{grp}}^t b \right) \right| + \lambda \cdot \frac{\max \left\{ r, \frac{1}{r} \right\}}{T}
$$

#### Information Value

特征选择中的方法之一。

|       | 原始                                                                                                                                                                                                                                                                  | 修改版                                                                                                                                                                    |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $WOE$ | $WoE_{\text{category}_i} = \ln \left( \frac{\left( \frac{\text{non\_events}_{\text{category}_i}}{\text{non\_events}_{\text{total}}} \right)}{\left( \frac{\text{events}_{\text{category}_i}}{\text{events}_{\text{total}}} \right)} \right)$<br>                      | $Modified \, WOE_i = \left[ \log \left( \frac{\% \text{conversions}_i}{\% \text{impressions}_i} \right) \right] \times 100\%$                                             |
| $IV$  | $IV_{\text{category}_i} = \sum \left( \left( \frac{\text{non\_events}_{\text{category}_i}}{\text{non\_events}_{\text{total}}} \right) - \left( \frac{\text{events}_{\text{category}_i}}{\text{events}_{\text{total}}} \right) \right) \times WoE_{\text{category}_i}$ | $IV = \sum_{i=1}^{n} \left( \% \text{conversions}_i - \% \text{impressions}_i \right) \times \log \left( \frac{\% \text{conversions}_i}{\% \text{impressions}_i} \right)$ |

#### Mutual Information

前面两个方法主要用于在某个 dimension 中评估多个 group。

评估多个 dimension 时，使用互信息。特征选择中的方法之一。

$$
MI(g_i, g_j, Y) = \sum_{(i,j) \in (g_i, g_j)} \sum_{y \in Y} p(i,j,y) \cdot \log \left( \frac{p(i,j,y)}{p(i,j) \cdot p(y)} \right)
$$

其中，

-   $g_i$为第$i$个 dimension 的 label
-   $p(i,j)$是来自两个 dimension 的两个 group 的展现次数的联合分布密度函数
-   $p(i, j, y)$是$(g_i, g_j, Y)$的联合分布密度函数
