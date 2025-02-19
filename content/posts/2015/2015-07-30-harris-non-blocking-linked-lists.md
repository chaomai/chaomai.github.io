---
title: "Harris' Non-Blocking Linked-Lists"
date: 2015-07-30 23:30:59
description: Harris的这篇论文提出了一种新的non-blocking linked-list，不同于Valois使用auxiliary node，Harris在操作的时候进行了mark，解决了插入丢失的问题。论文中有详细的伪代码，清晰的描述了实现的细节。但要注意的是，实际实现必然涉及到内存回收，没有自动内存回收机制的语言会有点麻烦。
categories:
    - concurrency
tags:
    - 'linked list'
    - reading
---

# specification

* ordered（ascending）
* no duplicated key

# features

* non-blocking
* linearizable
* compare-and-swap based

# 仅使用一个cas的缺陷

* insert
    ![](/images/2015/harris_single_insert.png)

* delete
    ![](/images/2015/harris_single_delete.png)

对于只有一个insert或者一个delete的情况，没有问题会出现。

* insert and delete
    ![](/images/2015/harris_insert_and_delete.png)

但是如果一个insert和一个delete同时进行，问题就会出现。删除30的时候，一个cas无法保证，也不能避免10和30之间的修改。

# 解决方法：用两个cas

## 基本思想

* stage 1
    ![](/images/2015/harris_logically_delete.png)
    用一个cas mark将要被删除结点的next field（logically deleted）；

* stage 2
    ![](/images/2015/harris_physically_delete.png)
    另一个cas来进行删除结点（physically deleted）。

stage 1结束以后，list的结构保持不变，mark以后的结点，避免了新节点insert到该结点的后面。

# algorithm

## marked and unmarked

> A node is marked if and only if its next field is marked.

这句话是关键：论文中的mark，实际上是mark了要被删除结点的next指针，而不是要被删除的结点本身。
我觉得从另一个视角来看，mark的效果是，不允许改变**要被删除结点的后继结点**。这点从避免前文提到的问题的角度来说，也应该是正确的理解。

`get_marked_reference`和`get_unmarked_reference`是以copy的形式传入reference，mark或者unmark以后的并不是reference本身。

## 数据结构

```cpp
struct Node {
  key_type key;
  node *next;
}

struct list {
  node *head;
  node *tail;
}
```

## search

满足这么几个要求：

* left_node.key < search_key <= right_node.key
* left_node and right_node are unmarked
* right_node is immediate successor（直接后继） of left_node

有这么几个步骤：

* 找到left_node和right_node
* 检查是不是直接后继
  * 是，直接返回
* 移除left_node and right_node之间的一个或多个marked结点

## insert

* 找到left_node和right_node
* cas插入

## delete

如前文解决方法中描述的，分两个阶段：

* stage 1:
  * 找到left_node和right_node
  * logically delete
* stage 2:
  * physically delete
    * cas删除
    * 或者search中删除

## find

* 找到left_node和right_node
* right_node == search_key

# Reference

1. A Pragmatic Implementation of Non-Blocking Linked-lists, Timothy L. Harris.
