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

运用了空安全思维当中的泛型边界“T: Any”，这样一来，我们就可以保证 T 类型一定是非空的。

还需要一个 KtCall 类，它的作用是承载 Callback，或者说，它是用来调用 Callback 的。

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

1. 为了将请求任务派发到异步线程，我们需要使用 OkHttp 的异步请求方法 enqueue()。
2. 根据请求结果，调用 callback.onSuccess() 或者是 callback.onFail()。如果请求成功了，我们在调用 onSuccess() 之前，还需要用 Gson 将请求结果进行解析，然后才返回。
3. 返回 OkHttp 的 Call 对象。

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

实现了 KtCall 以后，我们就只差 ApiService 这个接口了，这里我们定义 ApiServiceV3

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

只是在最后封装了一个 KtCall 对象，直接返回。所以在后续调用它的时候，我们就可以这么写了：ktCall.call()。

在后续调用它的时候，我们就可以这么写了：ktCall.call()。

```kotlin
private fun testAsync() {
    // 创建api对象
    val api: ApiServiceV3 = KtHttpV3.create(ApiServiceV3::class.java)

    // 获取ktCall
    val ktCall: KtCall<RepoList> = api.repos(
        lang = "Kotlin",
        since = "weekly"
    )

    // 发起call异步请求
    ktCall.call(object : Callback<RepoList> {
        override fun onSuccess(data: RepoList) {
            println(data)
        }

        override fun onFail(throwable: Throwable) {
            println(throwable)
        }
    })
}
```





