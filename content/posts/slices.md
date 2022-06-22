---
title: "译: Arrays, slices (and strings): The mechanics of 'append'"
date: 2022-05-26
tags:
- array
- slice
- string
- copy
- append
summary: Go 数组和切片是如何工作的，以及如何使用copy和append。
---
## 原文

Author : Rob Pike

Link : https://golang.google.cn/blog/slices

## 简介

程序编程语言最常见的特征之一是数组的概念。
数组看起来很简单，但在将它们添加到语言时必须回答许多问题，例如：

  - 固定长度还是可变长度?
  - size是其类型的一部分吗?
  - 多维数组是什么样的?
  - 空数组意味着什么?

这些问题的答案会影响数组是否只是语言的一个特性还是其设计的核心部分。

在早期的Go开发中，这个设计成熟之前，这些问题花费我们快一年的时间。
关键的一步是引入切片， 切片是一种灵活易拓展的数据结构。
然而，直到今天，刚接触 Go 的程序员经常对切片的工作方式感到困惑，
也许是因为其他语言的经验影响了他们的思维。

我们将在这篇博客中为你答疑解惑，
还会阐述`append`内置函数是如何工作的。

## 数组

在Go中，数组是一个非常重要的基本构成元素，像建筑的桩基一样，他们都是隐藏在可见的组件之下。
在我们继续讨论更有趣、更强大、更突出的切片概念之前，我们必须简要讨论一下它们。

数组在Go中并不常见，因为数组的大小是它类型的一部分，这个特性限制了他们的表现力

这个声明
```go
     var buffer [256]byte
```
声明了变量 `buffer`，它包含 256 个字节。
`buffer` 的类型包括它的大小，`[256]byte`。
具有 512 个字节的数组将是不同类型的 `[512]byte`。

与数组关联的数据就是：一个元素数组。
从示意图上看，我们的缓冲区在内存中看起来像这样，

	buffer: byte byte byte ... 256 times ... byte byte byte

也就是说，该变量只保存 256 个字节的数据，仅此而已。
我们可以使用熟悉的索引语法`buffer[0]`、`buffer[1]`、`buffer[255]`等等（索引范围 0 到 255 涵盖 256 个元素）访问其元素。
尝试使用超出此范围的值来访问 `buffer` 会使程序崩溃。

这里有个内置函数`len` ，它会返回数组\切片\或其他类型的元素数量。
对于数组来说，`len`返回了什么一目了然。
在我们的示例中，`len(buffer)`返回了固定值256。

数组有很广泛的用途， 转换矩阵示例就是一个很好的代表。
但它们在 Go 中最常见的用途是为切片占住内存。

## 切片: The slice header

Slices are where the action is, but to use them well one must understand 
exactly what they are and what they do.

~~切片是动作所在~~(此句翻译待定，仅供参考)，但要用好切片必须理解它们是什么以及是如何工作的。

切片是一种数据结构，描述了一个数组的连续部分，但不与切片变量本身存储在一起。
**一个切片不是一个数组**， 一个切片**描述**了一个数组的一部分。

给定上一节中的 `buffer` 数组变量，我们可以创建一个切片，通过 _切片_ 数组来描述元素 100 到 150（准确地说是 100 到 149，含149）：
```go
	var slice []byte = buffer[100:150]
```
在这个代码段中，我们使用了完整的变量声明。 
变量`slice`拥有类型`[]byte`，意为多个字节的切片， 
并从`buffer`数组里面切分第100(包含)个元素至第150(不含)个元素而来。

以下是更简洁的初始化语法，在之前的声明上隐藏了声明类型:
```go
	var slice = buffer[100:150]
```
在函数内部，我们可以使用这种短声明的方式
```go
	slice := buffer[100:150]
```
那么这个切片变量到底是什么?

你可以把切片当作一个拥有两个元素的数据结构，一个元素是长度， 一个元素是指针，
这个指针指向一个数组里的元素。

