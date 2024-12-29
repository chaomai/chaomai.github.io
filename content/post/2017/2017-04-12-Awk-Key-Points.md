---
title: Awk关键点记录
date: 2017-04-12 13:54:34
categories:
    - linux
tags:
    - awk
---

# Variables

一个变量名就是一个合法的表达式。awk 的变量被默认初始化为空 string，当转换为数字时，是0，因此不需要显式初始化变量。

在为 awk 指定参数时，可以定义变量并赋值，`-v variable=text`，这样的变量在 `BEGIN` 之前就被定义，可以在 `BEGIN` 中使用。但 `-v` 必须先于文件名参数和程序文本。

如果在程序文本之后，`awk '{ print $n }' n=4 inventory-shipped n=2 mail-list`，那么变量n会在读取每条记录时被赋值，会先打印文件 `inventory-shipped` 第四列的内容，然后打印文件 `mail-list` 第二列中的内容。


## Predefined Variables

少数变量是有特殊含义的，它们是 predefined variables。这些变量，要么被 awk 自动检查（User-modified），要么被自动设置（Auto-set）。


### User-modified

* CONVFMT：控制数字到 string 转换。
* FS：列分隔符，值为一个字符或匹配多个字符的正则表达式。 默认是 `" "`，会匹配任何 `\" ", \t, \n`。同时会使得每条记录开头和结尾的 `" ", \t, \n` 被忽略。
* OFS：输出文件列分隔符。
* ORS：输出文件记录分隔符（行尾）。
* RS：输入文件记录分隔符（行尾）。
* SUBSEP：下标分隔符，默认是 `\034`。


### Auto-set

* ARGC, ARGV：awk 参数信息，ARGV 的起始下标为0。程序文本不会被包含到这两个变量中。
* ENVIRON：关联数组，包含了 environment 的值。
* FILENAME：当前输入文件的文件名。 未设置输入文件（stdin输入）时，值为 `"-"`；在BEGIN中时，值为 `""`。
* FNR：当前文件的行号，在每次打开新文件时被重置为0。
* NF：当前记录的列数。在每读取一个新纪录的时候，或者 `$0` 改变的时候，被重置。
* NR：已处理的行数。
* RLENGTH：被 `match()` 匹配后，substring 的长度，没有匹配值为-1。
* RSTART：被 `match()` 匹配后，substring 的起始位置，没有匹配值为0。


# Pattern Elements

## Regexp Patterns

awk 中正则表达式的形式为`/regexp/`。


## Expression Patterns

awk 表达式都可作为一个 pattern，如果表达式的值非空或非零，那么就是匹配上了。常见的是由比较运算符构成的，

* `awk '$1 == "li" { print $2 }' mail-list`
	用于直接比较 string 的操作符除了 `==`，还可以是`<, <=, >, >=, ==, !=`。以及判断array中是否有以subscript为下表的元素，subscript in array。

* `awk '$1 ~ /li/ { print $2 }' mail-list`
	用于判断特定的 string 是否 match 正则表达式，需要用 `~, !~` 连接左右操作数。

* `awk '/edu/ && /li/' mail-list`
	也可以用布尔操作符连接正则表达式、由比较运算符构成的表达式，以及任何其他的 awk 表达式。


## Ranges

range pattern 由逗号分隔的两 个pattern 构成，`begpat, endpat`，用于匹配连续的记录。 当遇到匹配 `begpat` 的记录时，开始匹配包括这条记录在内的所有后续记录，直到遇到匹配 `endpat` 的记录时，停止匹配，然后开始下一轮。如果有某条记录使得两个 pattern 都为 true，那么只匹配一次，并开始下一轮。

```awk
file:
awefawef
aweawef%
awefawfe
aweawef%

awk '/%$/,/%$/ {print}' test
aweawef%
```

## `BEGIN/END`

前面的几个 pattern 都是会匹配输入的，`BEGIN/END` 是特殊的 pattern，不会匹配（因为在它们执行的时候还没有记录，或者已经没有记录存在了），它们指定了 awk 程序起始和结束时的 action，都只会执行一次。

如果存在多个 `BEGIN/END`，那么按定义的顺序执行。 当仅有 `BEGIN`，执行完 `BEGIN` 后，程序结束；当仅有 `END`，程序会读取输入，然后执行 `END`。


# 常用awk程序

下面是我常用到的，或者看到觉得不错的awk程序。


## 以某个字符串为key，统计对应的value总和

```awk
 awk '{a[$4"\t"$5"\t"$2]+=$3} END {for (x in a) print x"\t"a[x]}' $geo_file_ret | sort -k 4 -n -r > $1
```


## 打印非空行

```awk
awk 'NF'
```

因为 `NF` 是当前输入的列数，非空行的列数大于0。当条件为真，且没有其他的 action 时，awk执行默认操作，打印一行。扩展来说，可以只提供表达式，作为条件，表达式为真则打印响应行。

类似的，

```awk
awk '!a[$0]++'
```

`a` 是数组，每个元素被初始化为0，`$0` 作为索引。首次遇到的行，`++` 对以 `$0` 为下标的元素加1，后缀 `++` 返回原始值0，取反后为真，打印。后续如果再遇到相同的，可以取到一个非0的值，取反为false，不打印。因此实际上是去除了重复的行。

以及，

```awk
seq 1 30 | awk 'ORS=NR%5?FS:RS'
```

默认情况下，ORS是一个换行符 `"\n\"`。这里根据，已处理行数是否是5的倍数，为ORS赋值FS或RS，默认即，`" " or "\n"`。它们的值都不为0，作为条件都是true。因此每一行都会打印，只是被打印行的行为分隔符是FS或RS。

```
1 2 3 4 5
6 7 8 9 10
11 12 13 14 15
16 17 18 19 20
21 22 23 24 25
26 27 28 29 30
```


## 替换分隔符

在没有修改的情况下，awk 并不会重新 build `$0`，原因之一是，为了性能；二是，对于 `echo 'foo;bar' | awk -v FS=';' -v OFS=',' '/foo/'`，如果真的替换了，那 `foo,bar` 并不是期望匹配到的 `foo;bar`。 因此，不同于上面的 ORS，这里直接指定 FS 以及 OFS 是没有用的。

```awk
awk -v FS=';' -v OFS=',' '{$1=$1}1'
```

这里的确修改了 `$0`，但内容没有变，因此会替换FS。如果没有空行，可以把1去除。


## 连接字符串

利用 awk 中变量被初始化为空 string 或0这一点，可以方便的连接字符串

```awk
string = string sep somedata; sep = ";"
```

避免开头出现多余的 `";"`。


## 处理两个文件

当处理两个文件时，awk 按命令中指定的顺序，顺序读取每个文件。当读取第一个文件时，`NR == FNR`总为true。 基于这点，有一个常用的写法，

```awk
awk 'NR == FNR { # some actions; next} # other condition {# other actions}' file1.txt file2.txt
```

常用的场景比如，

* 两文件的交集，`awk 'NR == FNR{a[$0];next} $0 in a' file1.txt file2.txt`
* 两文件的差集（仅在第一个文件里出现），`awk 'NR == FNR{a[$0];next} $0 in a' file1.txt file2.txt`
* 基于某一列，join两文件，`awk 'NR == FNR{a[$1]=$2;next} {$3=a[$3]}1' map.txt data.txt`


不过这个写法，仅适用于第一个文件非空的情况，如果为空，会直接用 `NR == FNR { # some actions; next}` 处理第二个文件，可以检测FILENAME是否等于ARGV[1]。


# More

* http://coolshell.cn/articles/9070.html
* http://www.gnu.org/software/gawk/manual/gawk.html