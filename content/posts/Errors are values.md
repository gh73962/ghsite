---
title: "译: Errors are values"
date: 2022-03-26
tags:
- error
summary: Idioms and patterns for handling errors in Go.
---

## 原文链接
Author : Rob Pike       

Link: https://go.dev/blog/errors-are-values     

## 译文

如何处理错误是Go程序员之间经常讨论的话题，尤其是那些Go的初学者。当多次`if err != nil`代码出现，这种讨论常常会变为对这种模式的失望。
我们最近扫描了所有能找到的开源项目，并从中发现`if err != nil`每一两页才出现一次，比你想象的要少。
尽管如此，仍有看法坚持认为重复的写`if err != nil`来处理错误肯定是不对的,尤其是在在Go语言里。      

这种看法是具有误导性，但很容易纠正。
当一个Go初学者问到如何处理错误时，他们只需学习这种模式处理错误即可。
在其他的编程语言里，一部分程序员可能是使用try-catch模式或者其他类似的模式来处理错误。
因此当一个程序员在以前的编程里使用try-catch模式处理错误，现在在Go里需要`if err != nil`，并且大部分时间都需要写这种代码，他们会觉得很蹩脚很不灵活。        

不管这个解释是否合适，很明显这些Go程序员错过了关于错误的一个基本点：错误是值(_Errors are values_)。

值是可以被编程的，当一个错误成为了值，那么错误也可以被编程。

常见的错误处理就说检查它是否为`nil`，并且还有无数其他事情可以基于错误值来做，应用其中一些情况就可以使程序更好，如果每一个错误都使用一个机械的if语句来检查就可消除大部分的样板代码。    

举个简单的例子，`bufio`包里的`Scanner`类型。 它的`Scan`方法执行底层I/O，这种肯定会导致错误。 然而，`Scan`方法根本不会暴露错误。
相反，它返回一个布尔值，并在扫描结束时运行一个单独的方法来报告是否发生了错误。代码见下
```Go
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // process token
}
if err := scanner.Err(); err != nil {
    // process the error
}
```

当然，有一个`nil`检查错误，但它只出现并执行一次。Scan方法可以被定义为这样
`func (s *Scanner) Scan() (token []byte, error)`        


示例调用代码可能如下(取决于token怎么被检索)
```Go
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    `if err != nil` {
        return err // or maybe break
    }
    // process token
}
```

这两个例子并没有太多不同，但这里有一个重要的区别。那就是在这段代码中，客户端必须在每次循环中检查错误，但是在真正的`Scanner`API中，错误处理是从关键API元素中抽象出来的，该元素在token上迭代。使用真正的API，调用代码使用起来也更自然:循环直到完成，然后考虑处理错误。错误处理并不会掩盖你的控制流。        

底层代码所发生发生的情况则是一旦`Scan`遇到一个I/O错误，它就会记录错误并返回`false`，当调用代码时,独立的`Err`方法就会返回错误。
尽管这些微不足道，但这与将
```go
if err != nil
```
放在任何地方或要求调用者每次获得token后检查错误并不同。
它是通过错误值来编程的。 本质上也是一种很简单的编程模式。        

不论程序是如何设计的，不论错误是如何暴露的，检查错误都是至关重要的。这里讨论的不是如何避免检查错误，而是如何使用编程语言优雅地处理错误。       

当我在东京参加2014年秋季GoCon大会时，重复错误检查代码的话题就已经出现过了。一位热情的gopher对错误检查表达了类似的失望，代码如下       
```Go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```

这段代码有多重复的错误检查。 即使实际的工程代码里，这也是很长的了，通过使用helper 函数也不能很好的重构代码，
但在这种理想的处理错误的形式中，声明一个函数以闭包的形式来处理error对你会有所帮助     
```go
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// and so on
if err != nil {
    return err
}
```

这种模式效果很好，但是需要在每个执行写操作的函数中都有一个闭包; 单独的helper函数使用起来比较笨拙，因为需要在调用之间维护err变量(尝试一下)       

我们可以借用上面的`Scan`方法，使其更干净、更通用、更好的复用性。我在我们的讨论中提到了这种技术，但是@jxck不知道如何应用它。
经过长时间的交流，我问他是否可以借用他的笔记本电脑来演示一些代码给他看。       

我定义了一个对象为`errWriter`如下     
```Go
type errWriter struct {
    w   io.Writer
    err error
}
```

赋予`errWriter`一个`write`方法。 它并不需要标准库的`Write`函数签名，并且小写他们以示区别。 `write`方法调用`Write`方法以使用底层的`Writer`，记录首个错误以便参考。       
```Go
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

一旦发生错误，`write`方法就会变成空操作，但错误值会被保存下来。使用`errWriter`类型及其`write`方法，可以这样重构上面的代码。
```Go
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

这比使用闭包更简洁，也使实际的写操作序列可读性更好。再也不会显得杂乱。使用错误值(和interface接口)编程使代码更好。        

同一个包里的这类代码大概率都可以通过这种写法构建， 甚至可以直接使用`errWriter `    

而且，只要有`errWriter`， 还有更多的能力来帮助处理，尤其是那些更少需要人力付出的例子。 它可以累积计数字符串的长度，再合并写到一个缓冲区，再通过原子操作传输。 等等       

事实上，这种编程模式经常出现再Go标准库里， `archive/zip` 和`net/http` 都使用了这种模式。 更重要的是， `bufio`包里的`Writer`是实现了`errWriter`。
尽管` bufio.Writer.Write` 返回了错误， 这也是为了实现` io.Writer` 的接口。
`bufio.Writer` 的`Write`方法错误处理的方式和之前`errWriter.write`差不多，使用`Flush`返回错误，所以我们的示例如下      
```Go
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// and so on
if b.Flush() != nil {
    return b.Flush()
}
```

这种方法有个明显缺点，至少对于某些程序来说它无法知道在错误发生之前完成了多少次处理。如果这些信息很重要，那么就需要一种更细粒度的方法。不过，通常情况下，在最后进行全有或全无的检查错误就足够了。        

我们只讨论了一种避免重复处理错误的方式。我们必须要认识到使用`errWriter`或`bufio.Writer`并不是简化错误处理的唯一方法，因为这种方法并不适用于所有情况。不论如何，牢记错误就是值，Go语言的特质适用于解决错误处理。       

使用Go语言来简化你的错误处理。        

但是要记住: 不论在写什么功能，一定要检查你的error !        

最后, 如果你想看我和 @jxck_ 交流的过程, 和一些他记录的语音, 可以查看[他的博客](https://jxck.hatenablog.com/entry/golang-error-handling-lesson-by-rob-pike).