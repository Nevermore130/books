goroutine 在读取一个变量的值的时候，能够看到其它 goroutine 对这个变量进行的写的结果。

由于 CPU 指令重排和多级 Cache 的存在，保证多核访问同一个变量这件事儿变得非常复杂。毕竟，不同 CPU 架构（x86/amd64、ARM、Power 等）的处理方式也不一样，再加上编译器的优化也可能对指令进行重排，**所以编程语言需要一个规范，来明确多线程同时访问同一个变量的可见性和顺序**

* 向广大的程序员提供一种保证，以便他们在做设计和开发程序时，面对同一个数据同时被多个 goroutine 访问的情况，可以做一些串行化访问的控制，比如使用 Channel 或者 sync 包和 sync/atomic 包中的并发原语。
* 允许编译器和硬件对程序做一些优化。这一点其实主要是为编译器开发者提供的保证，这样可以方便他们对 Go 的编译器做优化。

### 重排和可见性的问题 ###

由于指令重排，代码并不一定会按照你写的顺序执行。

```go
var a, b int

func f() {
  a = 1 // w之前的写操作
  b = 2 // 写操作w
}

func g() {
  print(b) // 读操作r
  print(a) // ???
}

func main() {
  go f() //g1
  g() //g2
}
```

可以看到，第 9 行是要打印 b 的值。需要注意的是，即使这里打印出的值是 2，但是依然可能在打印 a 的值时，打印出初始值 0，而不是 1。这是因为，**程序运行的时候，不能保证 g2 看到的 a 和 b 的赋值有先后关系。**



### happens-before ###

在一个 goroutine 内部，程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样。

在下面的代码中，即使编译器或者 CPU 对 a、b、c 的初始化进行了重排，但是打印结果依然能保证是 1、2、3

```go
func foo() {
    var a = 1
    var b = 2
    var c = 3

    println(a)
    println(b)
    println(c)
}
```

但是，对于另一个 goroutine 来说，重排却会产生非常大的影响。**因为 Go 只保证 goroutine 内部重排对读写的顺序没有影响**

如果两个 action（read 或者 write）有明确的 happens-before 关系，你就可以确定它们之间的执行顺序

Go 内存模型通过 happens-before 定义两个事件（读、写 action）的顺序：**如果事件 e1 happens before 事件 e2，那么，我们就可以说事件 e2 在事件 e1 之后发生（happens after）。**

我们要保证 r 绝对能观察到 w 操作的结果，那么就需要同时满足两个条件：

* w happens before r；
* 其它对 v 的写操作（w2、w3、w4, ......） 要么 happens before w，要么 happens after r

```
Within a single goroutine, the happens-before order is the order expressed by the program.
```

在单个的 goroutine 内部， happens-before 的关系和代码编写的顺序是一致的。

在 goroutine 内部对一个局部变量 v 的读，一定能观察到最近一次对这个局部变量 v 的写。如果要保证多个 goroutine 之间对一个共享变量的读写顺序，**在 Go 语言中，可以使用并发原语为读写操作建立 happens-before 关系**

### Go 语言中保证的 happens-before 关系 ###

**init 函数**

应用程序的初始化是在单一的 goroutine 执行的。如果包 p 导入了包 q，那么，q 的 init 函数的执行一定 happens before p 的任何初始化代码。

**Channel**

往一个 Channel 中发送一条数据，通常对应着另一个 goroutine 从这个 Channel 中接收一条数据。

第一条规则：往 Channel 中的发送操作，happens before 从该 Channel 接收相应数据的动作完成之前，即第 n 个 send 一定 happens before 第 n 个 receive 的完成。

```go
var ch = make(chan struct{}, 10) // buffered或者unbuffered
var s string

func f() {
  s = "hello, world"
  ch <- struct{}{}
}

func main() {
  go f()
  <-ch
  print(s)
}
```

往 ch 发送数据 happens before 从 ch 中读取出一条数据（第 11 行），第 12 行打印 s 的值 happens after 第 11 行，所以，打印的结果肯定是初始化后的 s 的值“hello world”

第 2 条规则是，close 一个 Channel 的调用，肯定 happens before 从关闭的 Channel 中读取出一个零值

第 3 条规则是，对于 unbuffered 的 Channel，也就是容量是 0 的 Channel，从此 Channel 中读取数据的调用一定 happens before 往此 Channel 发送数据的调用完成

```go
var ch = make(chan int)
var s string

func f() {
  s = "hello, world"
  <-ch
}

func main() {
  go f()
  ch <- struct{}{}
  print(s)
}
```

#### Mutex/RWMutex ####

对于互斥锁 Mutex m 或者读写锁 RWMutex m，有 3 条 happens-before 关系的保证。

第 n 次的 m.Unlock 一定 happens before 第 n+1 m.Lock 方法的返回；

**Once**

f 函数的那个单次调用一定 happens before 任何 once.Do(f) 调用的返回。换句话说，就是函数 f 一定会在 Do 方法返回之前执行。















