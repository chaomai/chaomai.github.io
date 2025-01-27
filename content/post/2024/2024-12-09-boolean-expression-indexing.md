---
title: Boolean Expression Index
date: 2024-12-09 11:23:00
categories:
    - computational-advertising
tags:
    - boolean-expression-indexing
---

[📕Paper Notes](/2024/2024-11-02-paper-notes)

## 索引构建

1. 将 DNF BE 切分为 conjunctions。每个 conjunctions 是包含$\in$或者$\notin$的 predicates。
2. 将 conjunctions 根据 size K（$\in$的数量）进行分割，每组叫做 K-index。
3. 对于每个 K-index，为所有的可能的(attribute, value)对创建 posting list。(attribute, value)存储到 hashtable 中，便于为每个 assignment 检索相应的 posting list。
4. posting list 的每个 entry 由(conjunction id, bit)组成，表示这个 key 所在的 conjunction，以及是否为$\in$。
5. 每个 posting list 按 conjunction id，$\notin \lt \in$排序。
6. 所有的 posting list 按首个 entry 排序。

如果 conjunction 没有$\in$，则大小为 0，用$Z$表示。

entry 的总数正比于$|\mathcal{C}| \times P_{\text{avg}}$，$|\mathcal{C}|$是 conjunction 的数量，$P_{\text{avg}}$是平均每个 conjunction 的 predicate 的数量。

假设有 Figure 1 中的 6 个 conjunction，索引构建以后的结果如 Figure 2 所示。

**Figure 1: A set of conjunctions**

| ID    | Expression                                                                     | $K$ |
| ----- | ------------------------------------------------------------------------------ | --- |
| $c_1$ | $age \in \{3\} \land state \in \{\text{NY}\}$                                  | 2   |
| $c_2$ | $age \in \{3\} \land gender \in \{\text{F}\}$                                  | 2   |
| $c_3$ | $age \in \{3\} \land gender \in \{\text{M}\} \land state \notin \{\text{CA}\}$ | 2   |
| $c_4$ | $state \in \{\text{CA}\} \land gender \in \{\text{M}\}$                        | 2   |
| $c_5$ | $age \in \{3, 4\}$                                                             | 1   |
| $c_6$ | $state \notin \{\text{CA}, \text{NY}\}$                                        | 0   |

**Figure 2: Inverted list for Figure 1**

| $K$ | Key & UB                         | Posting List                                  |
| --- | -------------------------------- | --------------------------------------------- |
| 0   | $(\text{state}, \text{CA}), 2.0$ | $(6, \notin, 0)$                              |
|     | $(\text{state}, \text{NY}), 5.0$ | $(6, \notin, 0)$                              |
|     | $Z, 0$                           | $(6, \in, 0)$                                 |
| 1   | $(\text{age}, 3), 1.0$           | $(5, \in, 0.1)$                               |
|     | $(\text{age}, 4), 3.0$           | $(5, \in, 0.5)$                               |
| 2   | $(\text{state}, \text{NY}), 5.0$ | $(1, \in, 4.0)$                               |
|     | $(\text{age}, 3), 1.0$           | $(1, \in, 0.1), (2, \in, 0.1), (3, \in, 0.2)$ |
|     | $(\text{gender}, \text{F}), 2.0$ | $(2, \in, 0.3)$                               |
|     | $(\text{state}, \text{CA}), 2.0$ | $(3, \notin, 0), (4, \in, 1.5)$               |
|     | $(\text{gender}, \text{M}), 1.0$ | $(3, \in, 0.5), (4, \in, 0.9)$                |

## DNF 算法

这两个观察能有助于理解，

1. 对于一个 $K$-index（$K \leq t$），一个包含 $K$ 项的 conjunction $c$ 匹配 $S$ 的条件是，必须恰好存在 $K$ 个 posting list，其中每个 list 对应 $S$ 中的一个 key $(A, v)$，并且 $c$ 的 ID 在这些列表中具有 $\in$ 。
2. 对于 $S$ 中的任何 $(A, v)$ key，均不应存在 $c$ 以 $\notin$ 出现在 posting list 中的情况。

### 示例

给定一个 Assignment $S : \{\text{age} = 3, \text{state} = \text{CA}, \text{gender} = \text{M}\}$，在匹配 Figure 2 的倒排以后，结果如 Figure 3 所示。

**Figure 3: Posting lists for assignment $S$**

| $K$ | Key                         | Posting List                   |
| --- | --------------------------- | ------------------------------ |
| 0   | $(\text{state}, \text{CA})$ | $(6, \notin)$                  |
|     | $Z$                         | $(6, \in)$                     |
| 1   | $(\text{age}, 3)$           | $(5, \in)$                     |
| 2   | $(\text{age}, 3)$           | $(1, \in), (2, \in), (3, \in)$ |
|     | $(\text{state}, \text{CA})$ | $(3, \notin), (4, \in)$        |
|     | $(\text{gender}, \text{M})$ | $(3, \in), (4, \in)$           |

下面以 K=2 的 posting list 起始，当前的 entry 如下所示。

**Figure 4: Posting lists for $K=2$**

