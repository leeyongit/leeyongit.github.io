---
title: go语言并发与并行——goroutine和channel的详细理解
categories: [技术, Golang]
tags: [golang]
---

如果不是我对真正并行的线程的追求，就不会认识到[Go](http://lib.csdn.net/base/go "Go知识库")有多么的迷人。

Go语言从语言层面上就支持了并发，这与其他语言大不一样，不像以前我们要用Thread库 来新建线程，还要用线程安全的队列库来共享数据。

以下是我入门的学习笔记。

Go语言的goroutines、信道和死锁
=====================

goroutine
---------

Go语言中有个概念叫做goroutine, 这类似我们熟知的线程，但是更轻。

以下的程序，我们串行地去执行两次`loop`函数:
```go
func loop() {
    for i := 0; i < 10; i++ {
        fmt.Printf("%d ", i)
    }
}

func main() {
    loop()
    loop()
}
```
毫无疑问，输出会是这样的:

    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9

下面我们把一个loop放在一个goroutine里跑，我们可以使用关键字`go`来定义并启动一个goroutine:
```go
func main() {
    go loop() // 启动一个goroutine
    loop()
}
```
这次的输出变成了:

    0 1 2 3 4 5 6 7 8 9

可是为什么只输出了一趟呢？明明我们主线跑了一趟，也开了一个goroutine来跑一趟啊。

原来，在goroutine还没来得及跑loop的时候，主函数已经退出了。

main函数退出地太快了，我们要想办法阻止它过早地退出，一个办法是让main等待一下:
```go
func main() {
    go loop()
    loop()
    time.Sleep(time.Second) // 停顿一秒
}
```
这次确实输出了两趟，目的达到了。

可是采用等待的办法并不好，如果goroutine在结束的时候，告诉下主线说“Hey, 我要跑完了！”就好了， 即所谓阻塞主线的办法，回忆下我们[Python](http://lib.csdn.net/base/python "Python知识库")里面等待所有线程执行完毕的写法:
```go
for thread in threads:
    thread.join()
```
是的，我们也需要一个类似`join`的东西来阻塞住主线。那就是信道

信道
--

信道是什么？简单说，是goroutine之间互相通讯的东西。类似我们Unix上的管道（可以在进程间传递消息）， 用来goroutine之间发消息和接收消息。其实，就是在做goroutine之间的内存共享。

使用`make`来建立一个信道:
```golang
var channel chan int = make(chan int)
// 或
channel := make(chan int)
```
那如何向信道存消息和取消息呢？ 一个例子:
```go
func main() {
    var messages chan string = make(chan string)
    go func(message string) {
        messages <- message // 存消息
    }("Ping!")

    fmt.Println(<-messages) // 取消息
}
```
默认的，信道的存消息和取消息都是阻塞的 (叫做无缓冲的信道，不过缓冲这个概念稍后了解，先说阻塞的问题)。

也就是说, 无缓冲的信道在取消息和存消息的时候都会挂起当前的goroutine，除非另一端已经准备好。

比如以下的main函数和foo函数:
```go
var ch chan int = make(chan int)

func foo() {
    ch <- 0  // 向ch中加数据，如果没有其他goroutine来取走这个数据，那么挂起foo, 直到main函数把0这个数据拿走
}

func main() {
    go foo()
    <- ch // 从ch取数据，如果ch中还没放数据，那就挂起main线，直到foo函数中放数据为止
}
```
那既然信道可以阻塞当前的goroutine, 那么回到上一部分「goroutine」所遇到的问题「如何让goroutine告诉主线我执行完毕了」 的问题来, 使用一个信道来告诉主线即可:
```go
var complete chan int = make(chan int)

func loop() {
    for i := 0; i < 10; i++ {
        fmt.Printf("%d ", i)
    }

    complete <- 0 // 执行完毕了，发个消息
}

func main() {
    go loop()
    <- complete // 直到线程跑完, 取到消息. main在此阻塞住
}
```
如果不用信道来阻塞主线的话，主线就会过早跑完，loop线都没有机会执行、、、

其实，无缓冲的信道永远不会存储数据，只负责数据的流通，为什么这么讲呢？

*   从无缓冲信道取数据，必须要有数据流进来才可以，否则当前线阻塞

*   数据流入无缓冲信道, 如果没有其他goroutine来拿走这个数据，那么当前线阻塞


所以，你可以[测试](http://lib.csdn.net/base/softwaretest "软件测试知识库")下，无论如何，我们测试到的无缓冲信道的大小都是0 (`len(channel)`)

如果信道正有数据在流动，我们还要加入数据，或者信道干涩，我们一直向无数据流入的空信道取数据呢？ 就会引起死锁

死锁
--

一个死锁的例子:
```go
func main() {
    ch := make(chan int)
    <- ch // 阻塞main goroutine, 信道c被锁
}
```
执行这个程序你会看到Go报这样的错误:

    fatal error: all goroutines are asleep - deadlock!

何谓死锁? [操作系统](http://lib.csdn.net/base/operatingsystem "操作系统知识库")有讲过的，所有的线程或进程都在等待资源的释放。如上的程序中, 只有一个goroutine, 所以当你向里面加数据或者存数据的话，都会锁死信道， 并且阻塞当前 goroutine, 也就是所有的goroutine(其实就main线一个)都在等待信道的开放(没人拿走数据信道是不会开放的)，也就是死锁咯。

我发现死锁是一个很有意思的话题，这里有几个死锁的例子:

1.  只在单一的goroutine里操作无缓冲信道，一定死锁。比如你只在main函数里操作信道:
```go
    func main() {
        ch := make(chan int)
        ch <- 1 // 1流入信道，堵塞当前线, 没人取走数据信道不会打开
        fmt.Println("This line code wont run") //在此行执行之前Go就会报死锁
    }
```
2.  如下也是一个死锁的例子:
```go
    var ch1 chan int = make(chan int)
    var ch2 chan int = make(chan int)

    func say(s string) {
        fmt.Println(s)
        ch1 <- <- ch2 // ch1 等待 ch2流出的数据
    }

    func main() {
        go say("hello")
        <- ch1  // 堵塞主线
    }
```
    其中主线等ch1中的数据流出，ch1等ch2的数据流出，但是ch2等待数据流入，两个goroutine都在等，也就是死锁。

3.  其实，总结来看，为什么会死锁？非缓冲信道上如果发生了流入无流出，或者流出无流入，也就导致了死锁。或者这样理解 Go启动的所有goroutine里的非缓冲信道一定要一个线里存数据，一个线里取数据，要成对才行 。所以下面的示例一定死锁:
```go
    c, quit := make(chan int), make(chan int)

    go func() {
       c <- 1  // c通道的数据没有被其他goroutine读取走，堵塞当前goroutine
       quit <- 0 // quit始终没有办法写入数据
    }()

    <- quit // quit 等待数据的写
```
    仔细分析的话，是由于：主线等待quit信道的数据流出，quit等待数据写入，而func被c通道堵塞，所有goroutine都在等，所以死锁。

    简单来看的话，一共两个线，func线中流入c通道的数据并没有在main线中流出，肯定死锁。


但是，是否果真 所有不成对向信道存取数据的情况都是死锁?

如下是个反例:
```go
func main() {
    c := make(chan int)

    go func() {
       c <- 1
    }()
}
```
程序正常退出了，很简单，并不是我们那个总结不起作用了，还是因为一个让人很囧的原因，main又没等待其它goroutine，自己先跑完了， 所以没有数据流入c信道，一共执行了一个goroutine, 并且没有发生阻塞，所以没有死锁错误。

那么死锁的解决办法呢？

最简单的，把没取走的数据取走，没放入的数据放入， 因为无缓冲信道不能承载数据，那么就赶紧拿走！

具体来讲，就死锁例子3中的情况，可以这么避免死锁:
```go
c, quit := make(chan int), make(chan int)

go func() {
    c <- 1
    quit <- 0
}()

<- c // 取走c的数据！
<-quit
```
另一个解决办法是缓冲信道, 即设置c有一个数据的缓冲大小:
```go
c := make(chan int, 1)
```
这样的话，c可以缓存一个数据。也就是说，放入一个数据，c并不会挂起当前线, 再放一个才会挂起当前线直到第一个数据被其他goroutine取走, 也就是只阻塞在容量一定的时候，不达容量不阻塞。

这十分类似我们Python中的队列`Queue`不是吗？

无缓冲信道的数据进出顺序
------------

我们已经知道，无缓冲信道从不存储数据，流入的数据必须要流出才可以。

观察以下的程序:
```go
var ch chan int = make(chan int)

func foo(id int) { //id: 这个routine的标号
    ch <- id
}

func main() {
    // 开启5个routine
    for i := 0; i < 5; i++ {
        go foo(i)
    }

    // 取出信道中的数据
    for i := 0; i < 5; i++ {
        fmt.Print(<- ch)
    }
}
```
我们开了5个goroutine，然后又依次取数据。其实整个的执行过程细分的话，5个线的数据 依次流过信道ch, main打印之, 而宏观上我们看到的即 无缓冲信道的数据是先到先出，但是 无缓冲信道并不存储数据，只负责数据的流通

缓冲信道
----

终于到了这个话题了, 其实缓存信道用英文来讲更为达意: buffered channel.

缓冲这个词意思是，缓冲信道不仅可以流通数据，还可以缓存数据。它是有容量的，存入一个数据的话 , 可以先放在信道里，不必阻塞当前线而等待该数据取走。

当缓冲信道达到满的状态的时候，就会表现出阻塞了，因为这时再也不能承载更多的数据了，「你们必须把 数据拿走，才可以流入数据」。

在声明一个信道的时候，我们给make以第二个参数来指明它的容量(默认为0，即无缓冲):
```go
var ch chan int = make(chan int, 2) // 写入2个元素都不会阻塞当前goroutine, 存储个数达到2的时候会阻塞
```
如下的例子，缓冲信道ch可以无缓冲的流入3个元素:
```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
}
```
如果你再试图流入一个数据的话，信道ch会阻塞main线, 报死锁。

也就是说，缓冲信道会在满容量的时候加锁。

其实，缓冲信道是先进先出的，我们可以把缓冲信道看作为一个线程安全的队列：
```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3

    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3
}
```
信道数据读取和信道关闭
-----------

你也许发现，上面的代码一个一个地去读取信道简直太费事了，Go语言允许我们使用`range`来读取信道:
```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3

    for v := range ch {
        fmt.Println(v)
    }
}
```
如果你执行了上面的代码，会报死锁错误的，原因是range不等到信道关闭是不会结束读取的。也就是如果 缓冲信道干涸了，那么range就会阻塞当前goroutine, 所以死锁咯。

那么，我们试着避免这种情况，比较容易想到的是读到信道为空的时候就结束读取:
```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3
for v := range ch {
    fmt.Println(v)
    if len(ch) <= 0 { // 如果现有数据量为0，跳出循环
        break
    }
}
```
以上的方法是可以正常输出的，但是注意检查信道大小的方法不能在信道存取都在发生的时候用于取出所有数据，这个例子 是因为我们只在ch中存了数据，现在一个一个往外取，信道大小是递减的。

另一个方式是显式地关闭信道:
```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
ch <- 3

// 显式地关闭信道
close(ch)

for v := range ch {
    fmt.Println(v)
}
```
被关闭的信道会禁止数据流入, 是只读的。我们仍然可以从关闭的信道中取出数据，但是不能再写入数据了。

等待多gorountine的方案
----------------

那好，我们回到最初的一个问题，使用信道堵塞主线，等待开出去的所有goroutine跑完。

这是一个模型，开出很多小goroutine, 它们各自跑各自的，最后跑完了向主线报告。

我们讨论如下2个版本的方案:

1.  只使用单个无缓冲信道阻塞主线

2.  使用容量为goroutines数量的缓冲信道


对于方案1, 示例的代码大概会是这个样子:
```go
var quit chan int // 只开一个信道

func foo(id int) {
    fmt.Println(id)
    quit <- 0 // ok, finished
}

func main() {
    count := 1000
    quit = make(chan int) // 无缓冲

    for i := 0; i < count; i++ {
        go foo(i)
    }

    for i := 0; i < count; i++ {
        <- quit
    }
}
```
对于方案2, 把信道换成缓冲1000的:
```go
quit = make(chan int, count) // 容量1000
```
其实区别仅仅在于一个是缓冲的，一个是非缓冲的。

对于这个场景而言，两者都能完成任务, 都是可以的。

*   无缓冲的信道是一批数据一个一个的「流进流出」

*   缓冲信道则是一个一个存储，然后一起流出去