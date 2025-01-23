launch、async 可以创建、启动新的协程

我们常说 Java 线程的源代码是 Thread.java，这样说虽然不一定准确，但我们起码能看到几个暴露出来的方法。那么，在 Kotlin 协程当中，有没有类似 Coroutine.kt 的类呢



### 协程启动的基础 API ###

```kotlin
// 代码段2

public fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit> =
    SafeContinuation(createCoroutineUnintercepted(completion).intercepted(), COROUTINE_SUSPENDED)

public fun <T> (suspend () -> T).startCoroutine(
    completion: Continuation<T>
) {
    createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

启动协程有三种常见的方式：launch、runBlocking、async。它们其实属于协程中间层提供的 API，**而它们的底层都在某种程度上调用了“基础层”的协程 API。**

createCoroutine{}、startCoroutine{}，它们都是扩展函数，其扩展接收者类型是一个函数类型：suspend () -> T，代表了“无参数，返回值为 T 的挂起函数或者 Lambda”

```kotlin
// 代码段3

fun main() {
    testStartCoroutine()
    Thread.sleep(2000L)
}

val block = suspend {
    println("Hello!")
    delay(1000L)
    println("World!")
    "Result"
}

private fun testStartCoroutine() {

    val continuation = object : Continuation<String> {
        override val context: CoroutineContext
            get() = EmptyCoroutineContext

        override fun resumeWith(result: Result<String>) {
            println("Result is: ${result.getOrNull()}")
        }
    }

    block.startCoroutine(continuation)
}

/*
输出结果
Hello!
World!
Result is: Result
*/
```

Continuation 主要有两种用法: 

1. 一种是在实现挂起函数的时候，用于传递挂起函数的执行结果
2. 另一种是在调用挂起函数的时候，以匿名内部类的方式，用于接收挂起函数的执行结果

startCoroutine() 的作用其实就是创建一个新的协程，并且执行 block 当中的逻辑，**等协程执行完毕以后，将结果返回给 Continuation 对象**

```kotlin
代码段4

private fun testCreateCoroutine() {

    val continuation = object : Continuation<String> {
        override val context: CoroutineContext
            get() = EmptyCoroutineContext

        override fun resumeWith(result: Result<String>) {
            println("Result is: ${result.getOrNull()}")
        }
    }

    val coroutine = block.createCoroutine(continuation)

    coroutine.resume(Unit)
}

/*
输出结果
Hello!
World!
Result is: Result
*/
```

createCoroutine() 的作用其实就是创建一个协程，并暂时先不启动它。等我们想要启动它的时候，直接调用 resume() 即可。

我们就以 startCoroutine() 为例，来研究下它的实现原理。我们把代码段  反编译成 Java，看看它会变成什么样子：

```kotlin
// 代码段5

public final class LaunchUnderTheHoodKt {
    // 1
    public static final void main() {
        testStartCoroutine();
        Thread.sleep(2000L);
    }

    // 2
    private static final Function1<Continuation<? super String>, Object> block = new LaunchUnderTheHoodKt$block$1(null);

    // 3
    public static final Function1<Continuation<? super String>, Object> getBlock() {
        return block;
    }
    // 4
    static final class LaunchUnderTheHoodKt$block$1 extends SuspendLambda implements Function1<Continuation<? super String>, Object> {
        int label;

        LaunchUnderTheHoodKt$block$1(Continuation $completion) {
          super(1, $completion);
        }

        @Nullable
        public final Object invokeSuspend(@NotNull Object $result) {
          Object object = IntrinsicsKt.getCOROUTINE_SUSPENDED();
          switch (this.label) {
            case 0:
              ResultKt.throwOnFailure(SYNTHETIC_LOCAL_VARIABLE_1);
              System.out
                .println("Hello!");
              this.label = 1;
              if (DelayKt.delay(1000L, (Continuation)this) == object)
                return object; 
              DelayKt.delay(1000L, (Continuation)this);
              System.out
                .println("World!");
              return "Result";
            case 1:
              ResultKt.throwOnFailure(SYNTHETIC_LOCAL_VARIABLE_1);
              System.out.println("World!");
              return "Result";
          } 
          throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
        }

