https://go.dev/blog/errors-are-values

A common point of discussion among Go programmers, especially those new to the language, is how to handle errors. The conversation often turns into a lament at the number of times the sequence `if err != nil`  shows up. We recently scanned all the open source projects we could find and discovered that this snippet occurs only once per page or two, less often than some would have you believe. Still, if the perception persists that one must type `if err != nil` all the time, something must be wrong, and the obvious target is Go itself.       
如何处理错误是Go程序员之前经常讨论的话题,尤其是那些Go的初学者. 当多次`if err != nil`代码出现, 这种讨论常常会变为对Go处理错误的失望. 我们最近扫了很多的开源代码,并从中发现`if err != nil`的代码,每一两页才出现一次,比一些人说的要低. 尽管如此, 仍有看法坚持认为在Go语言里不断的写`if err != nil`来处理错误肯定是不对的.      

This is unfortunate, misleading, and easily corrected. Perhaps what is happening is that programmers new to Go ask, “How does one handle errors?”, learn this pattern, and stop there. In other languages, one might use a try-catch block or other such mechanism to handle errors. Therefore, the programmer thinks, when I would have used a try-catch in my old language, I will just type `if err != nil` in Go. Over time the Go code collects many such snippets, and the result feels clumsy.       
这种看法是具有误导性,但很容易纠正. 当一个Go初学者问到如何处理错误时,他们只需学习这种模式处理错误即可. 在其他的编程语言里,一部分程序员可能是使用try-catch模式或者其他类似的模式来处理错误. 因此当一个程序员在以前的编程里使用try-catch模式处理错误,现在在Go里需要`if err != nil`,并且大部分时间都需要写这种代码,他们会觉得很蹩脚很不灵活.        

Regardless of whether this explanation fits, it is clear that these Go programmers miss a fundamental point about errors: Errors are values.        
不论这种解释有多符合这些情况,很明显的是这类Go程序员忽略了错误是一个值, 这在Go 语言中是最重要的一个基本点

Values can be programmed, and since errors are values, errors can be programmed.    

Of course a common statement involving an error value is to test whether it is nil, but there are countless other things one can do with an error value, and application of some of those other things can make your program better, eliminating much of the boilerplate that arises if every error is checked with a rote if statement.        
常见的错误处理就说检查它是否为 nil, 并且还有无数其他事情可以基于错误值来做，应用其中一些情况就可以使程序更好，如果每一个错误都使用一个机械的if语句来检查就可消除大部分的样板代码    

Here’s a simple example from the bufio package’s Scanner type. Its Scan method performs the underlying I/O, which can of course lead to an error. Yet the Scan method does not expose an error at all. Instead, it returns a boolean, and a separate method, to be run at the end of the scan, reports whether an error occurred. Client code looks like this:      
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


Sure, there is a nil check for an error, but it appears and executes only once. The Scan method could instead have been defined as
很明显这里就检查了一次错误值是否为nil. Scan方法可以被定义为这样
`func (s *Scanner) Scan() (token []byte, error)`


and then the example user code might be (depending on how the token is retrieved),
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

This isn’t very different, but there is one important distinction. In this code, the client must check for an error on every iteration, but in the real Scanner API, the error handling is abstracted away from the key API element, which is iterating over tokens. With the real API, the client’s code therefore feels more natural: loop until done, then worry about errors. Error handling does not obscure the flow of control.      
这并没什么太大的不同,但这里有一个重要区别. 在这段代码中，客户端必须在每次循环中检查错误，但是在真正的Scanner API中，错误处理是从关键API元素中抽象出来的，该元素在token上迭代。使用真正的API，调用代码使用起来也更自然:循环直到完成，然后考虑处理错误。错误处理并不会模糊你的控制流。        

Under the covers what’s happening, of course, is that as soon as Scan encounters an I/O error, it records it and returns false. A separate method, Err, reports the error value when the client asks. Trivial though this is, it’s not the same as putting `if err != nil` everywhere or asking the client to check for an error after every token. It’s programming with error values. Simple programming, yes, but programming nonetheless.       
封装的代码里面错误处理逻辑则是一旦Scan遇到IO错误，它就会记录并返回false, 当调用Err就会返回错误. 尽管这很简单，但它与到处写`if err != nil`或要求调用者在每个标记后检查错误并不同。它是通过错误值来编程的. 本质上也是一种很简单的编程模式,        

It’s worth stressing that whatever the design, it’s critical that the program check the errors however they are exposed. The discussion here is not about how to avoid checking errors, it’s about using the language to handle errors with grace.      
不论程序是如何设计的,不论错误是如何暴露的,检查错误都是至关重要的.这里讨论的不是如何避免检查错误，而是如何使用编程语言优雅地处理错误。       

