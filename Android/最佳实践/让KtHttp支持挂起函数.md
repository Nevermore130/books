那么，异步代码又是长什么样的呢？

```kotlin
private fun testAsync() {
    // 异步代码
    KtHttpV3.create(ApiServiceV3::class.java).repos(
        lang = "Kotlin",
        since = "weekly"
    ).call(object : Callback<RepoList> {
        override fun onSuccess(data: RepoList) {
            println(data)
        }

        override fun onFail(throwable: Throwable) {
            println(throwable)
        }
    })
}
```

首先需要创建一个 Callback 接口，在这个 Callback 当中，我们可以拿到 API 请求的结果。

```kotlin
interface Callback<T: Any> {
    fun onSuccess(data: T)
    fun onFail(throwable: Throwable)
}
```

这里我们运用了空安全思维当中的泛型边界“T: Any”，这样一来，我们就可以保证 T 类型一定是非空的。

还需要一个 KtCall 类，它的作用是承载 Callback，或者说，它是用来调用 Callback 的。

```kotlin
class KtCall<T: Any>(
    private val call: Call,
    private val gson: Gson,
    private val type: Type
) {
    fun call(callback: Callback<T>): Call {
        // TODO
    }
}
```

* OkHttp 的 Call 对象、JSON 解析的 Gson 对象，以及反射类型 Type
* call() 方法，它接收的是前面我们定义的 Callback 对象，返回的是 OkHttp 的 Call 对象。

```kotlin
class KtCall<T: Any>(
    private val call: Call,
    private val gson: Gson,
    private val type: Type
) {
    fun call(callback: Callback<T>): Call {
        // 步骤1， 使用call请求API
        // 步骤2， 根据请求结果，调用callback.onSuccess()或者是callback.onFail()
        // 步骤3， 返回OkHttp的Call对象
    }
}
```

* 为了将请求任务派发到异步线程，我们需要使用 OkHttp 的异步请求方法 enqueue()。
* 根据请求结果，调用 callback.onSuccess() 或者是 callback.onFail()。如果请求成功了，我们在调用 onSuccess() 之前，还需要用 Gson 将请求结果进行解析，然后才返回。

```kotlin
class KtCall<T: Any>(
    private val call: Call,
    private val gson: Gson,
    private val type: Type
) {
    fun call(callback: Callback<T>): Call {
        call.enqueue(object : okhttp3.Callback {
            override fun onFailure(call: Call, e: IOException) {
                callback.onFail(e)
            }

            override fun onResponse(call: Call, response: Response) {
                try { // ①
                    val t = gson.fromJson<T>(response.body?.string(), type)
                    callback.onSuccess(t)
                } catch (e: Exception) {
                    callback.onFail(e)
                }
            }
        })
        return call
    }
}
```

由于 API 返回的结果并不可靠，即使请求成功了，其中的 JSON 数据也不一定合法，所以这里我们一般还需要进行额外的判断。在实际的商业项目当中，我们可能还需要根据当中的状态码，进行进一步区分和封装

```kotlin
interface ApiServiceV3 {
    @GET("/repo")
    fun repos(
        @Field("lang") lang: String,
        @Field("since") since: String
    ): KtCall<RepoList> // ①
}
```

由于 repo() 方法的返回值类型是 KtCall，为了支持这种写法，我们的 invoke 方法就需要跟着做一些小的改动：

```kotlin
// 这里也同样使用了泛型边界
private fun <T: Any> invoke(path: String, method: Method, args: Array<Any>): Any? {
    if (method.parameterAnnotations.size != args.size) return null

    var url = path
    val parameterAnnotations = method.parameterAnnotations
    for (i in parameterAnnotations.indices) {
        for (parameterAnnotation in parameterAnnotations[i]) {
            if (parameterAnnotation is Field) {
                val key = parameterAnnotation.value
                val value = args[i].toString()
                if (!url.contains("?")) {
                    url += "?$key=$value"
                } else {
                    url += "&$key=$value"
                }

            }
        }
    }

    val request = Request.Builder()
        .url(url)
        .build()

    val call = okHttpClient.newCall(request)
    val genericReturnType = getTypeArgument(method)
    
    // 变化在这里
    return KtCall<T>(call, gson, genericReturnType)
}

// 拿到 KtCall<RepoList> 当中的 RepoList类型
private fun getTypeArgument(method: Method) =
    (method.genericReturnType as ParameterizedType).actualTypeArguments[0]
```

在后续调用它的时候，我们就可以这么写了：ktCall.call()。

```kotlin
private fun testAsync() {
    KtHttpV3.create(ApiServiceV3::class.java)
    .repos(
        lang = "Kotlin",
        since = "weekly"
    ).call(object : Callback<RepoList> {
        override fun onSuccess(data: RepoList) {
            println(data)
        }

        override fun onFail(throwable: Throwable) {
            println(throwable)
        }
    })
}
```

由于我们的实际请求已经通过 OkHttp 派发（enqueue）到统一的线程池当中去了，并不会阻塞主线程，所以这样的代码模式执行在 Android、Swing 之类的 UI 编程平台，也不会引起 UI 界面卡死的问题。

为了让 KtHttp 同时支持两种请求方式，我们只需要增加一个 if 判断即可

```kotlin
private fun <T: Any> invoke(path: String, method: Method, args: Array<Any>): Any? {
    // 省略其他代码

    return if (isKtCallReturn(method)) {
        val genericReturnType = getTypeArgument(method)
        KtCall<T>(call, gson, genericReturnType)
    } else {
        // 注意这里
        val response = okHttpClient.newCall(request).execute()

        val genericReturnType = method.genericReturnType
        val json = response.body?.string()
        gson.fromJson<Any?>(json, genericReturnType)
    }
}

// 判断当前接口的返回值类型是不是KtCall
private fun isKtCallReturn(method: Method) =
    getRawType(method.genericReturnType) == KtCall::class.java
```