        @NotNull
        public final Continuation<Unit> create(@NotNull Continuation<? super LaunchUnderTheHoodKt$block$1> $completion) {
          return (Continuation<Unit>)new LaunchUnderTheHoodKt$block$1($completion);
        }

        @Nullable
        public final Object invoke(@Nullable Continuation<?> p1) {
          return ((LaunchUnderTheHoodKt$block$1)create(p1)).invokeSuspend(Unit.INSTANCE);
        }
    }

    // 5
    private static final void testStartCoroutine() {
        LaunchUnderTheHoodKt$testStartCoroutine$continuation$1 continuation = new LaunchUnderTheHoodKt$testStartCoroutine$continuation$1();
        ContinuationKt.startCoroutine(block, continuation);
    }

    // 6
    public static final class LaunchUnderTheHoodKt$testStartCoroutine$continuation$1 implements Continuation<String> {
        @NotNull
        public CoroutineContext getContext() {
          return (CoroutineContext)EmptyCoroutineContext.INSTANCE;
        }

        public void resumeWith(@NotNull Object result) {
          System.out.println(Intrinsics.stringPlus("Result is: ", Result.isFailure-impl(result) ? null : result));
        }
    }
}


internal abstract class SuspendLambda(
    public override val arity: Int,
    completion: Continuation<Any?>?
) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {}
```

* 注释 1，是我们的 main() 函数。由于它本身只是一个普通的函数，因此反编译之后，逻辑并没有什么变化。
* 注释 2、3，它们是 Kotlin 为 block 变量生成的静态变量以及方法。
* 注释 4，LaunchUnderTheHoodKt$block$1，其实就是 block 具体的实现类。这个类继承自 SuspendLambda，而 SuspendLambda 是 ContinuationImpl 的子类，因此它也间接实现了 Continuation 接口。**其中的 invokeSuspend()，也就是分析过的协程状态机逻辑**
* 注释 5，它对应了 testStartCoroutine() 这个方法，原本的 block.startCoroutine(continuation) 变成了“ContinuationKt.startCoroutine(block, continuation)”，这其实就体现出了扩展函数的原理。
* 注释 6，其实就是 continuation 变量对应的匿名内部类。

首先，main() 函数会调用 testStartCoroutine() 函数，接着，就会调用 startCoroutine() 方法。

```kotlin
// 代码段6

