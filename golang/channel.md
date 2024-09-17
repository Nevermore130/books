### 创建 channel ###

```go
ch1 := make(chan int)    //无缓冲 channel
ch2 := make(chan int, 5) //缓冲区长度为 capacity 的 channel 类型，是带缓冲 channel
```

### channel 的发送和接收 ###

channel 是用于 Goroutine 间通信的，所以绝大多数对 channel 的读写都被分别放在了不同的 Goroutine 中。

* Goroutine 对无缓冲 channel 的接收和发送操作是同步的。也就是说，对同一个无缓冲 channel，只有对它进行接收操作的 Goroutine 和对它进行发送操作的 Goroutine 都存在的情况下，通信才能得以进行，否则单方面的操作会让对应的 Goroutine 陷入挂起状态

```go
func main() {
    ch1 := make(chan int)
    go func() {
        ch1 <- 13 // 将发送操作放入一个新goroutine中执行
    }()
    n := <-ch1
    println(n)
}
```

**对无缓冲 channel 类型的发送与接收操作，一定要放在两个不同的 Goroutine 中进行，否则会导致 deadlock。**



* 对一个带缓冲 channel 来说，在缓冲区未满的情况下，对它进行发送操作的 Goroutine 并不会阻塞挂起；在缓冲区有数据的情况下，对它进行接收操作的 Goroutine 也不会阻塞挂起



使用操作符<-，我们还可以声明只发送 channel 类型（send-only）和只接收 channel 类型（recv-only）

```go
func produce(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i + 1
        time.Sleep(time.Second)
    }
    close(ch)
}

func consume(ch <-chan int) {
    for n := range ch {
        println(n)
    }
}

func main() {
    ch := make(chan int, 5)
    var wg sync.WaitGroup
    wg.Add(2)
    go func() {
        produce(ch)
        wg.Done()
    }()

    go func() {
        consume(ch)
        wg.Done()
    }()

    wg.Wait()
}
```

在消费者函数 consume 中，我们使用了 for range 循环语句来从 channel 中接收数据，for range 会阻塞在对 channel 的接收操作上，直到 channel 中有数据可接收或 channel 被关闭，才会继续向下执行。channel 被关闭后，for range 循环也就结束了。

### 关闭channel ###

produce 函数在发送完数据后，调用 Go 内置的 close 函数关闭了 channel。channel 关闭后，所有等待从这个 channel 接收数据的操作都将返回。**发送端负责关闭 channel。**

### select ###

当 select 语句中没有 default 分支，而且所有 case 中的 channel 操作都阻塞了的时候，整个 select 语句都将被阻塞，直到某一个 case 上的 channel 变成可发送，或者某个 case 上的 channel 变成可接收，select 语句才可以继续进行下去

### 无缓冲 channel 的惯用法 ###

**用作信号传递**

```go
type signal struct{}

func worker() {
    println("worker is working...")
    time.Sleep(1 * time.Second)
}

func spawn(f func()) <-chan signal {
    c := make(chan signal)
    go func() {
        println("worker start to work...")
        f()
        c <- signal{}
    }()
    return c
}

func main() {
    println("start a worker...")
    c := spawn(worker)
    <-c
    fmt.Println("worker work done!")
}
```

spawn 函数返回的 channel，**被用于承载新 Goroutine 退出的“通知信号”**，这个信号专门用作通知 main goroutine

有些时候，无缓冲 channel 还被用来实现 1 对 n 的信号通知机制。这样的信号通知机制，常被用于协调多个 Goroutine 一起工作

```go
func worker(i int) {
    fmt.Printf("worker %d: is working...\n", i)
    time.Sleep(1 * time.Second)
    fmt.Printf("worker %d: works done\n", i)
}

type signal struct{}
func spawnGroup(f func(i int), num int, groupSignal <-chan signal) <-chan signal {
    c := make(chan signal)
    var wg sync.WaitGroup

    for i := 0; i < num; i++ {
        wg.Add(1)
        go func(i int) {
            <-groupSignal
            fmt.Printf("worker %d: start to work...\n", i)
            f(i)
            wg.Done()
        }(i + 1)
    }

    go func() {
        wg.Wait()
        c <- signal{}
    }()
    return c
}

func main() {
    fmt.Println("start a group of workers...")
    groupSignal := make(chan signal)
    c := spawnGroup(worker, 5, groupSignal)
    time.Sleep(5 * time.Second)
    fmt.Println("the group of workers start to work...")
    close(groupSignal)
    <-c
    fmt.Println("the group of workers work done!")
}
```