你这样想象它在底层是这样构建的:
```go
	type sliceHeader struct {
		Length        int
		ZerothElement *byte
	}

	slice := sliceHeader{
		Length:        50,
		ZerothElement: &buffer[100],
	}
```
当然，这只是一个例证。尽管这段代码片段中的 `sliceHeader `结构对程序员是不可见的，
并且元素指针的类型取决于元素的类型，但这这段代码展示出了这个机制设计理念。

到目前为止，我们都是对数组进行切片，我们也可以对切片进行切片，如下:
```go
	slice2 := slice[5:10]
```
像前面这个操作，它创建了一个新的切片， 在这个例子中， 
原始切片第5至第9之间的元素，同时也是之前原始数组第105至109之间的元素。

变量`slice2`底层数据结构`sliceHeader`长这样:
```go
	slice2 := sliceHeader{
		Length:        5,
		ZerothElement: &buffer[105],
	}
```
注意这个header仍然指向同一个存储在`buffer`变量里的底层数组。

我们也可以**重新切片**，也就是说，切片一个切片，并将结果存储回原切片。
```go
	slice = slice[5:10]
```
`slice` 变量的`sliceHeader` 看起来就像`slice2` 变量的。

你会看到重新切片被广泛应用，例如截断切片。 此语句删除切片的第一个和最后一个元素：
```go
	slice = slice[1:len(slice)-1]
```
[练习：写出 `sliceHeader` 结构在这个赋值之后的样子。]