public fun <T> (suspend () -> T).startCoroutine(
    completion: Continuation<T>
) {
//        注意这里
//           ↓
createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

从代码段 6 里，我们可以看到，在 startCoroutine() 当中，首先会调用 createCoroutineUnintercepted() 方法。如果我们直接去看它的源代码，会发现它只存在一个声明，并没有具体实现：

```kotlin
// 代码段7

//    注意这里
//       ↓
public expect fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit>
```

expect，我们可以把它理解为一种声明，由于 Kotlin 是面向多个平台的，具体的实现，就需要在特定的平台实现。所以在这里，我们就需要打开 Kotlin 的源代码，找到 JVM 平台对应的实现：

```kotlin
// 代码段8

//    1，注意这里
//       ↓
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    //             2，注意这里
    //               ↓
    return if (this is BaseContinuationImpl)
        create(probeCompletion)
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}
```

这个 actual，代表了 createCoroutineUnintercepted() 在 JVM 平台的实现。

createCoroutineUnintercepted() 仍然还是一个扩展函数，注释 2 处的 this，其实就代表了前面代码段 3 当中的 block 变量。我们结合代码段 5 反编译出来的 `LaunchUnderTheHoodKt$block$1`，可以知道 block 其实就是 SuspendLambda 的子类，而 SuspendLambda 则是 ContinuationImpl 的子类

注释 2 处的 (this is BaseContinuationImpl) 条件一定是为 true 的。这时候，它就会调用 create(probeCompletion)。

```kotlin
// 代码段9

public open fun create(completion: Continuation<*>): Continuation<Unit> {
    throw UnsupportedOperationException("create(Continuation) has not been overridden")
}
```

那么，create() 方法是在哪里被重写的呢？答案其实就在代码段 5 的“LaunchUnderTheHoodKt$block$1”这个 block 的实现类当中

```kotlin
// 代码段10

static final class LaunchUnderTheHoodKt$block$1 extends SuspendLambda implements Function1<Continuation<? super String>, Object> {
    int label;

    LaunchUnderTheHoodKt$block$1(Continuation $completion) {
      super(1, $completion);
    }

    @Nullable
    public final Object invokeSuspend(@NotNull Object $result) {
      Object object = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (this.label) {
        case 0:
          ResultKt.throwOnFailure(SYNTHETIC_LOCAL_VARIABLE_1);
          System.out
            .println("Hello!");
          this.label = 1;
          if (DelayKt.delay(1000L, (Continuation)this) == object)
            return object; 
          DelayKt.delay(1000L, (Continuation)this);
          System.out
            .println("World!");
          return "Result";
        case 1:
          ResultKt.throwOnFailure(SYNTHETIC_LOCAL_VARIABLE_1);
          System.out.println("World!");
          return "Result";
      } 
      throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }

    // 1，注意这里
    public final Continuation<Unit> create(@NotNull Continuation<? super LaunchUnderTheHoodKt$block$1> $completion) {
      return (Continuation<Unit>)new LaunchUnderTheHoodKt$block$1($completion);
    }

    @Nullable
    public final Object invoke(@Nullable Continuation<?> p1) {
      return ((LaunchUnderTheHoodKt$block$1)create(p1)).invokeSuspend(Unit.INSTANCE);
    }
}
```

代码段 8 当中的 ``create(probeCompletion)``，最终会调用代码段 10 的 create() 方法，它最终会返回“LaunchUnderTheHoodKt$block$1”这个 block 实现类，对应的 Continuation 对象。

这行代码，**其实就对应着协程被创建的时刻。**

协程创建的逻辑就分析完了，我们再回到 startCoroutine() 的源码，看看它后续的逻辑。

```kotlin
// 代码段11

public fun <T> (suspend () -> T).startCoroutine(
    completion: Continuation<T>
) {
//                                           注意这里
//                                             ↓
createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

```kotlin
// 代码段12

public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
```

只是将 Continuation 强转成了 ContinuationImpl，调用了它的 intercepted()。这里有个细节，**由于 this 的类型是“LaunchUnderTheHoodKt$block$1”，它是 ContinuationImpl 的子类，所以这个类型转换一定可以成功**。

我们看看 ContinuationImpl 的源代码。

```kotlin
// 代码段13

internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {

    @Transient
    private var intercepted: Continuation<Any?>? = null

    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
}
```

这里其实就是通过 ContinuationInterceptor，对 Continuation 进行拦截，从而将程序的执行逻辑派发到特定的线程之上

回到 startCoroutine() 的源码，看看它的最后一步 ``resume(Unit)``。

```kotlin
// 代码段14

public fun <T> (suspend () -> T).startCoroutine(
    completion: Continuation<T>
) {
//                                                   注意这里
//                                                      ↓
createCoroutineUnintercepted(completion).intercepted().resume(Unit)
}
```

这里的`` resume(Unit)``，作用其实就相当于启动了协程

### launch 是如何启动协程的 ###

```go
// 代码段15

fun main() {
    testLaunch()
    Thread.sleep(2000L)
}

private fun testLaunch() {
    val scope = CoroutineScope(Job())
    scope.launch {
        println("Hello!")
        delay(1000L)
        println("World!")
    }
}

/*
输出结果：
Hello!
World!
*/
```

还是通过反编译，来看看它对应的 Java 代码长什么样

```go
// 代码段16

public final class LaunchUnderTheHoodKt {
  public static final void main() {
    testLaunch();
    Thread.sleep(2000L);
  }

  private static final void testLaunch() {
    CoroutineScope scope = CoroutineScopeKt.CoroutineScope((CoroutineContext)JobKt.Job$default(null, 1, null));
    BuildersKt.launch$default(scope, null, null, new LaunchUnderTheHoodKt$testLaunch$1(null), 3, null);
  }

  static final class LaunchUnderTheHoodKt$testLaunch$1 extends SuspendLambda implements Function2<CoroutineScope, Continuation<? super Unit>, Object> {
    int label;

    LaunchUnderTheHoodKt$testLaunch$1(Continuation $completion) {
      super(2, $completion);
    }

    @Nullable
    public final Object invokeSuspend(@NotNull Object $result) {
      Object object = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (this.label) {
        case 0:
          ResultKt.throwOnFailure(SYNTHETIC_LOCAL_VARIABLE_1);
          System.out
            .println("Hello!");
          this.label = 1;
          if (DelayKt.delay(1000L, (Continuation)this) == object)
            return object; 
          DelayKt.delay(1000L, (Continuation)this);
          System.out
            .println("World!");
          return Unit.INSTANCE;
        case 1:
          ResultKt.throwOnFailure(SYNTHETIC_LOCAL_VARIABLE_1);
          System.out.println("World!");
          return Unit.INSTANCE;
      } 
      throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }

    @NotNull
    public final Continuation<Unit> create(@Nullable Object value, @NotNull Continuation<? super LaunchUnderTheHoodKt$testLaunch$1> $completion) {
      return (Continuation<Unit>)new LaunchUnderTheHoodKt$testLaunch$1($completion);
    }

    @Nullable
    public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation<?> p2) {
      return ((LaunchUnderTheHoodKt$testLaunch$1)create(p1, p2)).invokeSuspend(Unit.INSTANCE);
    }
  }
}
```

* “LaunchUnderTheHoodKt$testLaunch$1”这个类，它其实对应的就是我们 launch 当中的 Lambda。

为了让它们之间的对应关系更加明显，可以换一种写法：

```go
// 代码段17