main goroutine 创建了一组 5 个 worker goroutine，这些 Goroutine 启动后会阻塞在名为 groupSignal 的无缓冲 channel 上。main goroutine 通过close(groupSignal)向所有 worker goroutine 广播“开始工作”的信号，收到 groupSignal 后，所有 worker goroutine 会“同时”开始工作，就像起跑线上的运动员听到了裁判员发出的起跑信号枪声。

**用于替代锁机制**

无缓冲 channel 具有同步特性，这让它在某些场合可以替代锁，让我们的程序更加清晰

```go
type counter struct {
    c chan int
    i int
}

//通过 创建channel的结构体cter 在单独的goroutine中
//执行 计数++操作，并将计数后的结果写入cter中的channel
func NewCounter() *counter {
    cter := &counter{
        c: make(chan int),
    }
    go func() {
        for {
            cter.i++
            cter.c <- cter.i
        }
    }()
    return cter
}

func (cter *counter) Increase() int {
    return <-cter.c
}

func main() {
    cter := NewCounter()
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(i int) {
            v := cter.Increase() //实际上是从cter结构体中的channel来读取 计数后的结果
            fmt.Printf("goroutine-%d: current counter value is %d\n", i, v)
            wg.Done()
        }(i)
    }
    wg.Wait()
}
```

将计数器操作全部交给一个独立的 Goroutine 去处理,通过无缓冲 channel 的同步阻塞特性,Goroutine 通过 Increase 函数试图增加计数器值的动作，实质上就转化为了一次无缓冲 channel 的接收动作

### 带缓冲 channel 的惯用法 ###

**用作消息队列**

异步收发的带缓冲 channel 更适合被用作为消息队列，并且，带缓冲 channel 在数据收发的性能上要明显好于无缓冲 channel。

### 用作计数信号量（counting semaphore） ###

**带缓冲 channel 中的当前数据个数代表的是，当前同时处于活动状态（处理业务）的 Goroutine 的数量，而带缓冲 channel 的容量（capacity），就代表了允许同时处于活动状态的 Goroutine 的最大数量**。向带缓冲 channel 的一个发送操作表示获取一个信号量，而从 channel 的一个接收操作则表示释放一个信号量。

```go
var active = make(chan struct{}, 3)
var jobs = make(chan int, 10)

func main() {
    go func() {
        for i := 0; i < 8; i++ {
            jobs <- (i + 1)
        }
        close(jobs)
    }()

    var wg sync.WaitGroup

    for j := range jobs {
        wg.Add(1)
        go func(j int) {
            active <- struct{}{} //向active channel 里写数据，获取信号
            log.Printf("handle job: %d\n", j) //比如正在 处理job里的 event
            time.Sleep(2 * time.Second)
            <-active //从 active channel 里读数据  释放信号
            wg.Done()
        }(j)
    }
    wg.Wait()
}
```

这个示例创建了一组 Goroutine 来处理 job，同一时间允许最多 3 个 Goroutine 处于活动状态。

为了达成这一目标，我们看到这个示例使用了一个容量（capacity）为 3 的带缓冲 channel: active 作为计数信号量，这意味着允许同时处于**活动状态**的最大 Goroutine 数量为 3

### len(channel) 的应用 ###

针对 channel ch 的类型不同，len(ch) 有如下两种语义：

* 当 ch 为无缓冲 channel 时，len(ch) 总是返回 0；
* 当 ch 为带缓冲 channel 时，len(ch) 返回当前 channel ch 中尚未被读取的元素个数。

channel 原语用于多个 Goroutine 间的通信，一旦多个 Goroutine 共同对 channel 进行收发操作，**len(channel) 就会在多个 Goroutine 间形成“竞态”**

Goroutine1 使用 len(channel) 判空后，就会尝试从 channel 中接收数据。**但在它真正从 channel 读数据之前，另外一个 Goroutine2 已经将数据读了出去，所以，Goroutine1 后面的读取就会阻塞在 channel 上，导致后面逻辑的失效。**

