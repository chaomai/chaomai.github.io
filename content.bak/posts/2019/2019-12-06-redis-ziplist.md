---
title: redis ziplist
date: 2019-12-06 00:00:00
categories:
    - redis
tags:
    - redis
---

# ziplist
# 结构
ziplist是一个压缩的双向链表，由一个特殊编码的连续内存块构成。如果没有特殊指定，所有字段都是以小端序来存储的。

```c
ziplist：
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

entry：
<prevlen> <encoding> <entry-data>
```

ziplist：
* `uint32_t zlbytes`：ziplist占据的内存大小，包括`zlbytes`字段。
* `uint32_t zltail`：最后一个entry（非zlend）的offset，记录了尾节点距离起始地址的字节数，相当于尾指针的作用。
* `uint16_t zllen`：entry的数目，如果元素数目大于`2^16-2`，那么值为`2^16-1`，只有通过遍历才能得知，有多少个元素。zlen不计入entry数目。
* `uint8_t zlend`：特殊的entry，ziplist的结尾标记，值为`0xFF`。

每个entry大小是不固定的：
* `prevlen`：前一个entry的长度，用于从后往前遍历时计算prev entry的起始地址。长度小于254字节和大于等于254字节分别使用两种存储方式。

    | value  | prevlen | size |
    |--------|---------|------|
    |   < 254     |   `0xbb`     | 1字节 |
    |   >=254     |   `0xFE xx xx xx xx`   | 5字节 |

    当长度大于等于254时，`0xFE`代表`prevlen`值的类型，后面的4个字节才是长度。

    这样存储方式刚好可以与`zlend`的`0xFF`区分开来，没有一个entry的开头会是`0xFF`。

* `encoding`：记录了`entry-data`数据类型和长度。

    前两个bit代表存储的是字节数组，还是整型。

    | encoding  | size | 值的类型 |
    |-----------|------|---------|
    | `00pppppp` | 1 byte | 长度<=63的字节数组 |
    | `01pppppp|qqqqqqqq` | 2 bytes（14 bits使用大端序存储） | 长度<=16383的字节数组 |
    | `10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt` | 5 bytes（32 bits使用大端序存储） | 长度>=16384，且<=2^32 - 1的字节数组 |
    | `11000000` | 1 byte | int16_t |
    | `11010000` | 1 byte | int32_t |
    | `11100000` | 1 byte | int64_t |
    | `11110000` | 1 byte | 24 bit signed integer |
    | `11111110` | 1 byte | 8 bit signed integer |
    | `1111xxxx` | 1 byte | 没有`content`属性，4个bit存储[0, 12]的值 |
    | `11111111` | 1 byte | zlend |

* `entry-data`：节点的值，字节数组或整形。

# 插入
ziplist使用了很多宏来实现，主要是进行类型转换和指针运算以便访问相关的字段。

插入分三个情况，
![-w629](/images/2019/15757879857637.jpg)

需要分别准备新entry的`prevlen`，`encoding`，分配空间。

1. `prevlen`
    先看插入位置是否在`zlen`之前，
    * 如果不在，`p`处的`prevlen`就是新entry的`prevlen`。
    * 如果在，此时无法直接得知`prevlen`，需要看`zlend`前是否还有元素。若没有，则`zipRawEntryLength(ptail)`计算`tail`元素的长度。

    `p`不是`zlend`的时候，新entry的len会作为`p`处entry的`prevlen`，需要确保`p`处`prevlen`空间足够。

    ![-w736](/images/2019/15757965712995.jpg)

2. `encoding`
    对于`encoding`，先会尝试编码为整型`zipTryEncoding(s,slen,&value,&encoding)`，
    * 如果可以编码为整型，那么`zipIntSize(encoding)`得到entry-data的大小。
    * 如果无法合法的编码为整型，那么根据`slen`编码为字节数组`zipStoreEntryEncoding(NULL,encoding,slen)`，entry-data的大小为`slen`。

3. 分配空间，
    计算出各字段所需的空间后，`memmove()`来完成空间的调整。要注意对源地址的处理，`p-nextdiff`。
    * 如果`nextdiff == 0`，说明`p`处entry的`prevlen`，可以保存新entry的大小。
    * 如果`nextdiff == 4`，那么说明空间不够，此时`p`处entry的`prevlen`原大小为1 byte，从`p`往前数4 bytes，加起来的5 bytes作为新`prevlen`的存储，因此从`p - 4`处开始`memmove()`。

        ![-w614](/images/2019/15757965415426.jpg)

    * 如果`nextdiff == -4`，说明空间多出4 bytes，从`p + 4`的位置开始`memmove()`。

```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted.
     *
     * entry开头的1 byte是不是0xFF
     *
     * 如果插入位置是在zlend处，需要判断zlend前是否有entry，
     * 如果没有，那么新prevlen为0，
     * 如果有，那么需要计算zl+zltail处entry的大小，将其设置为新entry的prevlen
     *
     */
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }

    /* See if the entry can be encoded */
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    int forcelarge = 0;
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    /* Store offset because a realloc may change the address of zl. */
    offset = p-zl;
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry. */
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        zipEntry(p+reqlen, &tail);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```

## 连锁更新
插入或删除时，可能导致后面entry存储的`prevlen`发生变化。理论上，扩张和缩小都会有，但redis有意忽略了缩小的情况，避免连续的插入导致频繁的扩容和缩小。

```c
if (next.prevrawlensize > rawlensize) {
    /* This would result in shrinking, which we want to avoid.
     * So, set "rawlen" in the available bytes. */
    zipStorePrevEntryLengthLarge(p+rawlen,rawlen);
} else {
    zipStorePrevEntryLength(p+rawlen,rawlen);
}

/* Stop here, as the raw length of "next" has not changed. */
break;
```