private fun testLaunch() {
    val scope = CoroutineScope(Job())
    val block: suspend CoroutineScope.() -> Unit = {
        println("Hello!")
        delay(1000L)
        println("World!")
    }
    scope.launch(block = block)
}
```

接下来，我们来看看 launch{} 的源代码

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    // 1
    val newContext = newCoroutineContext(context)
    // 2
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    // 3
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

* 注释 1，launch 会根据传入的 CoroutineContext 创建出新的 Context。
* 注释 2，launch 会根据传入的启动模式来创建对应的协程对象。这里有两种，一种是标准的，一种是懒加载的。
* 注释 3，尝试启动协程。

coroutine.start() 这个方法，会进入 AbstractCoroutine 这个抽象类

```kotlin
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {

    // 省略

    public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }
}
```

```kotlin
public enum class CoroutineStart {
    public operator fun <T> invoke(block: suspend () -> T, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(completion)
            ATOMIC -> block.startCoroutine(completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(completion)
            LAZY -> Unit // will start lazily
        }
}
```

在这个 invoke() 方法当中，它会根据 launch 传入的启动模式，以不同的方式启动协程。当我们的启动模式是 ATOMIC 的时候，就会调用 ```block.startCoroutine(completion)。```

而这个，其实就是研究过的``` startCoroutine() ```这个协程基础 API。

由于我们没有传入特定的启动模式，因此，这里会执行默认的模式，也就是调用“startCoroutineCancellable(completion)”这个方法。

```kotlin
public fun <T> (suspend () -> T).startCoroutineCancellable(completion: Continuation<T>): Unit = runSafely(completion) {
    // 1
    createCoroutineUnintercepted(completion).intercepted().resumeCancellableWith(Result.success(Unit))
}

public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)

    return if (this is BaseContinuationImpl)
        // 2
        create(probeCompletion)
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}
```