| Key                         | Posting List                               |
| --------------------------- | ------------------------------------------ |
| $(\text{age}, 3)$           | $\underline{(1, \in)}, (2, \in), (3, \in)$ |
| $(\text{state}, \text{CA})$ | $\underline{(3, \notin)}, (4, \in)$        |
| $(\text{gender}, \text{M})$ | $\underline{(3, \in)}, (4, \in)$           |

此时发现 PLists[0].CurrEntry.ID 和第 PLists[K-1].CurrEntry.ID 不相同，接下来第 0~K-1 posting list 的 conjunction id 都 skip 到 PLists[K-1].CurrEntry.ID。即第 1 个 posting list：1 -> 3。skip 之后重新排序，得到 Figure 5。

1. 为什么比较第 1 个和第 K-1 个 entry 的 conjunction id？
    - 由于 posting list 是按行和列排序的，因此只需要判断当前 conjunction size 的头尾
2. 为什么直接跳到 PLists[K-1].CurrEntry.ID？
    - 中间的 conjunction id 数量肯定不够 K 个

**Figure 5: Posting lists for $K=2$ after first skipping**

| Key                         | Posting List                               |
| --------------------------- | ------------------------------------------ |
| $(\text{state}, \text{CA})$ | $\underline{(3, \notin)}, (4, \in)$        |
| $(\text{age}, 3)$           | $(1, \in), (2, \in), \underline{(3, \in)}$ |
| $(\text{gender}, \text{M})$ | $\underline{(3, \in)}, (4, \in)$           |

这时 PLists[0~K]所有的 currnet entry 都有相同的 id，但 PLists[0].CurrEntry 是$\notin$。conjunction id 为 3 的肯定不会再满足要求，reject current conjunction ，跳过所有 conjunction id 为 3 的 entry。排序后得到 Figure 6。

**Figure 6: Posting lists for $K=2$ after second skipping**

| Key                         | Posting List                                           |
| --------------------------- | ------------------------------------------------------ |
| $(\text{state}, \text{CA})$ | $(3, \notin), \underline{(4, \in)}$                    |
| $(\text{gender}, \text{M})$ | $(3, \in)$, $\underline{(4, \in)}$                     |
| $(\text{age}, 3)$           | $(1, \in), (2, \in), (3, \in), \underline{\text{EOL}}$ |

找到 conjunction 4 满足条件。

### algorithm

**Input:** inverted list $idx$ and assignment $S$
**Output:** set of IDs $O$ matching $S$

1. $O \gets \emptyset$
2. For $K = \min(\text{idx.MaxConjunctionSize}, |S|)...0$ do:
    - _/\*List of posting lists matching $A$ for conjunction size $K$\*/_
    - $PLists \gets \text{idx.GetPostingLists}(S, K)$
    - $\text{InitializeCurrentEntries}(PLists)$
    - _/\*Processing $K=0$ and $K=1$ are identical\*/_
    - If $K = 0$ then $K \gets 1$
    - _/\*Too few posting lists for any conjunction to be satisfied\*/_
    - If $PLists.\text{size()} < K$ then:
        - Continue to the next for loop iteration
    - While $PLists[K-1].\text{CurrEntry} \neq \text{EOL}$ do:
        - $\text{SortByCurrentEntries}(PLists)$
        - _/\*Check if the first $K$ posting lists have the same conjunction ID in their current entries\*/_
        - If $PLists[0].\text{CurrEntry.ID} = PLists[K-1].\text{CurrEntry.ID}$ then:
            - _/\*Reject conjunction if $a \notin$ predicate is violated\*/_
            - If $PLists[0].\text{CurrEntry.AnnotatedBy}(\notin)$ then:
                - $\text{RejectID} \gets PLists[0].\text{CurrEntry.ID}$
                - For $L = K...(PLists.\text{size()}-1)$ do:
                    - If $PLists[L].\text{CurrEntry.ID} = \text{RejectID}$ then:
                        - _/\* Skip to smallest ID where $ID > \text{RejectID}$ \*/_
                        - $PLists[L].\text{SkipTo}(\text{RejectID}+1)$
                    - Else:
                        - Break out of the for loop
                - Continue to the next while loop iteration
            - Else: _/\* Conjunction is fully satisfied \*/_
                - $O \gets O \cup \{PLists[K-1].\text{CurrEntry.ID}\}$
            - _/\* NextID is the smallest possible ID after current ID \*/_
            - $\text{NextID} \gets PLists[K-1].\text{CurrEntry.ID} + 1$
        - Else:
            - _/\* Skip first $K-1$ posting lists \*/_
            - $\text{NextID} \gets PLists[K-1].\text{CurrEntry.ID}$
        - For $L = 0...K-1$ do:
            - _/\* Skip to smallest ID such that $ID \geq \text{NextID}$ \*/_
            - $PLists[L].\text{SkipTo}(\text{NextID})$
3. Return $O$

其中 For $L = K...(PLists.\text{size()}-1)$ do，这里比较奇怪，按示例中的 Figure 5 -> Figure 6 的过程，应该是遍历所有 posting list，skip 当前 conjunction id 为 reject id 的，但按原文中的算法执行，则只是 skip 第K个之后的posting list。这样执行后，会导致死循环。

## References

[Indexing Boolean expressions](https://www.vldb.org/pvldb/vol2/vldb09-83.pdf)
