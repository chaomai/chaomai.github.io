---
title: redis list
date: 2019-12-02 00:00:00
categories:
    - redis
tags:
    - redis
---

# 结构
reids的list采用的是双向链表的实现，未使用dummy node。

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

# 特点
* 双端
* 无环
* 有头尾指针
* 有长度计数器
* 多态（使用`void*`来保存节点值，并提供`free`，`dup`，`match`来处理节点）
