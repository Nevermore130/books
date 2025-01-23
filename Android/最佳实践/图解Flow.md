#### Flow 为什么是冷的？

```kotlin
// 代码段1

fun main() {
    val scope = CoroutineScope(Job())
    scope.launch {
        testFlow()
    }

    Thread.sleep(1000L)

    logX("end")
}

private suspend fun testFlow() {
    // 1
    flow {
        emit(1)
        emit(2)
        emit(3)
        emit(4)
        emit(5)
    }.collect {      // 2
            logX(it)
        }
}

/**
 * 控制台输出带协程信息的log
 */
fun logX(any: Any?) {
    println(
        """
================================
$any
Thread:${Thread.currentThread().name}
================================""".trimIndent()
    )
}


```

创建了一个 CoroutineScope，接着使用它创建了一个新的协程，在协程当中，我们使用 flow{} 这个高阶函数创建了 Flow 对象，接着使用了 collect{} 这个终止操作符。

分析一下 Flow 是怎么创建出来的

```kotlin
// 代码段2

public fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> =
     SafeFlow(block)

public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}
```

* flow{} 是一个高阶函数，它接收的参数类型是函数类型``FlowCollector.() -> Unit``，这个类型代表了：它是 FlowCollector 的扩展或成员方法，没有参数，也没有返回值
* flow() 的返回值类型是Flow，而它实际返回的类型是 SafeFlow

```kotlin
// 代码段3

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    // 1
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}

public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {
    // 省略
}

internal interface CancellableFlow<out T> : Flow<T>
```

* SafeFlow 其实是 AbstractFlow 的子类，而 AbstractFlow 则实现了 Flow 这个接口，所以 SafeFlow 算是间接实现了 Flow 接口
* 而 AbstractFlow 是协程当中所有 Flow 的抽象类，所以，它当中应该会有许多 Flow 通用的逻辑。

```kotlin
// 代码段4

public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {

    // 1
    public final override suspend fun collect(collector: FlowCollector<T>) {
        // 2
        val safeCollector = SafeCollector(collector, coroutineContext)
        try {
            // 3
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }

    public abstract suspend fun collectSafely(collector: FlowCollector<T>)
}
```

看这个挂起函数 collect()，它其实就是终止操作符 collect 对应的调用处

* 注释 2，collect() 的参数类型是 FlowCollector，这里只是将其重新封装了一遍，变成了 SafeColletor 对象。从它的名称，我们也大概可以猜出来，它肯定是会对 collect 当中的逻辑做一些安全检查的
* 注释 3，collectSafely()，这里其实就是调用了它的抽象方法，**而它的具体实现就在代码段 3 里 SafeFlow 的 collectSafely() 方法，而它的逻辑也很简单，它直接调用了 collector.block()**，这其实就相当于触发了 flow{} 当中的 Lambda 逻辑

FLow 之所以是冷的，是因为 Flow 的构造器，真的就只会构造一个 SafeFlow 对象，完全不会触发执行它内部的 Lambda 表达式的逻辑，**只有当 collect() 被调用之后，flow{} 当中的 Lambda 逻辑才会真正被触发执行。**



#### FlowCollector：上游与下游之间的桥梁

下游的 collect() 会触发上游的 Lambda 执行，上游的 Lambda 当中的 emit() 会把数据传递给下游

```kotlin
// 代码段5

public fun interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}

public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}
```

当我们在下游调用 collect{} 的时候，其实是在调用 Flow 接口的 collect 方法,为了让它们的关系更加清晰地暴露出来，我们可以换一种写法，来实现代码段 1 当中的逻辑。

```kotlin
// 代码段6

private suspend fun testFlow() {

    flow {
        // 1
        emit(1)
        emit(2)
        emit(3)
        emit(4)
        emit(5)
    }
        // 变化在这里
        .collect(object : FlowCollector<Int>{ 
            // 2
            override suspend fun emit(value: Int) {
                logX(value)
            }
        })
}
```

在这个匿名内部类的 emit() 方法，其实就充当着 Flow 的下游接收其中的数据流。

