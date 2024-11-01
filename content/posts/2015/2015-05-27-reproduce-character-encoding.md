title: "转 - 字符集和字符编码"
date: 2015-05-27 22:20:14
description: 原文是十分钟搞清字符集和字符编码，这里我简化了一些说明，稍微修改了原文的例子。
categories:
    - linux
tags:
    - 'character encoding'
---

<blockquote class="blockquote-center">

原文：[十分钟搞清字符集和字符编码](http://cenalulu.github.io/linux/character-encoding/)

</blockquote>

## 字符集

字符集就规定了某个文字对应的二进制数字存放方式（编码）和某串二进制数值代表了哪个文字（解码）的转换关系。

## 字符编码

字符集只是一个规则集合的名字，对应到真实生活中，字符集就是对某种语言的称呼。

对于一个字符集来说要正确编码转码一个字符需要三个关键元素：字库表（character repertoire）、编码字符集（coded character set）、字符编码（character encoding form）

### 字库表

一个相当于所有可读或者可显示字符的数据库，字库表决定了整个字符集能够展现表示的所有字符的范围。

### 编码字符集

用一个编码值**code point**来表示一个字符在字库中的位置。

### 字符编码

将编码字符集和实际存储数值之间的转换关系。

一般来说都会直接将code point的值作为编码后的值直接存储。例如在ASCII中`A`在表中排第65位，而编码后`A`的数值是`0100 0001`也即十进制的65的二进制转换结果。

* 为什么还要多此一举通过字符编码把序号转换成另外一种存储格式？

统一字库表的目的是为了能够涵盖世界上所有的字符，但实际使用过程中会发现真正用的上的字符相对整个字库表来说比例非常低。而如果把每个字符都用字库表中的序号来存储的话，每个字符就需要3个字节（这里以Unicode字库为例），这样对于原本用仅占一个字符的ASCII编码的英语地区国家显然是一个额外成本（存储体积是原来的三倍）。于是就出现了UTF-8这样的变长编码。在UTF-8编码中原本只需要一个字节的ASCII字符，仍然只占一个字节。而像中文及日语这样的复杂字符就需要2个到3个字节来存储。

## UTF-8和Unicode的关系

Unicode就是上文中提到的编码字符集，而UTF-8就是字符编码，即Unicode规则字库的一种实现形式。

Unicode标准几乎涵盖了各个国家语言可能出现的符号和文字，并将为他们编号。详见：Unicode on Wikipedia。Unicode的编号从`0000`开始一直到`10FFFF`共分为16个Plane，每个Plane中有65536个字符。而UTF-8则只实现了第一个Plane，可见UTF-8虽然是一个当今接受度最广的字符集编码，但是它并没有涵盖整个Unicode的字库，这也造成了它在某些场景下对于特殊字符的处理困难。

## UTF-8编码简介

### UTF-8的物理存储和Unicode序号的转换关系

UTF-8编码为变长编码。最小编码单位（`code unit`）为一个字节。一个字节的前1-3个bit为描述性部分，后面为实际序号部分。

* 如果一个字节的第一位为0，那么代表当前字符为单字节字符，占用一个字节的空间。0之后的所有部分（7个bit）代表在Unicode中的序号。

|Byte 1|
|-|
|0xxx xxxx|

|实际字符|在Unicode字库序号的十六进制|在Unicode字库序号的二进制|UTF-8编码后的二进制|UTF-8编码后的十六进制|
|-|-|-|-|-|
|$|0024|010 0100|0**010 0100**|24|

* 如果一个字节以110开头，那么代表当前字符为双字节字符，占用2个字节的空间。110之后的所有部分（5个bit）加上后一个字节的除10外的部分（6个bit）代表在Unicode中的序号。且第二个字节以10开头。

|Byte 1|Byte 2|
|-|-|
|0xxx xxxx|10xx xxxx|

|实际字符|在Unicode字库序号的十六进制|在Unicode字库序号的二进制|UTF-8编码后的二进制|UTF-8编码后的十六进制|
|-|-|-|-|-|
|¢|00A2|000 1010 0010|110**0 0010** 10**10 0010**|C2 A2|

* 如果一个字节以1110开头，那么代表当前字符为三字节字符，占用2个字节的空间。1110之后的所有部分（5个bit）加上后两个字节的除10外的部分（12个bit）代表在Unicode中的序号。且第二、第三个字节以10开头。

|Byte 1|Byte 2|Byte 3|
|-|-|-|
|0xxx xxxx|10xx xxxx|10xx xxxx|

|实际字符|在Unicode字库序号的十六进制|在Unicode字库序号的二进制|UTF-8编码后的二进制|UTF-8编码后的十六进制|
|-|-|-|-|-|
|€|20AC|0010 0000 1010 1100|1110 **0010** 10**00 0010** 10**10 1100**|E2 82 AC|

* 如果一个字节以10开头，那么代表当前字节为多字节字符的第二个字节。10之后的所有部分（6个bit）和之前的部分一同组成在Unicode中的序号。

* 3个字节的UTF-8十六进制编码一定是以`E`开头的。
* 2个字节的UTF-8十六进制编码一定是以`C`或`D`开头的。
* 1个字节的UTF-8十六进制编码一定是以比`8`小的数字开头的。

## 乱码

编码和解码时用了不同或者不兼容的字符集。

# Reference

1. [十分钟搞清字符集和字符编码](http://cenalulu.github.io/linux/character-encoding/)
2. [关于Python的默认字符集](http://cenalulu.github.io/python/python-encoding/)
3. [字符编码笔记：ASCII，Unicode和UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
4. [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html)
5. [字体编辑用中日韩汉字Unicode编码表](http://www.chi2ko.com/tool/CJK.htm)
