---
title: Go笔记
date: 2019-03-17 14:22:40
categories:
    - golang
tags:
    - golang
---

# 文档和资源
* [语言规范](https://golang.org/ref/spec)
* [go命令行文档](https://golang.org/doc/cmd)
* [包列表](https://golang.org/pkg/)
* [go wiki](https://github.com/golang/go/wiki)
* [The Go Blog](https://blog.golang.org/index)

# 要注意的点
* [关于引用类型](#其他类型)
* [Slice的内部存储](#Slice)
* [Slice的for range](#Slice)
* [method sets和calls](#方法和接口)

# Packages
每个go程序都由package构成。

## Exported name
在一个包中，如果一个name是以大写字母开头的，那么这个name将会被从这个包中导出。

当import一个包的时候，只能引用**被导出**的name。

# 函数
## 参数
参数名称在前，类型在后，[Go's Declaration Syntax](https://blog.golang.org/gos-declaration-syntax)。

当多个连续的函数参数有共同的类型是，可以省略不写除最后一个外的类型。

```go
func add(s string, x, y int) int {
	return x + y
}
```

## 返回值
一个函数可以返回任意个数的结果。

返回值可以是有名字的。当`return`不带任何参数时，函数返回named return values，这叫做"naked" return。

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
```

## 变量
`var`声明多个变量时，只能写最后一个的类型。

```go
var c, python bool

// error
// var c int, python bool
```

如果声明时给出了初始值，那么会以初始值的类型作为变量的类型，此时声明中的类型可省略。

在函数内部可以用短变量声明`:=`来声明变量，类型由值的类型来决定。但在函数外部，由于所有语句都需要以关键字开头，因此不可用这个方法。

**初始化**
在声明变量的时候，变量的值总是会被初始化，要么是用指定的值，要么是零值（变量类型的默认值）。

## 基本数据类型
```go
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128
```

声明变量但不显式给出初始值，变量会被赋予零值。数值类型：`0`，bool类型：`false`，string：`""`。

## 类型转换和推断
在进行类型转换时，go只能使用**显式**类型转换。

使用`:=`或`var =`声明变量、未指明类型、但给出初始值时，变量的类型由对初始值进行类型推断得到。如果右侧是数值常量，那么变量的类型可能是`int`，`float64`，`complex128`。

## 常量
只可以用`const`来声明。数值常量可表示任意精度，且不会溢出。一个未指定类型的常量由上下文来决定其类型。

## 全局变量
在程序运行期间，始终存在。声明和初始化方式与普通变量相同，需要在函数外部声明。

# 语句
## for
```go
// 对比c语言，括号可选、大括号必须
// 不可以使用var的方式声明
for i := 0; i < 10; i++ { ... }

// 初始化语句和循环的每次更新可省略
for ; sum < 1000; { ... }

// 可当做while使用
for sum < 1000 { ... }

// 无限循环
for { ... }
```

## if
```go
// 对比c语言，括号可选、大括号必须
if x < 0 { ... }

// 可以使用短语句，在判断之前执行，语句中声明的变量仅在if和后续的else语句块中可用
if v := math.Pow(x, n); v < lim { ... }

// else不能换行写
if v := math.Pow(x, n); v < lim {
    ...
} else {
    ...
}
```

## switch
求值顺序，按case的顺序，自上向下进行。

```go
// 对比c语言，
// break可选，不写时自动提供
// case不必是integer

switch os := runtime.GOOS; os {
 case "darwin":
     fmt.Println("OS X.")
 case ...
     ...

 // 不带条件的switch相当于switch true
 switch {
 case t.Hour() < 12:
     fmt.Println("Good morning!")
case ...
    ...
```

## defer
使用`defer`时，被`defer`的函数会被push到一个stack，参数会**立即计算**，但函数结束时，stack中的函数才会被pop出来执行。

```go
func t(s string) string {
    fmt.Printf("in func t: %s\n", s)
    return s
}

func main() {
    {
        defer fmt.Println(t("you"))
        fmt.Println("hello")
    }
}

// output:
// in func t: you
// hello
// you
```

`defer`的函数可以读取和赋值到函数的返回值。

```go
func c() (i int) {
    defer func() { i++ }()
    return 1
}

// c返回2

func c1() (int) {
    var i int
    defer func() { i++ }()
    return 1
}
// c1返回1
```

**defer、panic和recover**
当调用`panic`时，所有`defer`的函数都被正常执行。然后函数返回到调用者。

`recover`仅在`defer`的函数中有用，正常执行时调用，只会返回`nil`。

```go
func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    // g发生panic后，这个deferred的函数会执行，并捕获panic
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```

# 其他类型
关于slice，map和channel，某些书中会将它们描述为引用，但从实现上看（例如：[slice](https://golang.org/src/runtime/slice.go)、[map](https://golang.org/src/runtime/map.go)、[chan](https://golang.org/src/runtime/chan.go)），这些类型不过只是封装了底层指针的struct，且go spec也早就在文档中[移除](https://github.com/golang/go/commit/b34f0551387fcf043d65cd7d96a0214956578f94)了reference一词的使用，而在THE WAY TO GO一书中虽然使用了reference一词，但也明确指出，

> A reference type variable r1 contains the address (a number) of the memory location where the value of r1 is stored.
> ...
> When assigning r2 = r1, only the reference (the address) is copied.
> ...
> In Go pointers (see § 4.9) are reference types, as well as slices (ch 7), maps (ch 8) and channels (ch 13). ......

## 指针
存储了内存地址，零值为`nil`。`&`获得变量的地址，`*`解引用。

对比c的指针，go的指针无法进行算数运算。

```go
var p *int

i := 42
p = &i
```

**指针的类型转换**
`unsafe.Pointer`：`type Pointer int`，代表了变量的内存地址，可以将任意变量的内存地址与`Pointer`指针相互转换。
`uintptr`：`type uintptr int`，`Pointer`无法进行加减运算，需要转换为`uintptr`才可以，可以将`Pointer`与`uintptr`指针相互转换。
`unsafe.Offsetof`：可以得到字段在结构体内的偏移量。

```go
*T <=> unsafe.Pointer <=> uintptr

type Vertex struct {
	X int
	Y int
}

var v = Vertex {50, 50}
// *Vertex => Pointer => *int => int
var x = *(*int)(unsafe.Pointer(&v))
// *Vertex => Pointer => uintptr => Pointer => *int => int
var y = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&v)) + uintptr(8)))
var y = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&v)) + unsafe.Offsetof(v.Y))
```

## Struct
字段的集合，使用`.`来访问字段。首字母大写和小写分别代表公开和私有。私有变量只有同一个package才可以访问。

对于struct指针，可以使用`(*p).X`或直接使用`p.X`来进行访问。对比c，go不能用`->`来访问成员。

**初始化**
```go
type Vertex struct {
	X int
	Y int
}

// 使用多行的形式指定一个或多个字段的值的时候，最后的逗号不可以省略。如果只是一行，最后的逗号的可选
// 如果没有指定字段值，那么会使用相应类型默认的零值进初始化。
var v1 = Vertex {
    X: 1,
    Y: 2,
}
var v2 = Vertex {
    X: 1,
}
var v3 = Vertex { X: 1 }

// 零值结构体实际分配了结构体的内存空间
var v4 = Vertex {}
var v5 = Vertex { 1, 2 }
var v6 *Vertex = &Vertex { 1,2 }
var v7 *Vertex = new(Vertex)
var v8 Vertex

// nil结构体不会分配结构体内存
var v9 *Vertex = nil
```

**copy**
* 结构体之间的copy是深拷贝，不共享结构体内部字段。
* 结构体指针的copy是浅拷贝，共享内部字段。

**组合**
```go
type Vertex struct {
	X int
	Y int
}

type Circle1 struct {
    v Vertex
    Radius int
}

// 匿名组合，此时外部的结构体在使用时，可以直接使用内部结构体的成员和方法。如果内部和外部存在相同名字的方法，会调用外部结构体的方法。
type Circle2 struct {
    Vertex
    Radius int
}
```

## Array
和c一样，数组的大小也是数组类型的一部分，声明数组时必须有大小，通过下标访问数组中的元素。

程序执行时，go会检查访问是否越界。

```go
var a[10] int

var a = [6] int{2, 3, 5, 7, 11, 13}
var b[6] int = [6] int{2, 3, 5, 7, 11, 13}
c := [6] int{2, 3, 5, 7, 11, 13}
d := [...] int{2, 3, 5} // 自动推断长度
e := [5] int{1: 10, 2: 20} // len(e) = 5, e[1] = 10, e[2] = 20

// 数组字面值
primes := [6]int{2, 3, 5, 7, 11, 13}

// 同类型（长度和元素类型）的数组可以相互赋值，会copy数组的内容
var e[6] int
e = a
```

**内部存储**
连续分配的内存区域。

**copy**
数组的类型由元素的类型和数组的大小决定，相同类型的数组之间才可以copy。拷贝一个数组，数组的内部的元素也会被逐一拷贝，因此作为变量传递时，需要注意copy的开销。

## Slice
slice的类型为`[]int`，对数组进行，
* `a[low_index:high_index]`后得到，区间是前闭后开，可以省略`low_index`或`high_index`，默认值分别为`0`和数组长度。
    `cap(a)` = `len(array) - low_index`
* `a[low_index:high_index:cap_index]`后得到，区间是“闭、开、开”。`cap_index`代表可用到的底层数组的最大index，必须小于`len(array)`。
    `cap(a)` = `cap_index - low_index`

```go
var a[10]
a[:4]
a[:]
b := []string {99: ""} // len(b) = 100, cap(b) = 100
```

**内部存储**
```
       +---------------+
slice: |pointer|len|cap|
       +--+------------+
          |
          |
       +--v--------------+
array: |item1|item2|...  |
       +-----------------+
```

例如：
```go
s := []int{2, 3, 5, 7, 11, 13}

var s = []int{1,2,3,4,5,6,7,8,9,10}
var address = (**[10]int)(unsafe.Pointer(&s)) // 底层数组的地址
var len = (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(8)))
var cap = (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(16)))
```

**slice和array**
slice本身并不存储任何数据，仅仅是数组选定区间的描述，和数组共享底层的数据。`len()`和`cap()`对应了slice的长度，和底层数组从`low`起的大小，即：`len(array) - low`。

对已有slice再做一次slice，实际上是改变slice对底层数组的引用范围。

```go
s := []int{2, 3, 5, 7, 11, 13} // len(s)=6, cap(s)=6

// Slice the slice to give it zero length.
s = s[:0]

// Extend its length.
s = s[:4]
```

slice字面值类似数组的，区别是没有大小。底层实际上创建了相同大小的数组，然后再创建slice。

**nil slice**
一个nil slice，是未初始化的slice，`len`和`cap`都为0，且不会分配底层的数组，数组指针为`nil`。

```go
var a []int
```

**空slice**
一个空slice的`len`和`cap`都为0，且不会分配底层的数组，数组指针值不为空，但是也未分配底层数组。

```go
a := make([]int, 0)
b := []int{}
```

当想声明一个空的slice时，nil slice和空slice都可以，两者在功能上完全等价，但是更[推荐](https://github.com/golang/go/wiki/CodeReviewComments#declaring-empty-slices)nil slice。但二者进行序列化的时候，结果会不同，nil slice会编码为null，而空slice是`[]`。

**make slice**
通过`make`来创建动态长度的数组。

```go
a := make([]int, 5)  // len(a)=5, cap(a)=5
b := make([]int, 0, 5) // len(b)=0, cap(b)=5
var c []int = make([]int, 5)
var d = make([]int, 5)
```

**append**
`append`函数能够将相同类型元素追加至现有slice，若底层数组大小不够，则会重新分配内存，并将slice指向新数组。

如果发生了扩容，且有另一个slice存在，那么另一个slice的仍然指向老的数组。

扩容时，如果`cap < 1024`，那么会扩100%，否则扩25%。

```go
func append(s []T, vs ...T) []T
```

**range**
除了普通方法遍历slice，还能使用`range`，

```go
// i是index，v是相应元素的copy。
for i, v := range s { ... }

// 可忽略i或v的赋值
for _, value := range s { ... }

for i, _ := range s { ... }
// 或
for i := range s { ... }
```

当使用`range`返回的值，`v`时，**要注意的是**`range`返回的是**元素的copy**，而不是引用，如果对齐进行`&`，那么得不到期望的结果。具体来说，Go会使用**同一个**变量，在每轮迭代中保存元素的copy。可以使用[kyoh86/scopelint](https://github.com/kyoh86/scopelint)来检查代码中的unpinned variables。

1. 取地址。

    ```go
    a := make([]int, 5)

    for i, v := range a {
        fmt.Println(&v, &a[i]) // v的地址保持不变，且不等于a中任意元素的地址
    }
    ```

2. 这个原因还有可能导致使用goroutine时出现[意外](https://github.com/golang/go/wiki/CommonMistakes#using-goroutines-on-loop-iterator-variables)。

    ```go
    // 由于closure已经绑定到了val，又因为goroutine可能在for结束后才执行
    // 因此打印出的可能全都是values的最后一个值。
    for _, val := range values {
    	go func() {
    		fmt.Println(val)
    	}()
    }

    // ok
    for _, val := range values {
    	go func(val interface{}) {
    		fmt.Println(val)
    	}(val)
    }

    // ok
    for i := range valslice {
    	val := valslice[i] // val在每次迭代中都会分配新的
    	go func() {
    		fmt.Println(val)
    	}()
    }
    ```

**copy**
slice的copy是浅拷贝，两个slice共享底层数组。本质上copy的是：指向底层数组的指针、`len`和`cap`。

```go
a := []int{2, 3, 5, 7, 11, 13} // len(s)=6, cap(s)=6
var b = a
```

go还提供了一个函数`copy`来实现数组内容的copy，copy时，会以目的切片的容量为准。

```go
func copy(dst, src []T) int
```

结合slice的内部存储、`append`和拷贝，有的使用场景不注意可能导致意料之外的结果。

```go
func FuncSlice(s []int, t int) {
	s[0]++
	s = append(s, t)
	s[0]++
}
func main() {
	a := []int{0, 1, 2, 3}
	FuncSlice(a, 4)
	fmt.Println(a)
}

// [1 1 2 3]
```

具体过程分析如下：
1. `s[0]++`：`a`和`s`都指向同一个底层数组`arr1`，此时`a->arr1`，`s->arr1`，修改了`arr1`。
2. `append`：由于扩容，`append`返回了一个新的底层数组`arr2`，`a->arr1`，`s->arr2`。
3. `s[0]++`：修改了`arr2`，`arr1`不变。

## Map
一个仅做了声明的map是`nil`，需要使用`make`来进行初始化。切片、函数以及包含切片的结构类型这些类型由于具有引用语义，不能作为映射的key，使用这些类型会造成编译错误。

```go
type Vertex struct {
    Lat, Long float64
}

var m map[string]Vertex

fmt.Println(m == nil) // true
m = make(map[string]Vertex)
m["Bell Labs"] = Vertex{
    40.68433, -74.39967,
}
```

map字面值和struct字面值类似。

```go
var m = map[string]Vertex{
    "Bell Labs": Vertex{ 40.68433, -74.39967,}
}

// 当最上层的类型只是类型名的时候，可以省略。
var m = map[string]Vertex{
    "Bell Labs": { 40.68433, -74.39967,}, //最后的逗号不可缺少
}
```

**mil map**
nil map未进行初始化，不能用于存储key-value。

```go
var m map[string]Vertex
```

**make map**
通过`make`来创建map。

```go
var m = make(map[string]Vertex, 10)
```

**range**
类似slice，且slice中存在的问题，map中也同样存在。由于无法获取index，因此只能通过每轮迭代创建变量来解决。

```go
// k是key，v是相应元素的copy。
for k, v := range m { ... }

// 可忽略k或v的赋值
for _, v := range s { ... }

for k, _ := range s { ... }
// 或
for i := range s { ... }
```

**操作**
* 新增和更新：`m[key] = elem`
* 访问key的值：`elem = m[key]`，如果不存在，那么`elem`为此类型的零值，但如果值真的是零值，那通过这个方法来判断key是否存在就失效了。
* 删除：`delete(m, key)`
* test：`var elem, ok = m[key]`，如果不存在，那么`elem`为此类型的零值。

## 字符和字符串
### rune
rune字面值代表一个rune常量，是一个标识Unicode code point的整型值。`alias for int32`。

rune字面值可以用`'单个字符'`来表示，可以用`\`转义的多个字符来表示，具体见[Rune literals](https://golang.org/ref/spec#Rune_literals)。

### string
string字面值代表了包含一系列字符的string常量，只读。有两种形式：raw string字面值和interpreted string字面值。

* raw string字面值，是经过未转义处理的，在raw string内部，可以出现任意字符。string中出现的`'\r'`会被忽略。
* interpreted string字面值，go会进行转义处理。具体见[String literals](https://golang.org/ref/spec#String_literals)

```
fmt.Println("\U000065e5\U0000672c\U00008a9e")    \\ 日本語

fmt.Println(`\U000065e5\U0000672c\U00008a9e`)    \\ \U000065e5\U0000672c\U00008a9e
```

**内部存储**
```
            +---------------+
byte slice: |pointer|len|cap|
            +--+------------+
               |
               |
            +--v------------+
array:      |item1|item2|...|
            +---^-----------+
                |
            +---+-----------+
string:     |pointer|len|cap|
            +---------------+
```

```go
a := "Hello，你好"
fmt.Printf("%x\n", *(*[2]int)(unsafe.Pointer(&a)))    \\ [4b9f3c e]

b := a
fmt.Printf("%x\n", *(*[2]int)(unsafe.Pointer(&b)))    \\ [4b9f3c e]
```

由于底层存储是数组，因此可以做slice，但要注意的是，这里本质上是对字节来做slice，因此如果slice的Unicode code point不是一个完整的字符，那么打印的时候，是不会正确显示的。

```go
a := "Hello，你好"
b := a[0:9]
fmt.Println(b)    \\ Hello，�
```

从字符串得到字节slice或者从字节slice得到字符串，会发生底层数组的copy。如果想避免copy，可以手动一个string或slice，获得一个原始string或者slice的“reference”，这种方式不可以通过slice修改string，因为修改后，“reference”到的原有string失效了，可能会被gc回收。

```go
a := "Hello，World"
b := []byte(a) // copy
c := string(b) // copy

// 手动构造
func str2bytes(s string) []byte {
    var strhead = *(*[2]int)(unsafe.Pointer(&s))
    var slicehead [3]int
    slicehead[0] = strhead[0]
    slicehead[1] = strhead[1]
    slicehead[2] = strhead[1]
    return *(*[]byte)(unsafe.Pointer(&slicehead))
}

func bytes2str(bs []byte) string {
    return *(*string)(unsafe.Pointer(&bs))
}
```

**遍历**
* 按字节遍历：通过下标。
* 按字符遍历：range方式遍历。

## Function values
函数也是值，可作为参数传递，作为返回值返回。

闭包（closure）是function value引用了函数体外部的变量，函数可以访问和修改这些变量。换句话说，闭包包含了**函数、以及所在的环境的上下文**。

# 方法和接口
## 方法集（method sets）和调用
[方法集](https://golang.org/ref/spec#Method_sets)和[函数调用](https://golang.org/ref/spec#Calls)的规范明确了一个类型**有哪些方法**，以及**在什么时候可以调用**什么样的方法。go wiki上关于这两个概念有比较详细的[例子](https://github.com/golang/go/wiki/MethodSets)。

**method set**
1. 对于一个接口类型，接口是方法集。

2. 对于一个类型`T`，所有receiver为T的方法是方法集。

    对于类型T对应的指针类型`*T`，所有receiver为`T`或`*T`的方法是方法集。

| Type | method sets |
| --- | --- |
| interface type | interface |
| T | func (T) f() |
| *T | func (*T) f(), func (T) f() |

一个类型`T`的方法集决定了，这类型`T`的接口类型的实现，和使用`T`作为receiver时可以被调用的方法。

**call**
对于一个方法调用`x.m()`，
1. 如果`x`的method set包含`m()`，且调用时的参数列表合法，那么这个调用是合法的。

2. 如果`x`可以[取地址的](https://golang.org/ref/spec#Address_operators)，并且`&x`的method set包含`m()`，那么`x.m()`等价于`(&x).m()`。

    *map元素和interface存储的具体值不可取地址。*

## 方法
方法是带有特殊receiver参数（`func`和函数名之间）的函数。这个receiver不必是struct，但要求receiver的类型定义必须在同一个package里面，且不能直接将内置类型作为receiver。

```go
type Vertex struct { X, Y float64 }
func (v Vertex) Abs() float64 { ... }

type MyFloat float64
func (f MyFloat) Abs() float64 { ... }
```

**Point receiver**
若要修改字段，则必须使用point receiver，**无论变量本身是否是指针类型**，**非指针receiver调用时发生了copy**。

从这里可以得出使用point receiver的场景：1. 避免copy；2. 修改值本身。一般来说，某个类型的receiver应该**统一**，要么是point receive，要么是普通receiver。

```go
func (v *Vertex) Scale(f float64) { v.X = v.X * f ... }
func (v Vertex) Scale2(f float64) { v.X = v.X * f ... }
```

* 若v不是指针类型，那么go会把`v.Scale`自动转换为`（&v）.Scale`。
* 反过来，若v是指针类型，在调用`Scale2`时，go会把`v.Scale2`转换为`（*v).Scale2`。

对比c++的成员函数，`this`指针类似于point receiver，但go的普通receiver是不同于c++的。

## 接口
接口是方法签名的集合。

接口的实现是隐式的，无需类似`implement`的关键字。隐式实现**解耦**接口的定义和实现，在package中，接口的定义可以出现在方法和类型定义之后。注意方法的实现**区分普通receiver和point receiver**。

一个接口值可以被赋值为**任何实现了接口中所有方法的值**。接口值底层实际包含了具体值的类型，接口值可以看做是值和具体类型的元组。调用接口值的方法，实际上会调用具体类型的方法。

```go
type Abser interface { Abs() float64 }

type MyFloat float64
func (f MyFloat) Abs() float64 { ... }

type MyInt int64
func (f MyInt) Abs() int64 { ... }

type Vertex struct { X, Y float64 }
func (v *Vertex) Abs() float64 { ... }

var a Abser
f := MyFloat(1)
i := MyInt(2)
v := Vertex{3, 4}

a = f
a = i
a = &v

// a = v
// error，Vertex没有实现Abs()，*Vertex才实现了Abs()
```

对比c++的多态，c++中通过继承基类，并覆盖基类的虚函数，在运行时进行动态绑定，以此实现多态。go的接口方法定义可以看做是基类和虚函数，而`a = f`相当于将子类的指针赋值给基类指针，这样完成了动态绑定。

不同的点还是receiver，实现接口的方法时，go区分了point receiver和普通receiver。

**内部存储**
实现上，一个接口值底层包含了指向类型和数据的指针。

接口类型之间的赋值和类型转换是共享数据的，而结构体之间的赋值、结构体转接口、接口转结构体，都会导致数据的copy。

**空的具体类型值**
如果接口的具体类型值是空的，那么将会使用`nil` receiver来调用方法，不引发空指针异常。

```go
type Abser interface { Abs() float64 }

type Vertex struct { X, Y float64 }
func (v *Vertex) Abs() float64 {
    if v == nil {
        ...
    }
    ...
}

var i Abser
var v *Vertex
i = v

i.Abs()
// or
v.Abs()
```

**空的接口值**
会发生运行时错误，没有具体的`Abs`方法可以调用。

```go
type Abser interface { Abs() float64 }
var i Abser
i.Abs()
```

**空接口**
空接口的值可以包含任何类型。

```
var i interface {}
i = 32
i = "hello"

// i = 100000000000000000000000000
// overflows int
```

**接口变量的赋值**
对于数值类型，底层的具体类型**只能是**`int`，`float64`，`complex128`。

**类型断言**
类型断言提供了访问接口底层具体类型值的能力。

* `t := i.(T)`断言接口`i`拥有具体类型`T`，并把类型`T`的值赋值给`t`。如果不是类型`T`，则触发panic。
* test：`t, ok := i.(T)`断言不正确的情况，不触发panic，而是`ok`为false，且t为类型`T`的零值。

```go
var i interface{} = 100000000000000000000000000 // overflow
a := i.(int64)

var i interface{} = 100000
a := i.(int64) // int
```

Type swtiches是允许断言多个类型的结构。类似switch语句，但是每个case是特定的类型。

```go
// 如果test成功，那么v会转换为相应的类型。

switch v := i.(type) {
case T:
    // here v has type T
case S:
    // here v has type S
default:
    // no match; here v has the same type as i
}
```

对比scala的pattern matching，go的type swtiches像，但不是pattern matching。scala的pattern matching会检查值和pattern是否匹配，能够把值解构为构成值的各部分。**猜测**go的type swtiches是类型字符串是否相等的test。

## 一些内置的接口
**Stringer**
类似python的`__str__`，定义在`fmt`中。

```go
type Stringer interface {
    String() string
}

type A ...
func (a A) String() string { return "hello" }

a := A()
fmt.Println(a) // hello
```

**error**
类似Stringer，fmt在print的时候也会查找`error`接口。从fmt的实现上看，是error**优先**。

```go
type error interface {
    Error() string
}
```

`error`更适合用于专门定义的错误类型。否则功能上，`stringer`和`error`就冗余了。

可以使用`fmt.Errorf`或`errors.New`来创建`error`类型的值。

```go
fmt.Errorf("math: square root of negative number %g", f)
```

**Reader**
io包定义了`io.Reader`接口，代表读取stream，有多个实现（文件、网络等）。

其中`func (T) Read(b []byte) (n int, err error)`方法使用现有数据填充`b`，并返回填充的字节数和`error`。stream结束时，`error`为`io.EOF`。

# 并发
## Goroutines
由go运行时管理的轻量级线程。

```go
go f(x, y, z)
```

参数的计算在当前goroutine中完成，函数`f`的调用发生在新的goroutine。所有子协程都是平级的关系（包括在子协程内部启动另一个协程）。

## Channels
channel是带类型的管道（typed conduit），每次只能发送或接受一个元素。默认情况下，发送方和接收方会一直阻塞到另一方ready，每次只能唤醒一个发送或接受方。

**方向**
```go
chan T // 可以发送和接受类型为T的数据
chan<- T // 可以发送类型为T的数据
<-chan T // 可以接受类型为T的数据
```

**unbuffered channel**
unbuffered channel必须保证先有goroutine正在接收，否则发送方会一直阻塞到有goroutine来接收为止。

```go
ch := make(chan int) // len(ch) == 0, cap(ch) == 0

/////////////////////

ch <- 1 // block
v := <-ch

/////////////////////

// 先有goroutine正在接收
go func() {
    v := <-ch
    fmt.Println(v)
}()

ch <- 1
```

**buffered channel**
`ch := make(chan int, 100)`，buffered channel在满或空的情况下，分别会导致发送方和接收方阻塞。

**range**
`for i := range ch`可以从channel逐个接收值，直到channel被关闭。

**close**
* 发送方可以通过`close(ch)`来告诉接收方没有后续的值会发送。如果向关闭的channel发送元素，那么会导致抛出异常。
* 接收方可以使用`v, ok := <-ch`判断channel是否被关闭。如果从一个已经关闭的channel接收元素，会返回channel类型的零值，因此是不能用这个方式来判断channel是否关闭的。

**select**
select语句可以让goroutine等待多个通信操作（发送或接受都可以），block直到其中某个case能执行。如果同时有多个case能执行，则随机选择一个。

若存在default，则当没有case ready的时候，执行default，因此可以通过default实现非阻塞式的发送或接受。

```go
select {
case i := <-c:
    // use i
default:
    // receiving from c would block
}
```

## 并发模式
* [Go Concurrency Patterns](https://www.youtube.com/watch?v=f6kdp27TYZs)
* [Advanced Go Concurrency Patterns](https://www.youtube.com/watch?v=QDDwwePbDtw)

## 内存模型
go内存模型

# 反射

# 构建
## static build
`go build`启用race，也需要启用cgo。

# 测试

# 总结
抛开go的运行时环境和gc不说，go很像c，同时还有着少量函数式语言的特性。

go中，我很喜欢的几点是：

1. 变量和函数的声明简洁清晰
2. goroutine
3. 提供了CSP来实现goroutine之间的通信