要分析“上游与下游是如何连接的”这个问题，我们只需要看注释 2 处的 emit() 是如何被调用的即可。

collect() 方法传入的 FlowCollector 参数，其实是被传入 SafeCollector 当中，被封装了起来

```val safeCollector = SafeCollector(collector, coroutineContext)```

```kotlin
// 代码段7

internal actual class SafeCollector<T> actual constructor(
    // 1
    @JvmField internal actual val collector: FlowCollector<T>,
    @JvmField internal actual val collectContext: CoroutineContext
) : FlowCollector<T>, ContinuationImpl(NoOpContinuation, EmptyCoroutineContext), CoroutineStackFrame {

    internal actual val collectContextSize = collectContext.fold(0) { count, _ -> count + 1 }
    private var lastEmissionContext: CoroutineContext? = null
    private var completion: Continuation<Unit>? = null

    // ContinuationImpl
    override val context: CoroutineContext
        get() = completion?.context ?: EmptyCoroutineContext

    // 2
    override suspend fun emit(value: T) {
        return suspendCoroutineUninterceptedOrReturn sc@{ uCont ->
            try {
                // 3
                emit(uCont, value)
            } catch (e: Throwable) {
                lastEmissionContext = DownstreamExceptionElement(e)
                throw e
            }
        }
    }

    private fun emit(uCont: Continuation<Unit>, value: T): Any? {
        val currentContext = uCont.context
        currentContext.ensureActive()

        // 4
        val previousContext = lastEmissionContext
        if (previousContext !== currentContext) {
            checkContext(currentContext, previousContext, value)
        }
        completion = uCont
        // 5
        return emitFun(collector as FlowCollector<Any?>, value, this as Continuation<Unit>)
    }

}

// 6
private val emitFun =
    FlowCollector<Any?>::emit as Function3<FlowCollector<Any?>, Any?, Continuation<Unit>, Any?>

public interface Function3<in P1, in P2, in P3, out R> : Function<R> {
    public operator fun invoke(p1: P1, p2: P2, p3: P3): R
}
```

* 注释 1，collector，它是 SafeCollector 的参数,对应匿名内部类 FlowCollector
* 注释 2，emit()，Flow 上游发送的数据，最终会传递到这个 emit() 方法当中来。我们可以将其看作上游的 emit()。
* 注释 3，emit(uCont, value)，这里的 suspendCoroutineUninterceptedOrReturn 这个高阶函数，是把挂起函数的 Continuation 暴露了出来，并且将其作为参数传递给了另一个 emit() 方法。
* 注释 4，这里会对当前的协程上下文与之前的协程上下文做对比检查，如果它们两者不一致，就会在 checkContext() 当中做进一步的判断和提示
* emitFun(collector as FlowCollector, value, this as Continuation)，这里其实就是在调用下游的 emit()，也就是代码段 6 当中的注释 2 对应的 emit() 方法

这里的 emitFun() 是什么呢？我们可以在注释 6 处找到它的定义：FlowCollector::emit，这是函数引用的语法，代表了它就是 FlowCollector 的 emit() 方法，它的类型是``Function3, Any?, Continuation, Any?>。``

#### flow流程总结

1. flow {} 中的lambda表达式被 转为 FlowCollector的扩展函数并传入flow函数``flow(block: suspend FlowCollector.() -> Unit): Flow``
2. flow函数返回SafeFlow对象，SafeFlow对象继承AbstractFlow
3. SafeFlow调用collect函数，即调用AbstractFlow的collect函数
4. collect函数接收一个FlowCollector的匿名对象collector
5. collector 包装成SafeCollector对象并传给``collectSafely(safeCollector)`` 函数
6. collectSafely 是SafeFlow实现的，即调用``safeCollector.block()``
7. ``safeCollector.block()`` 会执行flow{}中传入的扩展函数，即执行``emit(1)`` 即**上游数据发送**
8. ``emit(1)`` 即执行 SafeCollector 对象的emit函数
9. emit函数 最终会执行 匿名对象collector 中的emit(value) 函数，**即下游接收**

