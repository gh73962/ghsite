---
title: "译: Errors are values"
date: 2022-03-26
tags:
- error
summary: 
---

## 原文链接

https://go.dev/blog/errors-are-values

## 译文

如何处理错误是Go程序员之前经常讨论的话题,尤其是那些Go的初学者. 当多次`if err != nil`代码出现, 这种讨论常常会变为对Go处理错误的失望. 我们最近扫了很多的开源代码,并从中发现`if err != nil`的代码,每一两页才出现一次,比一些人说的要低. 尽管如此, 仍有看法坚持认为在Go语言里不断的写`if err != nil`来处理错误肯定是不对的.      

这种看法是具有误导性,但很容易纠正. 当一个Go初学者问到如何处理错误时,他们只需学习这种模式处理错误即可. 在其他的编程语言里,一部分程序员可能是使用try-catch模式或者其他类似的模式来处理错误. 因此当一个程序员在以前的编程里使用try-catch模式处理错误,现在在Go里需要`if err != nil`,并且大部分时间都需要写这种代码,他们会觉得很蹩脚很不灵活.        

不论这种解释有多符合这些情况,很明显的是这类Go程序员忽略了错误是一个值, 这在Go 语言中是最重要的一个基本点

Values can be programmed, and since errors are values, errors can be programmed.    

Of course a common statement involving an error value is to test whether it is nil, but there are countless other things one can do with an error value, and application of some of those other things can make your program better, eliminating much of the boilerplate that arises if every error is checked with a rote if statement.        
常见的错误处理就说检查它是否为 nil, 并且还有无数其他事情可以基于错误值来做，应用其中一些情况就可以使程序更好，如果每一个错误都使用一个机械的if语句来检查就可消除大部分的样板代码    

举个简单的例子, bufio包里的Scanner类型. 它的Scan方法执行底层IO,这种肯定会导致错误. 然而，Scan方法根本不会暴露错误。相反，它返回一个布尔值，并在扫描结束时运行一个单独的方法来报告是否发生了错误。代码见下       
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

很明显这里就检查了一次错误值是否为nil. Scan方法可以被定义为这样
`func (s *Scanner) Scan() (token []byte, error)`

使用示例代码如下
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

这并没什么太大的不同,但这里有一个重要区别. 在这段代码中，客户端必须在每次循环中检查错误，但是在真正的Scanner API中，错误处理是从关键API元素中抽象出来的，该元素在token上迭代。使用真正的API，调用代码使用起来也更自然:循环直到完成，然后考虑处理错误。错误处理并不会模糊你的控制流。        

封装的代码里面错误处理逻辑则是一旦Scan遇到IO错误，它就会记录并返回false, 当调用Err就会返回错误. 尽管这很简单，但它与到处写`if err != nil`或要求调用者在每个标记后检查错误并不同。它是通过错误值来编程的. 本质上也是一种很简单的编程模式,        

不论程序是如何设计的,不论错误是如何暴露的,检查错误都是至关重要的.这里讨论的不是如何避免检查错误，而是如何使用编程语言优雅地处理错误。       

当我在东京参加2014年秋季GoCon大会时，重复错误检查代码的话题就已经出现过了。一位热情的gopher对错误检查提出同样的抱怨, 代码如下       
```Go
_, err = fd.Write(p0[a:b])
`if err != nil` {
    return err
}
_, err = fd.Write(p1[c:d])
`if err != nil` {
    return err
}
_, err = fd.Write(p2[e:f])
`if err != nil` {
    return err
}
// and so on
```

这段代码有多重复的错误检查. 即使实际的工程代码里,这也是很长的了,通过使用helper 函数也不能很好的重构代码,但在这种理想的处理错误的形式中,声明一个函数以闭包的形式来处理error对你会有所帮助     
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

我们可以借用上面的Scan方法，使其更干净、更通用、更好的复用性。我在我们的讨论中提到了这种技术，但是@jxck不知道如何应用它。经过长时间的交流，我问他是否可以借用他的笔记本电脑来演示一些代码给他看。       

我定义了一个对象为errWriter     
```Go
type errWriter struct {
    w   io.Writer
    err error
}
```

赋予errWriter一个write方法. 它并不需要标准的Write函数签名,并且小写他们以示区别. write方法调用Write方法以使用底层的Writer,记录首个错误以便参考       
```Go
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

一旦发生错误，write方法就会变成空操作，但错误值会被保存下来。使用errWriter类型及其write方法，可以这样重构上面的代码
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

这比使用闭包更简洁，也使实际的写操作序列可读性更好。再也不会显得杂乱。使用错误值(和接口)编程使代码更好。        

同一个包里的这类代码大概率都可以通过这种写法构建, 甚至可以直接使用errWriter     

而且,只要有errWriter, 还有更多的能力来帮助处理,尤其是那些更少需要人力付出的例子. 它可以累积计数字符串的长度,再合并写到一个缓冲区,再通过原子操作传输. 等等       

事实上,这种模式经常出现再Go标准库里, archive/zip 和net/http 都使用了这种模式. 更重要的是, bufio包里的Writer是实现了errWriter. 尽管 bufio.Writer.Write 返回了错误, 这也是为了实现 io.Writer 的interface. bufio.Writer 的Write方法错误处理的方式和之前errWriter.write差不多,使用Flush返回错误,所以我们的示例如下      
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

我们只讨论了一种避免重复处理错误的方式。我们必须要认识到使用errWriter或bufio.Writer并不是简化错误处理的唯一方法，因为这种方法并不适用于所有情况。不论如何，牢记错误就是值，Go语言的特质适用于解决错误处理       

使用Go语言来简化你的错误处理        

但是要记住: 不论在写什么功能,一定要检查你的error        