#### 4.0 版本：支持挂起函数 ####

**扩展 KtCall**

```kotlin
/*
注意这里                   函数名称
   ↓                        ↓        */
suspend fun <T: Any> KtCall<T>.await(): T = TODO()
```

定义了一个扩展函数 await()。首先，它是一个挂起函数，其次，它的扩展接收者类型是 KtCall，其中带着一个泛型 T，挂起函数的返回值也是泛型 T

由于它是一个挂起函数，所以，我们的代码就可以换成这样的方式来写了。

```kotlin
fun main() = runBlocking {
    val ktCall = KtHttpV3.create(ApiServiceV3::class.java)
        .repos(lang = "Kotlin", since = "weekly")

    val result = ktCall.await() // 调用挂起函数
    println(result)
}
```

Kotlin 官方提供的一个顶层函数：suspendCoroutine{}，它的函数签名是这样的：

```kotlin
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T {
    // 省略细节
}
```

它其实就等价于挂起函数类型！可以使用 suspendCoroutine{} 来实现 await() 方法：

```kotlin
/*
注意这里                   
   ↓                                */
suspend fun <T: Any> KtCall<T>.await(): T = suspendCoroutine{
    continuation ->
    //   ↑
    // 注意这里 
}
```

**suspendCoroutine{} 的作用，其实就是将挂起函数当中的 continuation 暴露出来。**

回顾一下 Continuation 这个接口

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    // 关键在于这个方法
    public fun resumeWith(result: Result<T>)
}
```

整个 Continuation 只有一个方法，那就是 resumeWith()，根据它的名字我们就可以推测出，它是用于“恢复”的，参数类型是 Result。所以很明显，这就是一个带有泛型的“结果”，**它的作用就是承载协程执行的结果。**

```kotlin
suspend fun <T: Any> KtCall<T>.await(): T =
    suspendCoroutine { continuation ->
        call(object : Callback<T> {
            override fun onSuccess(data: T) {
                continuation.resumeWith(Result.success(data))
            }

            override fun onFail(throwable: Throwable) {
                continuation.resumeWith(Result.failure(throwable))
            }
        })
    }
```

当网络请求执行成功以后，我们就调用 resumeWith()，同时传入 Result.success(data)；如果请求失败，我们就传入 Result.failure(throwable)，将对应的异常信息传进去。

如果我们在协程当中调用 await() 方法的话，代码是可以正常执行的。不过，这种做法其实还有一点瑕疵，那就是不支持取消。

```kotlin
fun main() = runBlocking {
    val start = System.currentTimeMillis()
    val deferred = async {
        KtHttpV3.create(ApiServiceV3::class.java)
            .repos(lang = "Kotlin", since = "weekly")
            .await()
    }

    deferred.invokeOnCompletion {
        println("invokeOnCompletion!")
    }
    delay(50L)

    deferred.cancel()
    println("Time cancel: ${System.currentTimeMillis() - start}")

    try {
        println(deferred.await())
    } catch (e: Exception) {
        println("Time exception: ${System.currentTimeMillis() - start}")
        println("Catch exception:$e")
    } finally {
        println("Time total: ${System.currentTimeMillis() - start}")
    }
}

suspend fun <T : Any> KtCall<T>.await(): T =
    suspendCoroutine { continuation ->
        call(object : Callback<T> {
            override fun onSuccess(data: T) {
                println("Request success!") // ①
                continuation.resume(data)
            }

            override fun onFail(throwable: Throwable) {
                println("Request fail!：$throwable")
                continuation.resumeWithException(throwable)
            }
        })
    }

/*
输出结果：
Time cancel: 536   // ②
Request success!   // ③
invokeOnCompletion!
Time exception: 3612  // ④
Catch exception:kotlinx.coroutines.JobCancellationException: DeferredCoroutine was cancelled; job=DeferredCoroutine{Cancelled}@6043cd28
Time total: 3612
*/
```

* 结合注释①、③一起分析，我们发现，即使调用了 deferred.cancel()，网络请求仍然会继续执行。根据“Catch exception:”输出的异常信息，我们也发现，当 deferred 被取消以后我们还去调用 await() 的时候，会抛出异常
* 对比注释②、④，我们还能发现，deferred.await() 虽然会抛出异常，但是它却耗时 3000ms。虽然 deferred 被取消了，但是当我们调用 await() 的时候，它并不会马上就抛出异常，而是会等到内部的网络请求执行结束以后，才抛出异常，在此之前都会被挂起。

使用 Kotlin 官方提供的另一个 API：suspendCancellableCoroutine{}。

```kotlin
suspend fun <T : Any> KtCall<T>.await(): T =
//            变化1
//              ↓
    suspendCancellableCoroutine { continuation ->
        val call = call(object : Callback<T> {
            override fun onSuccess(data: T) {
                println("Request success!")
                continuation.resume(data)
            }

            override fun onFail(throwable: Throwable) {
                println("Request fail!：$throwable")
                continuation.resumeWithException(throwable)
            }
        })

//            变化2
//              ↓
        continuation.invokeOnCancellation {
            println("Call cancelled!")
            call.cancel()
        }
    }
```

可以往 continuation 对象上面设置一个监听：invokeOnCancellation{}，它代表当前的协程被取消了，这时候，我们只需要将 OkHttp 的 call 取消即可。

