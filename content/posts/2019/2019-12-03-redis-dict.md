---
title: redis dict
date: 2019-12-03 00:00:00
categories:
    - redis
tags:
    - redis
---

# 结构
dict使用链地址法解决冲突，`size`为$2^n$，`sizemask = size - 1`，用于计算key所属的bucket上，避免了mod，还便于处理scan时发生rehash的情况。每个dict有两个dictht，多个出来的一个用于rehash。

使用了类似list中的方式来保存key和value，并提供了`hashFunction`，`keyDup`等函数操作key和value。

```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

# 添加元素
添加元素的过程，
1. `dictAdd()`
2. `dictAddRaw()`

    * 如果需要rehash，`_dictRehashStep()`。
    * `_dictKeyIndex()`查找entry。
    * 如果找不到，创建新的entry，并插入链表的头部（这里假设了最近创建的会被更频繁的使用）。当rehash时，选择新的hash table，`d->ht[1]`进行操作。

    ```c
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    ```

3. `_dictKeyIndex()`

    * 检查是否需要扩张hash table（扩张或为`d->ht[0]`进行初始化），`_dictExpandIfNeeded()`。
    * 使用`hash & d->ht[table].sizemask`计算所属bucket。
    * 查找entry。在rehash的时候，会在两个hash table中进行查找。

    ```c
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    ```

# 扩张hash table
每次add都会调用`_dictExpandIfNeeded()`。

```c
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

* 如果`d->ht[0]`为空，那么`dictExpand()`中为`d->ht[0]`初始化，初始大小为4。

    ```c
    d->ht[0] = n;
    ```

* 如果load factor达到了1，还需要看dict当前是否可以resize，如果不可以，那么会提高load factor到5（有子进程时，为了避免COW内存复制），然后才会在`dictExpand()`中分配新的hash table。

    ```c
    d->ht[1] = n;
    d->rehashidx = 0;
    ```

`dictExpand()`中，如果分配了新的hash table，那么会set `d->rehashidx = 0;`，标识开始rehash。

```c
/* Expand or create the hash table */
int dictExpand(dict *d, unsigned long size)
{
    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```

# 删除key
删除分为两种，
* `dictDelete()`：会把key，value和entry都真正的进行删除。最终是调用`dictGenericDelete()`进行删除。
* `dictUnlink()`：只会把entry从链表中移除，并返回entry。最终是调用`dictGenericDelete()`进行删除。
    这个函数主要用于，想把entry从hash table中移除，但同时又想使用这个entry的情况。如果没有这个函数，则需要进行两次查找。

    ```c
    entry = dictFind(...);
    // Do something with entry
    dictDelete(dictionary,entry);
    ```

```c
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list */
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    zfree(he);
                }
                d->ht[table].used--;
                return he;
            }
            prevHe = he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return NULL; /* not found */
}
```

# rehash
为了避免集中的rehash操作对性能的影响，redis使用的是渐进式rehash。`dict.rehashidx`初始化为-1，在rehash时，`[0, rehashidnex)`代表老的hash table已经迁移的bucket。

rehash会两种情况下发生，
1. 每次查询或更新操作时，都会调用`dictRehash()`执行一步rehash，一般来说这个函数只会移动一个bucket，也有可能一个都不移动，因为rehash时，每次检查至多10个bucket。

    ```c
    if (dictIsRehashing(d)) _dictRehashStep(d);

    dictRehash(d,1);
    ```

2. 如果server比较空闲，上述rehash的过程就会很慢，dict将会占用两个hash table较长的时间。在初始化redis时，`serverCron()`会作为计时器的回调函数定时执行。

    如果启用了`server.activerehashing`，执行过程中，会分配1ms的时间来执行`dictRehash()`。

    * `serverCron()`
    * `databasesCron()`
    * `incrementallyRehash()`
    * `dictRehashMilliseconds(server.db[dbid].dict,1)`，1ms
    * `dictRehash(d,100)`