你会经常听到有经验的 Go 程序员谈论`slice header`，因为这确实是存储在切片变量中的内容。
例如，当你调用一个将切片作为参数的函数（例如 [bytes.IndexRune](https://golang.google.cn/pkg/bytes/#IndexRune)）时，该header就是实际传递给函数。

在这个调用中
```go
	slashPos := bytes.IndexRune(slice, '/')
```
`slice`参数传入`IndexRune`函数，实际上传入的是一个`slice header`。

`slice header` 中还有一个元素，我们将在后面讨论，
在此之前我们应先明白`slice header`在编程中意味着什么。

## 将切片传入函数

你一定要知道即使切片包含指针它本身也是一个值。
在表象之下，slice是一个包含指针和长度的结构体，它**不是**指向一个结构体的指针。

这很重要。

在上一个调用`IndexRune`示例中，他实际传入的是slice header的值拷贝。
这种行为有着重要的影响。

看看这个简单的函数:

```go
func AddOneToEachElement(slice []byte) {
    for i := range slice {
        slice[i]++
    }
}
```

这个函数的功能如其函数命名，遍历切片每一个元素，并为其+1。

尝试一下这个:

```go
func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }
    fmt.Println("before", slice)
    AddOneToEachElement(slice)
    fmt.Println("after", slice)
}
```

（如果您想探索，可以编辑并重新执行这些可运行的代码。）

尽管`slice header`是值传递，但header包含一个指向数组元素的指针，
因此传递给函数的原始`slice header`和header拷贝都描述相同的数组。

因此，当函数返回时，可以通过原始slice变量查看修改后的元素。

函数的参数实际上是一个副本，如下例所示：

```go
func SubtractOneFromLength(slice []byte) []byte {
    slice = slice[0 : len(slice)-1]
    return slice
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    newSlice := SubtractOneFromLength(slice)
    fmt.Println("After:  len(slice) =", len(slice))
    fmt.Println("After:  len(newSlice) =", len(newSlice))
}
```

这里我们看到，slice参数的**内容**可以由函数修改，但其标头不能。
存储在`slice`变量里的长度，是不能通过调用函数来修改的，
只要函数传入一个`slice header`，函数的slice就不再是原始slice。
因此，要是我们想写一个可以修改header的函数，我们必须将其作为结果参数返回，就像我们在这里所做的那样。
之前的`slice`变量并未被改动，但返回的`newSlice`具有新的长度。
## 切片的指针：方法接收器

让函数修改切片标头的另一种方法是向其传递指针。

下面是我们前面示例的一个变体，它可以做到这一点：

```Go
func PtrSubtractOneFromLength(slicePtr *[]byte) {
    slice := *slicePtr
    *slicePtr = slice[0 : len(slice)-1]
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    PtrSubtractOneFromLength(&slice)
    fmt.Println("After:  len(slice) =", len(slice))
}
```

在那个例子中，它看起来很笨拙，尤其是需要额外使用临时变量来完成，
但有一种常见的情况，你会看到指向切片的指针。
对于修改切片的方法，使用指针接收器是惯用做法。

假设我们有个切片的方法是在最后一个斜杠的地方截断，可以这样写:

```Go
type path []byte

func (p *path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(*p, []byte("/"))
    if i >= 0 {
        *p = (*p)[0:i]
    }
}

func main() {
    pathName := path("/usr/bin/tso") // Conversion from string to path.
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)
}
```

如果运行此示例，你可以看到调用这个方法后切片就被更新了。

[练习：将接收器的类型改为值而不是指针，然后再次运行。 解释发生了什么。]

另一方面，如果我们想为`path`编写一个方法，使`path`中的ASCII字母大写（局部忽略非英语名称），
那么该方法可以是一个值，因为值接收器仍将指向相同的底层数组。

```Go
type path []byte

func (p path) ToUpper() {
    for i, b := range p {
        if 'a' <= b && b <= 'z' {
            p[i] = b + 'A' - 'a'
        }
    }
}

func main() {
    pathName := path("/usr/bin/tso")
    pathName.ToUpper()
    fmt.Printf("%s\n", pathName)
}
```

这里，`ToUpper`方法使用`for` `range`构造中的两个变量来捕获索引和切片元素。
这种形式的循环避免了在正文中多次写入 `p[i]`。

[ 练习：将`ToUpper`方法转换为使用指针接收器，看看它的行为是否改变。]

[ 进阶练习：转换 `ToUpper` 方法来处理 Unicode 字母，而不仅仅是 ASCII。]

## Capacity 容量

查看以下函数，该函数将`ints`的参数片扩展了一个元素：

```Go
func Extend(slice []int, element int) []int {
    n := len(slice)
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

（为什么它需要返回修改后的切片？）现在运行它：

```go
func main() {
    var iBuffer [10]int
    slice := iBuffer[0:0]
    for i := 0; i < 20; i++ {
        slice = Extend(slice, i)
        fmt.Println(slice)
    }
}
```

查看切片如何增长，直到。。。 它不能运行

现在我们来谈谈`slice header`的第三个组成部分：它的**容量**。
除了数组指针和长度之外，`slice header`还存储其容量：

	type sliceHeader struct {
		Length        int
		Capacity      int
		ZerothElement *byte
	}

字段`Capacity` 记录了它底层数组实际有多少内存空间，这是长度可以达到的最大值。
试图将切片扩展到超出array的容量限制，则会导致panic。

通过这个例子创建一个切片后，

	slice := iBuffer[0:0]

它的header是这样的:

	slice := sliceHeader{
		Length:        0,
		Capacity:      10,
		ZerothElement: &iBuffer[0],
	}

`Capacity`字段等于底层数组的长度，减去切片第一个元素的数组中的索引（在本例中为零）。
如果要查询片的容量，请使用内置函数`cap`：

	if cap(slice) == len(slice) {
		fmt.Println("slice is full!")
	}

## Make

如果我们想将该片扩大到超出其容量，该怎么办？你做不到的！
根据定义，容量是增长的极限。
但是，你可以通过分配一个新数组、复制数据以及修改切片来描述新数组来实现相同的结果。

先从分配开始，我们可以使用内置函数`new`来分配到更大的数组，然后对结果进行切片，但使用`make`内置函数更简单。
它一次分配一个新数组并创建一个`slice header`来描述它。
`make`函数有三个参数：切片的类型、初始长度和容量，这是`make`函数分配用来保存切片数据的数组的长度。
这个命令创建了一个长度为10的切片，可以再容纳5个（15-10），运行它可以看到：

```Go
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

此片段将`int`切片的容量加倍，但保持其长度不变：

```Go
    slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
    newSlice := make([]int, len(slice), 2*cap(slice))
    for i := range slice {
        newSlice[i] = slice[i]
    }
    slice = newSlice
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

运行此代码后，切片在需要再次重新分配之前有更大的增长空间。

创建切片时，长度和容量通常是相同的。
`make`内置函数有此常见情况的简洁使用方式。
长度参数默认为容量，因此可以省去它，将两者设置为相同的值。

在

	gophers := make([]Gopher, 10)

之后， 切片变量`gophers`的长度和容量就都被设为10了。

## Copy

在上一节中，当我们将切片的容量加倍时，我们编写了一个循环，将旧数据复制到新切片。
Go有一个内置函数`copy`，使这个操作更容易。
它的参数是两个切片，并把右边切片的数据复制到左边切片。

`copy`使用示例如下

```Go
    newSlice := make([]int, len(slice), 2*cap(slice))
    copy(newSlice, slice)
```

`copy`函数是智能的。
它只复制它能复制的内容，但要注意两个参数的长度。
换句话说，它复制的元素数是两个切片长度中的最小值。
这可以节省一些开销。
此外，`copy`函数返回一个整数值，即它复制的元素数，尽管它并不总是值得检查。

当源和目标重叠时，`copy`函数也能正确处理问题，这意味着它可以用于在单个切片中来移动项目。
下面介绍如何使用`copy`将值插入到切片的中间。

```Go
// Insert inserts the value into the slice at the specified index,
// which must be in range.
// The slice must have room for the new element.
func Insert(slice []int, index, value int) []int {
    // Grow the slice by one element.
    slice = slice[0 : len(slice)+1]
    // Use copy to move the upper part of the slice out of the way and open a hole.
    copy(slice[index+1:], slice[index:])
    // Store the new value.
    slice[index] = value
    // Return the result.
    return slice
}
```

在这个函数中需要注意几件事。
当然，首先，它必须返回更新的切片，因为其长度已更改。
其次，使用了更为方便的快捷方式。
表达式

    slice[i:]

和

    slice[i:len(slice)]

是同一个意思

此外，虽然我们还没有使用技巧，但我们也可以省略切片表达式的第一个元素；
它的默认值是0，因此

    slice[:]

就代表它自己，这在切片一个数组时很有用
下面这种表达式的含义是"一个描述了数组所有元素的切片"

    array[:]

现在，让我们运行`Insert`函数。

```Go
    slice := make([]int, 10, 20) // Note capacity > length: room to add element.
    for i := range slice {
        slice[i] = i
    }
    fmt.Println(slice)
    slice = Insert(slice, 5, 99)
    fmt.Println(slice)
```

## Append: 一个示例


在几个章节之前，我们写了一个`Extend`函数，用来为切片增加一个元素。
但那段代码有bug的，因为 如果切片的容量太小，函数就会崩溃(我们的`Insert`函数也有同样的问题)。
现在我们已经准备好了修复该问题的各个部分，所以让我们为一个针对整数切片的`Extend`的实现写代码。

```Go
func Extend(slice []int, element int) []int {
    n := len(slice)
    if n == cap(slice) {
        // Slice is full; must grow.
        // We double its size and add 1, so if the size is zero we still grow.
        newSlice := make([]int, len(slice), 2*len(slice)+1)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

在这种情况下，返回切片尤其重要，因为当它重新分配结果切片时，描述的是一个完全不同的数组。
这里有一个小片段来演示当切片填满时会发生什么：

```Go
    slice := make([]int, 0, 5)
    for i := 0; i < 10; i++ {
        slice = Extend(slice, i)
        fmt.Printf("len=%d cap=%d slice=%v\n", len(slice), cap(slice), slice)
        fmt.Println("address of 0th element:", &slice[0])
    }
```

请注意，当大小为5的初始数组被填满时，会重新分配一片内存空间。
分配新数组时，第0个元素的容量和地址都会更改。

以增强后的`Extend`函数为指导，我们可以编写一个更好的通过多个元素扩展切片函数。
为此，在调用函数时， 我们使用了这种使一列函数参数转为一个切片的能力。
也就是说，我们使用 Go 的可变参数函数工具。

让我们调用一下`Append`函数。
对于第一个版本，我们可以重复调用 `Extend`，这样变参函数的机制就很清楚了。
`Append`的函数签名是:

    func Append(slice []int, items ...int) []int

这就是说 `Append` 接受一个参数，一个切片，后跟零个或多个`int` 参数。
就 `Append` 的实现而言，这些参数正是 `int` 的一部分，就如从代码看到的：

```Go
// Append appends the items to the slice.
// First version: just loop calling Extend.
func Append(slice []int, items ...int) []int {
    for _, item := range items {
        slice = Extend(slice, item)
    }
    return slice
}
```

注意 `for` `range` 循环遍历 `items` 参数的元素，它具有隐含的类型 `[]int`。
还要注意使用空白标识符`_`来丢弃循环中的索引，在这种情况下我们不需要。

尝试运行它:

```Go
    slice := []int{0, 1, 2, 3, 4}
    fmt.Println(slice)
    slice = Append(slice, 5, 6, 7, 8)
    fmt.Println(slice)
```

这个例子中的另一个新技术是我们通过编写一个合成文法来初始化切片，
它由切片的类型和大括号中的元素组成：

	    slice := []int{0, 1, 2, 3, 4}

`Append` 函数之所以有趣还有另一个原因。
我们不仅可以追加多个元素，还可以通过在调用时使用 `...` 将第二个切片“分解”成参数来追加到目标切片里：

```Go
    slice1 := []int{0, 1, 2, 3, 4}
    slice2 := []int{55, 66, 77}
    fmt.Println(slice1)
    slice1 = Append(slice1, slice2...) // The '...' is essential!
    fmt.Println(slice1)
```

当然， 我们可以通过修改`Extend`的内部实现，让其内存分配不超过1次， 
这样就会让`Append`函数的效率更高:

```Go
// Append appends the elements to the slice.
// Efficient version.
func Append(slice []int, elements ...int) []int {
    n := len(slice)
    total := len(slice) + len(elements)
    if total > cap(slice) {
        // Reallocate. Grow to 1.5 times the new size, so we can still grow.
        newSize := total*3/2 + 1
        newSlice := make([]int, total, newSize)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[:total]
    copy(slice[n:], elements)
    return slice
}

```

在这里，请注意我们如何使用 `copy` 两次，
一次是将切片数据移动到新分配的内存，
然后将追加项复制到旧数据的末尾。

尝试一下; 行为与以前相同：

```Go
    slice1 := []int{0, 1, 2, 3, 4}
    slice2 := []int{55, 66, 77}
    fmt.Println(slice1)
    slice1 = Append(slice1, slice2...) // The '...' is essential!
    fmt.Println(slice1)
```

## Append: 内置函数

这样我们就得出了`append`内置函数的设计动机。
它与我们的 `Append` 示例完全相同，效率相当，但它适用于任何切片类型。

Go 的一个弱点是任何泛型类型的操作都必须由运行时提供。 总有一天，情况可能会发生变化，但就目前而言，为了更轻松地使用切片，Go 提供了一个内置的通用 `append` 函数。
它的工作方式与我们的 `int` 切片版本相同，但适用于 **任何** 切片类型。

请记住，由于`slice header`总是通过调用 `append` 来更新，因此您需要在调用后保存返回的切片。
事实上，编译器不会让你在不保存结果的情况下调用`append`。

尝试它们，编辑它们并探索：

```Go
    // Create a couple of starter slices.
    slice := []int{1, 2, 3}
    slice2 := []int{55, 66, 77}
    fmt.Println("Start slice: ", slice)
    fmt.Println("Start slice2:", slice2)

    // Add an item to a slice.
    slice = append(slice, 4)
    fmt.Println("Add one item:", slice)

    // Add one slice to another.
    slice = append(slice, slice2...)
    fmt.Println("Add one slice:", slice)

    // Make a copy of a slice (of int).
    slice3 := append([]int(nil), slice...)
    fmt.Println("Copy a slice:", slice3)

    // Copy a slice to the end of itself.
    fmt.Println("Before append to self:", slice)
    slice = append(slice, slice...)
    fmt.Println("After append to self:", slice)
```

值得花点时间详细考虑该示例的最后一行，以了解切片的设计如何使这个简单的调用能够正常工作。

在社区构建的["Slice Tricks" Wiki page](https://github.com/golang/go/wiki/SliceTricks)
上还有更多的 `append`、`copy` 和其他使用切片的示例

## Nil

顺便说一句，利用我们新学到的知识，我们可以看到`nil`切片的表示形式是什么。
自然是`slice header`的零值：

	sliceHeader{
		Length:        0,
		Capacity:      0,
		ZerothElement: nil,
	}

 或只是

	sliceHeader{}

关键细节是元素指针也是`nil`。 由 

    array[0:0]

创建的切片长度为零（甚至容量为零），
但它的指针不是 `nil`，因此它不是 `nil` 切片。

应该清楚，一个空切片可以增长（假设它具有非零容量），
但是一个`nil`切片没有数组可以放入值，并且永远不会增长到容纳一个元素。

也就是说，`nil`切片在功能上等同于零长度切片，即使它不指向任何内容。
它的长度为零，可以通过分配附加到。
举个例子，看看上面的单行代码，它通过附加到一个 `nil` 切片来复制一个切片。

## 字符串

现在简要介绍切片上下文中的 Go 字符串。

字符串实际上非常简单：它们只是只读的字节切片，并在语言层面有一些额外语法支持。

因为它们是只读的，所以不需要容量（你不能扩容它们），但是对于大多数用途，您可以将它们视为只读字节片。

对于初学者，我们可以索引它们以访问单个字节：

	slash := "/usr/ken"[0] // yields the byte value '/'.


我们可以对字符串进行切片以获取子字符串：

	usr := "/usr/ken"[0:4] // yields the string "/usr"


现在，当我们对字符串进行切片时，幕后发生的事情应该很明显了。

我们还可以获取一个普通的字节切片，并通过简单的转换从中创建一个字符串：

	str := string(slice)

然后也可以反向进行：

	slice := []byte(usr)

字符串底层的数组是隐藏的； 除了通过字符串之外，其他方式无法访问其内容。 
这意味着当我们进行任何一种转换时，都必须制作数组的副本。
当然，Go 会处理这个问题，所以你不必这样做。
在这些转换中的任何一个之后，对字节切片底层数组的修改都不会影响相应的字符串。

这种类似切片的字符串设计的一个重要影响是可以非常高效的创建子字符串。
所需要做的就是创建一个two-word字符串header。
由于字符串是只读的，故原始字符串和切片操作产生的字符串可以安全地共享同一个数组。

一个历史：最早的字符串实现总是分配的，但是当切片被添加到语言中时，
它们提供了一个高效字符串处理的模型。
一些基准测试结果相对之前的结果获得巨大提升。

当然，还有更多关于字符串的东西可以讲，这有一个 [博文](https://golang.google.cn/blog/strings) 可以更深入地介绍了它们。

## 结论

了解了切片是如何工作的，有助于你了解它们是如何实现的。
有一个小数据结构，即切片header，即与切片变量关联的项，该header描述了单独分配的数组的一部分。
当我们传递切片值时，header会被复制，但它指向的数组始终是共享的。

一旦你了解它们的工作原理，切片使用起来会更加方便，
而且功能强大且有更强表达能力，尤其是在 `copy` 和 `append` 内置函数的帮助下。

## 延申阅读

关于 Go 中的 slices 的 intertubes 周围有很多东西可以找到。
如前所述，["Slice Tricks" Wiki page](/wiki/SliceTricks)有很多例子。
[Go Slices](https://golang.google.cn/blog/go-slices-usage-and-internals) 博客文章用清晰的图表描述了内存布局的细节。
Russ Cox 的 [Go Data Structures](https://research.swtch.com/godata) 文章讨论了切片以及 Go 的其他一些内部数据结构。

虽然有这么多的材料，但了解切片的最佳方法还是是使用它们写代码。
