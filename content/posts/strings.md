---
title: "译: Strings, bytes, runes and characters in Go"
date: 2022-06-17
tags:
- strings
- bytes
- runes
- characters
summary: How strings work in Go, and how to use them.
---
## 英文原文

作者: Rob Pike

原文地址: https://go.dev/blog/strings

## 简介

[上篇博客](https://blog.golang.org/slices)阐述了切片在Go中如何工作，并列举了很多例子来说明它的机制。
基于这个背景，我们将在这篇博客中讨论Go中字符串。
首先，对于一篇博客来说字符串这个话题似乎太简单了，但要用好字符串不仅要理解字符串是怎么工作的，
还要掌握一个byte，一个character，一个rune的区别，
Unicode和UTF-8的区别，
字符串和字符串字面量的区别，
以及其他更细微的区别。

只有一种方法可以让你切入这个话题，那就是思考一些常见问题的答案，
如 "为什么我不能在Go字符串的第n个位置，得到第n个字符?"
如你所见，这个问题会引导我们了解有关文本在现代世界中如何工作的许多细节。

对其中一些问题的精彩介绍，是 Joel Spolsky 的著名博文，[The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html)。
他提出的许多观点将在这里得到回应。

## 什么是字符串？

让我们先从一些基础的东西开始。

在Go中，字符串实际上是只读的字节切片。
如果你完全不确定字节切片是什么以及它是如何工作的，请阅读[上篇博客](https://blog.golang.org/slices)；
现在我们假设你有这些知识。

重要的是要预先声明。
首先我们要在前面声明一个可以包含任意字节的字符串。
它不需要保存 Unicode 文本、UTF-8文本或任何其他预定义格式。
就字符串的内容而言，它完全等同于一个字节切片。

这里是使用`\xNN` 符号来定义一个包含一些特殊字节值的字符串常量。
（字节范围从十六进制值 00 到 FF，包括在内。）
```Go
	const sample = "\xbd\xb2\x3d\xbc\x20\xe2\x8c\x98"
```

## 打印字符串

因为我们sample字符串有些字符并不是有效的ASCII字符，甚至不是有效的UTF-8字符，
直接打印它们会导致你的输出非常难看。这个简单的打印

```Go
	fmt.Println(sample)
```

会输出这种乱码 （它的最终外观因环境而异）:

	��=� ⌘

为了搞清楚字符串包含了什么东西，我们需要把它拆开检查一下。
我们有几种方法可以做这件事情。
一个最明显的方法就算遍历它的内容，打出所有的字符。
比如下面这个`for`循环

```Go
    for i := 0; i < len(sample); i++ {
        fmt.Printf("%x ", sample[i])
    }
```

在前面这个演示中，对字符串进行索引访问的是单个独立字节，并不是整个字符。
现在，我们只用字节。
这个是从字节到字节的循环

	bd b2 3d bc 20 e2 8c 98

请注意各个字节如何与定义字符串的十六进制转义匹配。

在`fmt.Printf`中使用`%x`（十六进制）可以将一个有乱码的字符串打印的更美观易看些,也更简便。
它只是将字符串的连续字节转储为十六进制数字，每个字节两个。

```Go
    fmt.Printf("%x\n", sample)
```

可以用这个打印结果与上面的做对比:

	bdb23dbc20e28c98

有个很好的技巧是在格式化字符串中使用空格标志，在`%`和`x`之间多打一个空格。
可以用下面这个格式化打印和上面比较一下，

```Go
	fmt.Printf("% x\n", sample)
```

注意字节之间的空格是如何出现的:

	bd b2 3d bc 20 e2 8c 98

[还有更多的占位符](https://pkg.go.dev/fmt）。`%q`（带引号的，译注:使用 Go 语法安全转义的单引号字符文字。）
动词将转义字符串中任何不可打印的字节序列，因此输出是明确的。

```Go
    fmt.Printf("%q\n", sample)
```

当一个字符串大部分都是易读的文本时这种方法会很方便，但有一些特殊的字符要移除；它会打印出:

	"\xbd\xb2=\xbc ⌘"

如果仔细观察， 我们可以发现这段乱码中藏了一个ASCII等号，以及一个规则的空格，
最后面出现的是著名的瑞典“野营地”符号。
该符号的Unicode值是U+2318，将空格后面的字节编码为UTF-8十六进制值 `20`）：`e2` `8c` `98`。

如果一个字符串中有我们不认识的奇怪的值，我们可以在`%q`占位符使用"+"标志。
此标志在打印UTF-8时，不仅可以转义不可打印的序列，还能转义任何非ASCII字节。
这样，它会打印出一个正确的UTF-8编码的字符串，它包含了非ASCII字节的Unicode值:

```Go
	fmt.Printf("%+q\n", sample)
```

在这个格式化打印中，瑞典符号的Unicode值则被转义为`\u`:

	"\xbd\xb2=\xbc \u2318"

这些打印技巧在调试字符串的内容时很好用，并且在接下来的讨论中会派上用场。
值得指出的是，所有这些方法对**字节切片**的操作与对**字符串**的操作完全一样。

这里我编写了一个完整的程序，这个程序列举刚才我们涉及到的所有打印选项，你可以运行它（原网页可以运行）:

```go
package main

import "fmt"

func main() {
    const sample = "\xbd\xb2\x3d\xbc\x20\xe2\x8c\x98"

    fmt.Println("Println:")
    fmt.Println(sample)

    fmt.Println("Byte loop:")
    for i := 0; i < len(sample); i++ {
        fmt.Printf("%x ", sample[i])
    }
    fmt.Printf("\n")

    fmt.Println("Printf with %x:")
    fmt.Printf("%x\n", sample)

    fmt.Println("Printf with % x:")
    fmt.Printf("% x\n", sample)

    fmt.Println("Printf with %q:")
    fmt.Printf("%q\n", sample)

    fmt.Println("Printf with %+q:")
    fmt.Printf("%+q\n", sample)
}
```

[练习：修改上面的示例，用字节切片替换字符串。 提示：使用类型转换来创建切片。]

[练习：遍历字符串，使用`%q`打印每个字节。 你发现了什么？]

## UTF-8 和 字符串字面量

如我们所见， 遍历字符串会返回它的每个字节，而不是字符: 字符串就是一堆字节。
这就意味着，当我们存储一个字符值到字符串中时，我们存储的是组成那个字符的字节。
我们来看一个更加灵活的例子，看看这个过程是如何发生的。

这里有个简单的程序，它用三种方式打印一个字符串常量，
一种是作为一个纯字符串，一种是作为ASCII字符串，一种是作为十六进制的字节。
为了避免任何混淆，我们创建一个“原始字符串”，用反斜杠括起来，
这样它只能包含字面量文本。（用双引号括起来的常规字符串可以包含如上所示的转义序列。）
```Go
func main() {
    const placeOfInterest = `⌘`

    fmt.Printf("plain string: ")
    fmt.Printf("%s", placeOfInterest)
    fmt.Printf("\n")

    fmt.Printf("quoted string: ")
    fmt.Printf("%+q", placeOfInterest)
    fmt.Printf("\n")

    fmt.Printf("hex bytes: ")
    for i := 0; i < len(placeOfInterest); i++ {
        fmt.Printf("%x ", placeOfInterest[i])
    }
    fmt.Printf("\n")
}
```

输出是这样的:

	plain string: ⌘
	quoted string: "\u2318"
	hex bytes: e2 8c 98

我们可以看到Unicode字符值是U+2318，
瑞典“野营地”符号的字节则是`e2` `8c` `98`，
这些字节是十六进制值 2318 的 UTF-8 编码。

这可能一眼就可以看出来，也可能不那么明显，
这取决于你对UTF-8的熟悉程度，
但值得花一点时间来解释一下如何用UTF-8表示字符串。
一个简单的事实是：它是在编写源码时就定下了。

Go 中的源代码被定义成 UTF-8 文本；其他均不被容许。
这意味着，在源码中，我们写入的文本中，编辑器将符号 ⌘ 的 UTF-8 编码的放入源文本。
当我们打印出十六进制字节时，我们只是输出编辑器放入文件中的数据。

简而言之，Go源代码采用UTF-8编码，因此源码中的字符串字面量是UTF-8文本。
如果字符串字面量包含未转义字符序列，那么它的构造字符串将只包含引号中的源文本。
这意味着，原始字符串总是包含它的内容的有效UTF-8表示。
因此，经过定义和构造的原始字符串，它的内容总是包含有效的UTF-8表示。
同样，除非它包含UTF-8中断转义序列，否则一般的字符串字面量也将包含有效的UTF-8。

一些人认为Go字符串总是UTF-8，其实不是：只有字符串字面量是UTF-8。
我们在上一节中已经提到，字符串值可以包含任意字节；
我们在这一节中也提到，只要字符串字面量没有字节级别的转义序列，它就总是包含UTF-8文本。

总结一下就是，字符串可以包含任意字节，但当从字符串字面量构造时，这些字节都是（绝大多数情况下）UTF-8。
## 代码点（code point），字符，和 runes

到目前为止，我们在使用“字节”和“字符”这两个词时都非常小心。
一部分是因为字符串由字节组成，另一部分原因是因为“字符”的概念难以定义。
Unicode标准使用“代码点”来指代单个值所表示的东西。
代码点U+2318，十六进制值2318，就是符号⌘。
（更多关于这个代码点的信息，请参见[它的Unicode页面](http://unicode.org/cldr/utility/character.jsp?a=2318)）。

举一个平淡无奇的例子，Unicode 代码点 U+0061 是小写拉丁字母'A': a。

但小写重音字母 'A'， à呢？
这是一个字符，同样也是一个代码点（U+00E0），但它有其他表示。
比如，我们可以使用“组合”重音符号代码点，U+0300，
并将它附加到小写字母 a，U+0061，来创建相同的字符 à。
一般来说，一个字符可以由一个或多个代码点表示，而不同的代码点序列可能对应不同的UTF-8字节序列。

在计算学中，字符这个概念是很模糊的，或者至少是令人困惑的，所以我们在使用它时非常小心。
现在还是有一些标准化的技术可以保证一个字符总是由相同的代码点表示，但是话题就离我们现在所讨论的太远了。
我们会在后续的博文中讲解Go标准库如何解决标准化的问题。

“代码点“有点囫囵吞枣了，所以Go为这个概念提供了一个更短的术语:_rune_。
该术语出现在标准库和源代码中，其含义与“代码点”完全相同，但有一个有趣的补充。

Go语言中定义了这个词`rune`，它是int32的别名，因此程序可以清楚地表示一个整数代码点。
另一方面，Go中被称为rune常量的可能是你所认为的字符常量。
这个表达式的类型和值

	'⌘'

是整型为`0x2318`的`rune`。

总结一下，这里有几个重点:

  - Go源码编码是UTF-8。
  - 一个字符串可以包含任何字节。
  - 一个没有任何字节级别的转义字符串字面量，总是包含有效的UTF-8序列。
  - 可以代表Unicode代码点的序列，称为rune。
  - Go不保证字符串中的字符被规范化过。
## 循环遍历

除Go源码编码是UTF-8这个事情我们不再赘述，还有一个就是在Go中只有一种特殊方式可以处理UTF-8， 
这个方式就是使用一个`for` `range`循环遍历一个字符串。

我们前面已经看过常规的for循环会发生什么。
相反的， `for` `range`循环会在每次循环中解码一个UTF-8编码的rune。
每次循环中， `for` `range`循环的索引是当前rune的起始位置，以字节为单位，而代码点就是它的值。
这里有一个例子，使用便捷的`Printf`进行格式化打印，占位符`%#U`可以显示代码点的Unicode值和它的实际打印。

```Go
	const nihongo = "日本語"
    for index, runeValue := range nihongo {
        fmt.Printf("%#U starts at byte position %d\n", runeValue, index)
    }
```

输出显示每个代码点如何占用多个字节:

	U+65E5 '日' starts at byte position 0
	U+672C '本' starts at byte position 3
	U+8A9E '語' starts at byte position 6

[ 练习：如何将一个无效的UTF-8字节序列放入字符串中。循环遍历会发生什么？]

## 标准库

Go标准库为 UTF-8 提供了强大的解译能力。
如果一个`for` `range`不能满足你的需求，那么你需要在标准库中寻找你所需要的包。

一个非常重要的包是[`unicode/utf8`](https://pkg.go.dev/unicode/utf8)， 
它包含验证、分解和重组 UTF-8 字符串的辅助例程。
这里还有一个程序可以达到和`for` `range`例子一样效果， 
只是需要额外调用包中`DecodeRuneInString`函数才行。
函数的返回值是rune和它在UTF-8编码字节的宽度。

```go
    const nihongo = "日本語"
    for i, w := 0, 0; i < len(nihongo); i += w {
        runeValue, width := utf8.DecodeRuneInString(nihongo[i:])
        fmt.Printf("%#U starts at byte position %d\n", runeValue, i)
        w = width
    }
```

运行它看看是不是一样的效果。
`for` `range`循环和`DecodeRuneInString`都是定义用来生成一样的序列。

看这个[文档](https://pkg.go.dev/unicode/utf8)，
你可以看到`unicode/utf8`包提供的其他功能。

## 结论

回答一下开篇提到的问题: 字符串是由字节组成的，所以下标索引它们会返回字节，而不是字符。
字符串可能不包含字符。
事实上，“字符”的定义是模棱两可的，试图通过定义一个由字符组成的字符串来解决歧义是错误的。

关于 Unicode， UTF-8，多语言文本处理还有很多可以说， 但你要等我们另一篇文章了。
现在， 我们希望你能够更好地理解 Go 字符串，
尽管它可以包含任意字节，但UTF-8是 Go 字符串设计的核心部分。