`int dictRehash(dict *d, int n)`中，最多移动100个bucket，在移动每个bucket时，至多会检查`n*10`个bucket，因为有的bucket可能是空的，因此需要多看几个。

```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

# scan
scan可以用于遍历dict中的元素，初始提供一个`cursor = 0`，每次迭代返回一个新的`cursor`，当`cursor`再次为0的时候，遍历结束。

遍历每个bucket时，调用`fn`对每个元素`de`进行处理，这个处理可能是，复制`de`到其他地方，例如：`void scanCallback(void *privdata, const dictEntry *de)`。

scan需要考虑是否正在rehash的情况，
1. scan过程中，dict没有变化

    由于hash table的大小总是$2^n$，因此每个bucket的index总是`key & 2^n - 1`后的一个值，即`idx = hash & d->ht[table].sizemask`。

    每次调用`scan`时，处理完毕一个bucket中的元素后，`v |= ~m0`保留cursor的低n个bit（在遍历结束前，`v == v | ~m0`），并加1。如果遍历结束，那么cursor值为`2^n - 1`，加1，并`v |= ~m0`后，`v`为0。则停止调用`dictScan`。

    这里还看不出`reverse cursor`的特殊作用。无论高位加1并`rev()`，还是低位加1，都可以完成上述过程。

    ```c
    t0 = &(d->ht[0]);
    m0 = t0->sizemask;

    /* Emit entries at cursor */
    if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
    de = t0->table[v & m0];
    while (de) {
        next = de->next;
        fn(privdata, de);
        de = next;
    }

    /* Set unmasked bits so incrementing the reversed cursor
     * operates on the masked bits */
    v |= ~m0;

    /* Increment the reverse cursor */
    v = rev(v);
    v++;
    v = rev(v);
    ```

2. 如果正在rehash
    如果正在rehash，scan是通过高位掩码的方式来完成扫描的。

    先来看bucket的计算方式，`idx = hash & d->ht[table].sizemask`。如果发生rehash，假设小表a大小为`2^n`，大表b为`2^m`。某个key，在两个表中的bucket分别为`hash & d->ht[a].sizemask`，`hash & d->ht[b].sizemask`。hash是一致的，这就意味着，两个bucket的低n位也是一致的。这是rehash时能保证元素一定会被扫描到的关键。

    scan时，t0（小表）索引为`v & m0`的bucket，然后扫描t1中索引低`n`为`v & m0`的bucket。由于大表中低`n`为`v & m0`的bucket是多个，因此需要在高`m - n`位不断递增。由于需要在高位不断递增，因此需要先`rev()`，然后再加1。

    总结来说，rehash时的扫描过程，其实就是通过一个函数，将一个bucket值`xxxxx`，映射到集合`aa...axxxxx`的过程。

    scan大表时的停止条件就是高`m - n`位都被扫描一遍，`v & (m0 ^ m1)`。

    ```c
    t0 = &d->ht[0];
    t1 = &d->ht[1];

    /* Make sure t0 is the smaller and t1 is the bigger table */
    if (t0->size > t1->size) {
        t0 = &d->ht[1];
        t1 = &d->ht[0];
    }

    m0 = t0->sizemask;
    m1 = t1->sizemask;

    /* Emit entries at cursor */
    if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
    de = t0->table[v & m0];
    while (de) {
        next = de->next;
        fn(privdata, de);
        de = next;
    }

    /* Iterate over indices in larger table that are the expansion
     * of the index pointed to by the cursor in the smaller table */
    do {
        /* Emit entries at cursor */
        if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
        de = t1->table[v & m1];
        while (de) {
            next = de->next;
            fn(privdata, de);
            de = next;
        }

        /* Increment the reverse cursor not covered by the smaller mask.*/
        v |= ~m1;
        v = rev(v);
        v++;
        v = rev(v);

        /* Continue while bits covered by mask difference is non-zero */
    } while (v & (m0 ^ m1));
    ```