The topic of repetitive error-checking code arose when I attended the autumn 2014 GoCon in Tokyo. An enthusiastic gopher, who goes by @jxck_ on Twitter, echoed the familiar lament about error checking. He had some code that looked schematically like this:     
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

It is very repetitive. In the real code, which was longer, there is more going on so it’s not easy to just refactor this using a helper function, but in this idealized form, a function literal closing over the error variable would help:        
这段代码有多重复的错误检查. 即使实际的工程代码里,这也是很长的了,通过使用helper 函数也不能很好的重构代码,但在这种理想的处理错误的形式中,声明一个函数以闭包的形式来处理erro对你会有所帮助     
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
This pattern works well, but requires a closure in each function doing the writes; a separate helper function is clumsier to use because the err variable needs to be maintained across calls (try it).     
这种模式效果很好，但是需要在每个执行写操作的函数中都有一个闭包; 单独的helper函数使用起来比较笨拙，因为需要在调用之间维护err变量(尝试一下)       

We can make this cleaner, more general, and reusable by borrowing the idea from the Scan method above. I mentioned this technique in our discussion but @jxck_ didn’t see how to apply it. After a long exchange, hampered somewhat by a language barrier, I asked if I could just borrow his laptop and show him by typing some code.      
我们可以借用上面的Scan方法，使其更干净、更通用、更好的复用性。我在我们的讨论中提到了这种技术，但是@jxck不知道如何应用它。经过长时间的交流，我问他是否可以借用他的笔记本电脑来演示一些代码给他看。       

I defined an object called an errWriter, something like this:       
我定义了一个对象为errWriter     
```Go
type errWriter struct {
    w   io.Writer
    err error
}
```
and gave it one method, write. It doesn’t need to have the standard Write signature, and it’s lower-cased in part to highlight the distinction. The write method calls the Write method of the underlying Writer and records the first error for future reference:      
赋予errWriter一个write方法. 它并不需要标准的Write函数签名,并且小写他们以示区别. write方法调用Write方法以使用底层的Writer,记录首个错误以便参考       
```Go
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

As soon as an error occurs, the write method becomes a no-op but the error value is saved.
Given the errWriter type and its write method, the code above can be refactored:        
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

This is cleaner, even compared to the use of a closure, and also makes the actual sequence of writes being done easier to see on the page. There is no clutter any more. Programming with error values (and interfaces) has made the code nicer.        
这比使用闭包更简洁，也使实际的写操作序列可读性更好。再也不会显得杂乱。使用错误值(和接口)编程使代码更好。        

It’s likely that some other piece of code in the same package can build on this idea, or even use errWriter directly.       
同一个包里的这类代码大概率都可以通过这种写法构建, 甚至可以直接使用errWriter     

Also, once errWriter exists, there’s more it could do to help, especially in less artificial examples. It could accumulate the byte count. It could coalesce writes into a single buffer that can then be transmitted atomically. And much more.        
而且,只要有errWriter, 还有更多的能力来帮助处理,尤其是那些更少需要人力付出的例子. 它可以累积计数字符串的长度,再合并写到一个缓冲区,再通过原子操作传输. 等等       

In fact, this pattern appears often in the standard library. The archive/zip and net/http packages use it. More salient to this discussion, the bufio package’s Writer is actually an implementation of the errWriter idea. Although bufio.Writer.Write returns an error, that is mostly about honoring the io.Writer interface. The Write method of bufio.Writer behaves just like our errWriter.write method above, with Flush reporting the error, so our example could be written like this:        
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

There is one significant drawback to this approach, at least for some applications: there is no way to know how much of the processing completed before the error occurred. If that information is important, a more fine-grained approach is necessary. Often, though, an all-or-nothing check at the end is sufficient.       
这种方法有个明显缺点，至少对于某些程序来说它无法知道在错误发生之前完成了多少次处理。如果这些信息很重要，那么就需要一种更细粒度的方法。不过，通常情况下，在最后进行全有或全无的检查错误就足够了。        

We’ve looked at just one technique for avoiding repetitive error handling code. Keep in mind that the use of errWriter or bufio.Writer isn’t the only way to simplify error handling, and this approach is not suitable for all situations. The key lesson, however, is that "errors are values" and the full power of the Go programming language is available for processing them.        
我们只讨论了一种避免重复处理错误的方式。我们必须要认识到使用errWriter或bufio.Writer并不是简化错误处理的唯一方法，因为这种方法并不适用于所有情况。不论如何，牢记错误就是值，Go语言的特质适用于解决错误处理       

Use the language to simplify your error handling.       
使用Go语言来简化你的错误处理        

But remember: Whatever you do, always check your errors!        
但是要记住: 不论在写什么功能,一定要检查你的error        