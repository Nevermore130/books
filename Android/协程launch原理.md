协程到底是如何创建的？它对应的源代码，到底在哪个类？具体在哪一行？

#### 协程启动的基础 API ####

在Continuation.kt这个文件当中，还有两个重要的扩展函数：

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

启动协程有三种常见的方式：launch、runBlocking、async。它们其实属于协程中间层提供的 API，而它们的底层都在某种程度上调用了“基础层”的协程 API

createCoroutine{}、startCoroutine{}，它们都是扩展函数，其扩展接收者类型是一个函数类型：``suspend () -> T``，代表了“无参数，返回值为 T 的挂起函数或者 Lambda”

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

在这段代码中，我们定义了一个 Lambda 表达式 block，它的类型就是 ``suspend () -> T``。这样一来，我们就可以用 block.startCoroutine() 来启动协程了。这里，我们还创建了一个匿名内部类对象 continuation，作为 startCoroutine() 的参数。

从代码段 3 的执行结果中，我们可以看出来，**startCoroutine() 的作用其实就是创建一个新的协程**，**并且执行 block 当中的逻辑，等协程执行完毕以后，将结果返回给 Continuation 对象**。而这个逻辑，我们使用 createCoroutine() 这个方法其实也可以实现。

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

createCoroutine() 的作用其实就是创建一个协程，并暂时先不启动它。等我们想要启动它的时候，直接调用 resume() 即可

就以 startCoroutine() 为例，来研究下它的实现原理。我们把代码段 3 反编译成 Java，看看它会变成什么样子：

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
* 注释 4，``LaunchUnderTheHoodKt$block$​1``，其实就是 block 具体的实现类。这个类继承自 SuspendLambda，而 SuspendLambda 是 ContinuationImpl 的子类，因此它也间接实现了 Continuation 接口。其中的 invokeSuspend()，也就是协程状态机逻辑。除此之外，它还有一个 create() 方法，我们在后面会来分析它
* 注释 5，它对应了 testStartCoroutine() 这个方法
* 注释 6，其实就是 continuation 变量对应的匿名内部类

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

上面代码中的 expect，我们可以把它理解为一种声明，由于 Kotlin 是面向多个平台的，具体的实现，就需要在特定的平台实现。所以在这里，我们就需要打开 Kotlin 的源代码，找到 JVM 平台对应的实现：

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

这里的注释 1，这个 actual，代表了 createCoroutineUnintercepted() 在 JVM 平台的实现。

另外，**createCoroutineUnintercepted() 仍然还是一个扩展函数**，**注释 2 处的 this，其实就代表了前面代码段 3 当中的 block 变量**。我们结合代码段 5 反编译出来的 LaunchUnderTheHoodKt$block$1，可以知道 block 其实就是 SuspendLambda 的子类，而 SuspendLambda 则是 ContinuationImpl 的子类。

因此，注释 2 处的 ``(this is BaseContinuationImpl) ``条件一定是为 true 的。这时候，它就会调用 ``create(probeCompletion)``。

如果你去查看 create() 的源代码，会看到这样的代码：

```kotlin
// 代码段9

public open fun create(completion: Continuation<*>): Continuation<Unit> {
    throw UnsupportedOperationException("create(Continuation) has not been overridden")
}
```

潜台词就是，create() 方法应该被重写！如果不被重写，就会抛出异常

那么，create() 方法是在哪里被重写的呢？答案其实就在代码段 5 的“``LaunchUnderTheHoodKt$block$1``”这个 block 的实现类当中

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

**这行代码，其实就对应着协程被创建的时刻。**

再回到 startCoroutine() 的源码，看看它后续的逻辑。

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

类似的，intercepted() 这个方法的源代码，我们也需要去 Kotlin 的源代码当中找到对应的 JVM 实现。

```kotlin
// 代码段12

public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
```

它的逻辑很简单，只是将 Continuation 强转成了 ContinuationImpl，调用了它的 intercepted()。这里有个细节，由于 this 的类型是“``LaunchUnderTheHoodKt$block$1``”，它是 ContinuationImpl 的子类，所以这个类型转换一定可以成功。

看看 ContinuationImpl 的源代码。

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

回到 startCoroutine() 的源码，看看它的最后一步 resume(Unit)。

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

这里的 resume(Unit)，作用其实就相当于启动了协程。

好，现在我们已经弄清楚了 startCoroutine() 这个协程的基础 API 是如何启动协程的了。接下来，我们来看看中间层的 launch{} 函数是如何启动协程的。



#### launch 是如何启动协程的？ ####

```kotlin
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

通过反编译，来看看它对应的 Java 代码长什么样：

```kotlin
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



注意的是“``LaunchUnderTheHoodKt$testLaunch$1``”这个类，它其实对应的就是我们 launch 当中的 Lambda。

我们来看看 launch{} 的源代码。

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
* 注释 3，尝试启动协程

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

继续跟进 start(block, receiver, this)，就会进入 CoroutineStart.invoke()。

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

在这个 invoke() 方法当中，它会根据 launch 传入的启动模式，以不同的方式启动协程。当我们的启动模式是 ATOMIC 的时候，就会调用 ``block.startCoroutine(completion)``。而这个，其实就是 startCoroutine() 这个协程基础 API。

对于代码段 15 的 launch 逻辑而言，由于我们没有传入特定的启动模式，因此，这里会执行默认的模式，也就是调用“``startCoroutineCancellable(completion)``”这个方法。

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

通过查看 startCoroutineCancellable() 的源代码，我们能发现，它最终还是会调用我们之前分析过的 createCoroutineUnintercepted()，而在它的内部，仍然会像我们之前分析过的，去调用 create(probeCompletion)













