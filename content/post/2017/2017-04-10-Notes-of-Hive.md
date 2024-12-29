---
title: Hive笔记
date: 2017-04-10 17:39:19
categories:
    - hive
tags:
    - hive
---

# cluster by vs order by vs sort by

* ORDER BY x: guarantees global ordering, but does this by pushing all data through just one reducer. This is basically unacceptable for large datasets. You end up one sorted file as output.

* SORT BY x: orders data at each of N reducers, but each reducer can receive overlapping ranges of data. You end up with N or more sorted files with overlapping ranges.

* DISTRIBUTE BY x: ensures each of N reducers gets non-overlapping ranges of x, but doesn't sort the output of each reducer. You end up with N or unsorted files with non-overlapping ranges.

* CLUSTER BY x: ensures each of N reducers gets non-overlapping ranges, then sorts by those ranges at the reducers. This gives you global ordering, and is the same as doing (DISTRIBUTE BY x and SORT BY x). You end up with N or more sorted files with non-overlapping ranges.


# Hive NULL处理

默认情况下，每行数据在被输入script前，列会被转换为 `\t` 分隔的string，所有 `NULL` 值会被转换为字符串 `\N`，目的是为了与空string（长度为0）区分开。script输出一行数据时，标准输出会将输出是为 `\t` 分隔的列，同时 `\N` 被转换为 `NULL`。

因此Hive的空值处理分为：

* `NULL` 和 `\N`

    HIVE底层是用\N代替NULL进行保存的，可以通过`alter table name SET SERDEPROPERTIES('serialization.null.format' = '\N');` 来更改底层以什么样的字符来存储NULL。

    查询空值字段可用：

    ```
    a is null 或 a = '\\N'
    ```

    相应的在script在判断是否为 `NULL` 时，是需要判断`!= '\N'`。

* 空string

    `''` 表示长度为0的 string，但此时字段并非为 `NULL`，而是一个长度为0的 string。

    查询空string可用：

    ```
    a = '' 或 length(a)=0
    ```

    在使用outer join的时候尤其要注意以上区别，要想清楚究竟是要判断字段为 `NULL`，还是判断string为空。

    ```hive
    select case when t.idfa != '' and t.idfa != null then t.idfa end from (select 'B5BF3011-A8A0-46A2-B8C3-0A1976FDD4BE' as idfa) as t;
    select case when t.idfa != '' and t.idfa != NULL then t.idfa end from (select 'B5BF3011-A8A0-46A2-B8C3-0A1976FDD4BE' as idfa) as t;
    select case when t.idfamd5 != null then '1' else '2' end from (select '\N' as idfamd5) as t;
    select case when t.idfamd5 != null then '1' else '2' end from (select NULL as idfamd5) as t;
    ```


# Insert into bucketed table

Hive 2.x之前，向分桶表中插入数据时，需要（二者之一），

* 方法1：`set hive.enforce.bucketing = true`

* 方法2：
    1. 设置reducer数目，`set mapred.reduce.tasks`，使其等于bucket的数目
    2. 在select中使用，`cluster by`