---
title: redis intset
date: 2019-12-05 00:00:00
categories:
    - redis
tags:
    - redis
---

# 结构
intset存储了有序的整数集合。

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

`encoding`决定如何解析`contents`，取值为，

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

# 字节序
不同于其他结构，intset在存储的时候考虑了字节序的问题，redis会使用小端序来存储intset的所有字段。目的是intset能够兼容不同字节序的cpu。

sds也有多字节的字段，为什么sds不做这个转换？

# 查找
由于`contents[]`是有序的，因此直接使用二分查找。

在执行二分查找前，先进行了特殊情况的判断，避免进行多余的搜索，
* 如果数组长度为0，则直接返回。
* 如果`value`大于末尾元素，或小于首部元素，也直接返回，并set插入位置`pos`。

二分查找中使用了`mid = ((unsigned int)min + (unsigned int)max) >> 1`来计算mid，会不会存在溢出的问题？毕竟intset的`length`的类型是`uint32_t`，理论可以保存$2^{32}$个元素。redis仅使用intset存储少量的元素，如果元素过多，会使用其他方式存储。

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

# 插入
插入分两种情况，
* 可直接插入
* 新元素的类型比集合存储的类型长，需要升级

## 直接插入
对于直接插入，先使用`intsetSearch()`找到待插入位置`pos`，然后移动`pos`后的所有元素。

```c
if (intsetSearch(is,value,&pos)) {
    if (success) *success = 0;
    return is;
}

is = intsetResize(is,intrev32ifbe(is->length)+1);
if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
```

移动元素分两个步骤完成，
* 在`intsetResize()`中调用`zrealloc()`先分配`is->length+1`的空间，这里可能会有一次内存拷贝。
* 然后在`intsetMoveTail()`使用`memmove()`移动内存，这里一定会有一次内存拷贝。`memmove()`保证了在原位置和目标位置的情况下，能够安全的进行拷贝。

  计算源地址和目的地址时，需要先将`contents`转换为内容实际对应的类型，然后再做指针的计算。

```c
static intset *intsetResize(intset *is, uint32_t len) {
    uint32_t size = len*intrev32ifbe(is->encoding);
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}

static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    uint32_t bytes = intrev32ifbe(is->length)-from;
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from;
        dst = (int64_t*)is->contents+to;
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    memmove(dst,src,bytes);
}
```

## 需要升级编码
对于插入类型大于集合现有类型的情况，那么待插入的元素一定是在数组头部（负数的时候）或者尾部。

```c
/* Upgrades the intset to a larger encoding and inserts the given integer. */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

判断新的编码类型并分配内存以后，从后往前遍历，逐个升级编码。这里需要处理prepend和append的情况。区别就是对于prepend，遍历时，新的元素的位置是当前位置+1，`_intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc))`。

```c
/* Set the value at pos, using the configured encoding. */
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```

## 降级编码
除了遍历，无法确定是否存在超出范围的元素，intset不支持降级。

# 删除
```c
/* Delete integer from intset */
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        is = intsetResize(is,len-1);
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```
