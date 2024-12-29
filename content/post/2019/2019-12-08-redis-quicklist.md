---
title: redis quicklist
date: 2019-12-08 00:00:00
categories:
    - redis
tags:
    - redis
---

quicklist是由ziplist构成的双向链表。ziplist需要可能对内存进行复制，在长度较长的时候，性能不佳。quicklist存储多个小ziplist，对除`head`和`tail`外的节点还进行了压缩，保证了push和pop性能的同时，又减少了内存的占用。

# 结构
```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```

`quicklistLZF`：存储压缩后的ziplist

`quicklist`：
* `fill`：存放`list-max-ziplist-size`参数
    * `fill`取正值时，表示entry个数
    * 取负值时，表示大小
* `compress`：存放`list-compress-depth`参数

# push
push时，先检查本次push是否会超过限制大小，`quicklist->fill`，依次按照用户定义大小、默认个数（`SIZE_SAFETY_LIMIT`）、用户定义个数检查。
* 如果没有超过，那么在`head`的ziplist头部添加entry。
* 如果超过，那么创建新的ziplist，并在`quicklist->head`前插入新node。最后压缩原来的`quicklist->head`节点。

`quicklistPushTail`也是类似的，但是在ziplist的尾部添加entry，且插入后是压缩原来的`quicklist->tail`节点。

```c
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
```