因此，为了不阻塞在 channel 上，常见的方法是将“判空与读取”放在一个“事务”中，将“判满与写入”放在一个“事务”中，而这类“事务”我们可以通过 select 实现。

```go
func producer(c chan<- int) {
    var i int = 1
    for {
        time.Sleep(2 * time.Second)
        ok := trySend(c, i)
        if ok {
            fmt.Printf("[producer]: send [%d] to channel\n", i)
            i++
            continue
        }
        fmt.Printf("[producer]: try send [%d], but channel is full\n", i)
    }
}

func tryRecv(c <-chan int) (int, bool) {
    select {
    case i := <-c:
        return i, true

    default:
        return 0, false
    }
}

func trySend(c chan<- int, i int) bool {
    select {
    case c <- i:
        return true
    default:
        return false
    }
}

func consumer(c <-chan int) {
    for {
        i, ok := tryRecv(c)
        if !ok {
            fmt.Println("[consumer]: try to recv from channel, but the channel is empty")
            time.Sleep(1 * time.Second)
            continue
        }
        fmt.Printf("[consumer]: recv [%d] from channel\n", i)
        if i >= 3 {
            fmt.Println("[consumer]: exit")
            return
        }
    }
}

func main() {
    var wg sync.WaitGroup
    c := make(chan int, 3)
    wg.Add(2)
    go func() {
        producer(c)
        wg.Done()
    }()

    go func() {
        consumer(c)
        wg.Done()
    }()

    wg.Wait()
}
```

由于用到了 select 原语的 default 分支语义，当 channel 空的时候，tryRecv 不会阻塞；当 channel 满的时候，trySend 也不会阻塞。

这种方法适用于大多数场合，但是这种方法有一个“问题”，**那就是它改变了 channel 的状态，会让 channel 接收了一个元素或发送一个元素到 channel。**

但是在特定的场景下，我们可以**用 len(channel) 来实现**

我们想在不改变 channel 状态的前提下，单纯地侦测 channel 的状态，而又不会因 channel 满或空阻塞在 channel 上。但很遗憾，目前没有一种方法可以在实现这样的功能的同时，适用于所有场合。

情景 (a) 是一个“多发送单接收”的场景，也就是有多个发送者，但**有且只有一个接收者**。在这样的场景下，我们可以在接收 goroutine 中使用 len(channel)是否大于0来判断是否 channel 中有数据需要接收。

情景 (b) 呢，是一个“多接收单发送”的场景，也就是有多个接收者，但有且只有一个发送者。在这样的场景下，我们可以在发送 Goroutine 中使用 len(channel)是否小于 cap(channel)来判断是否可以执行向 channel 的发送操作。

### 与 select 结合使用的一些惯用法 ###

* **利用 default 分支避免阻塞**

select 语句的 default 分支的语义，就是在其他非 default 分支因通信未就绪，而无法被选择的时候执行的，**这就给 default 分支赋予了一种“避免阻塞”的特性。**

* **第二种用法：实现超时机制**

通过超时事件，我们既可以避免长期陷入某种操作的等待中，也可以做一些异常处理工作。

下面示例代码实现了一次具有 30s 超时的 select：

```go
func worker() {
  select {
  case <-c:
       // ... do some stuff
  case <-time.After(30 *time.Second):
      return
  }
}
//time.After(30 *time.Second) 将返回一个只读的 channel，并且内部注册定时器在 30s 后往这个 channel 中写一条数据。通过定时器+channel 实现超时机制
```

* **第三种用法：实现心跳机制**

结合 time 包的 Ticker，我们可以实现带有心跳机制的 select。这种机制让我们可以在监听 channel 的同时，执行一些周期性的任务

```go
func worker() {
  heartbeat := time.NewTicker(30 * time.Second)
  defer heartbeat.Stop()
  for {
    select {
    case <-c:
      // ... do some stuff
    case <- heartbeat.C:
      //... do heartbeat stuff
    }
  }
}
```

这里我们使用 time.NewTicker，创建了一个 Ticker 类型实例 heartbeat。这个实例包含一个 channel 类型的字段 C，这个字段会按一定时间间隔持续产生事件，就像“心跳”一样。这样 for 循环在 channel c 无数据接收时，会每隔特定时间完成一次迭代，然后回到 for 循环进行下一次迭代。









