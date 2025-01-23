为了描述服务器返回的内容，我们定义了两个数据类

```java
// 这种写法是有问题的，但这节课我们先不管。

data class RepoList(
    var count: Int?,
    var items: List<Repo>?,
    var msg: String?
)

data class Repo(
    var added_stars: String?,
    var avatars: List<String>?,
    var desc: String?,
    var forks: String?,
    var lang: String?,
    var repo: String?,
    var repo_link: String?,
    var stars: String?
)
```

除了数据类以外，我们还要定义一个用于网络请求的接口：

```kotlin
interface ApiService {
    @GET("/repo")
    fun repos(
        @Field("lang") lang: String,
        @Field("since") since: String
    ): RepoList
}
```

* GET 注解，代表了这个网络请求应该是 GET 请求
* Field 注解，代表了 GET 请求的参数。Field 注解当中的值也会和 URL 拼接在一起

```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class GET(val value: String)

@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class Field(val value: String)
```

GET 注解只能用于修饰函数，Field 注解只能用于修饰参数。另外，这两个注解的 Retention 都是 AnnotationRetention.RUNTIME，这意味着这两个注解都是运行时可访问的

再来看看 KtHttp 是如何使用的：

```kotlin
fun main() {
    // ①
    val api: ApiService = KtHttpV1.create(ApiService::class.java)

    // ②
    val data: RepoList = api.repos(lang = "Kotlin", since = "weekly")

    println(data)
}
```

* 相当于创建了 ApiService 这个接口的实现类的对象。
* 注释②：我们调用 api.repos() 这个方法，传入了 Kotlin、weekly 这两个参数，代表我们想查询最近一周最热门的 Kotlin 开源项目。

通过 Proxy，就可以动态地创建 ApiService 接口的实例化对象。具体的做法如下：

```kotlin
fun <T> create(service: Class<T>): T {

    // 调用 Proxy.newProxyInstance 就可以创建接口的实例化对象
    return Proxy.newProxyInstance(
        service.classLoader,
        arrayOf<Class<*>>(service),
        object : InvocationHandler{
            override fun invoke(proxy: Any?, method: Method?, args: Array<out Any>?): Any {
                // 省略
            }
        }
    ) as T
}    
```

InvocationHandler 其实是符合 SAM 转换要求的，所以我们的 create() 方法可以进一步简化成这样：

```kotlin
fun <T> create(service: Class<T>): T {

    return Proxy.newProxyInstance(
        service.classLoader,
        arrayOf<Class<*>>(service)
    ) { proxy, method, args ->
        // 待完成
    } as T
}
```

Proxy.newProxyInstance()，会帮我们创建 ApiService 的实例对象，**而 ApiService 当中的接口方法的具体逻辑，我们需要在 Lambda 表达式当中实现。**

```kotlin
object KtHttpV1 {

    // 底层使用 OkHttp
    private var okHttpClient: OkHttpClient = OkHttpClient()
    // 使用 Gson 解析 JSON
    private var gson: Gson = Gson()

    // 这里以baseurl.com为例，实际上我们的KtHttpV1可以请求任意API
    var baseUrl = "https://baseurl.com"

    fun <T> create(service: Class<T>): T {
        return Proxy.newProxyInstance(
            service.classLoader,
            arrayOf<Class<*>>(service)
        //           ①     ②
        //           ↓      ↓
        ) { proxy, method, args ->
            // ③
            val annotations = method.annotations
            for (annotation in annotations) {
                // ④
                if (annotation is GET) {
                    // ⑤
                    val url = baseUrl + annotation.value
                    // ⑥
                    return@newProxyInstance invoke(url, method, args!!)
                }
            }
            return@newProxyInstance null

        } as T
    }

    private fun invoke(url: String, method: Method, args: Array<Any>): Any? {
        // 待完成
    }
}
```

* method 的类型是反射后的 Method，在我们这个例子当中，它最终会代表被调用的方法，也就是 ApiService 接口里面的 repos() 这个方法。
* 注释②：args 的类型是对象的数组，在我们的例子当中，它最终会代表方法的参数的值，也就是“api.repos("Kotlin", "weekly")”当中的"Kotlin"和"weekly"。
* 注释③：method.annotations，代表了我们会取出 repos() 这个方法上面的所有注解，**由于 repos() 这个方法上面可能会有多个注解，因此它是数组类型。**
* 注释④：我们使用 for 循环，遍历所有的注解，找到 GET 注解。
* 注释⑤：我们找到 GET 注解以后，要取出 @GET(“/repo”) 当中的"/repo"，也就是“annotation.value”。这时候我们只需要用它与 baseURL 进行拼接，就可以得到完整的 URL；
* 注释⑥：return@newProxyInstance，用的是 Lambda 表达式当中的返回语法，在得到完整的 URL 以后，我们将剩下的逻辑都交给了 invoke() 这个方法。

```kotlin
private fun invoke(url: String, method: Method, args: Array<Any>): Any? {
    // ① 根据url拼接参数，也就是：url + ?lang=Kotlin&since=weekly
    // ② 使用okHttpClient进行网络请求
    // ③ 使用gson进行JSON解析
    // ④ 返回结果
}
```

```kotlin
private fun invoke(url: String, method: Method, args: Array<Any>): Any? {
    // ① 根据url拼接参数，也就是：url + ?lang=Kotlin&since=weekly

    // 使用okHttpClient进行网络请求
    val request = Request.Builder()
            .url(url)
            .build()
    val response = okHttpClient.newCall(request).execute()

    // ② 获取repos()的返回值类型 genericReturnType

    // 使用gson进行JSON解析
    val body = response.body
    val json = body?.string()
    //                              根据repos()的返回值类型解析JSON
    //                                            ↓
    val result = gson.fromJson<Any?>(json, genericReturnType)

    // 返回结果
    return result
}
```

利用反射，解析出 repos() 的返回值类型，用于 JSON 解析

```kotlin
private fun invoke(path: String, method: Method, args: Array<Any>): Any? {
    // 条件判断
    if (method.parameterAnnotations.size != args.size) return null

    // 解析完整的url
    var url = path
    // ①
    val parameterAnnotations = method.parameterAnnotations
    for (i in parameterAnnotations.indices) {
        for (parameterAnnotation in parameterAnnotations[i]) {
            // ②
            if (parameterAnnotation is Field) {
                val key = parameterAnnotation.value
                val value = args[i].toString()
                if (!url.contains("?")) {
                    // ③
                    url += "?$key=$value"
                } else {
                    // ④
                    url += "&$key=$value"
                }

            }
        }
    }
    // 最终的url会是这样：
    // https://baseurl.com/repo?lang=Kotlin&since=weekly

    // 执行网络请求
    val request = Request.Builder()
        .url(url)
        .build()
    val response = okHttpClient.newCall(request).execute()

    // ⑤
    val genericReturnType = method.genericReturnType
    val body = response.body
    val json = body?.string()
    // JSON解析
    val result = gson.fromJson<Any?>(json, genericReturnType)

    // 返回值
    return result
}
```

#### 函数式思维 ####

先来看看 KtHttpV1 这个单例的成员变量：

```kotlin
object KtHttpV1 {
    private var okHttpClient: OkHttpClient = OkHttpClient()
    private var gson: Gson = Gson()

    fun <T> create(service: Class<T>): T {}
    fun invoke(url: String, method: Method, args: Array<Any>): Any? {}
}
```

okHttpClient、gson 这两个成员是不支持懒加载的，因此我们首先应该让它们支持懒加载。

```kotlin
object KtHttpV2 {
    private val okHttpClient by lazy { OkHttpClient() }
    private val gson by lazy { Gson() }

    fun <T> create(service: Class<T>): T {}
    fun invoke(url: String, method: Method, args: Array<Any>): Any? {}
}
```

我们直接使用了 by lazy 委托的方式，它简洁的语法可以让我们快速实现懒加载

```kotlin
//                      注意这里
//                         ↓
fun <T> create(service: Class<T>): T {
    return Proxy.newProxyInstance(
        service.classLoader,
        arrayOf<Class<*>>(service)
    ) { proxy, method, args ->
    }
}
```

create() 会接收一个Class类型的参数。其实，针对这样的情况，我们完全可以省略掉这个参数

inline，来实现类型实化（Reified Type）。我们常说，Java 的泛型是伪泛型，而这里我们要实现的就是真泛型。

```kotlin
//  注意这两个关键字
//  ↓          ↓
inline fun <reified T> create(): T {
    return Proxy.newProxyInstance(
        T::class.java.classLoader, // ① 变化在这里
        arrayOf(T::class.java) // ② 变化在这里
    ) { proxy, method, args ->
        // 待重构
    }
}
```

通过使用 inline 和 reified 这两个关键字,使用``“T::class.java”``来得到 Class 对象。

reified 可以避免Java带来的泛型擦除问题，泛型擦除使得Java只能通过参数传递来获得T的classLoader

create() 里面“待重构”的代码该如何写

```kotlin
inline fun <reified T> create(): T {
    return Proxy.newProxyInstance(
        T::class.java.classLoader,
        arrayOf(T::class.java)
    ) { proxy, method, args ->

        return@newProxyInstance method.annotations
            .filterIsInstance<GET>()
            .takeIf { it.size == 1 }
            ?.let { invoke("$baseUrl${it[0].value}", method, args) }
    } as T
}
```

* 通过 method.annotations，来获取 method 的所有注解；
* 用filterIsInstance()，来筛选出我们想要找的 GET 注解。这里的 filterIsInstance 其实是 filter 的升级版，也就是过滤的意思；
* 判断 GET 注解的数量，它的数量必须是 1，其他的都不行，这里的 takeIf 其实相当于我们的 if；
* 通过拼接出 URL，然后将程序执行流程交给 invoke() 方法。这里的"?.let{}"相当于判空。

接下来我们来看看 invoke() 方法该如何重构。

```kotlin
fun invoke(url: String, method: Method, args: Array<Any>): Any? =
    method.parameterAnnotations
        .takeIf { method.parameterAnnotations.size == args.size }
        ?.mapIndexed { index, it -> Pair(it, args[index]) }
        ?.fold(url, ::parseUrl)
        ?.let { Request.Builder().url(it).build() }
        ?.let { okHttpClient.newCall(it).execute().body?.string() }
        ?.let { gson.fromJson(it, method.genericReturnType) }
```

* 第一步，我们通过 ``method.parameterAnnotations``，获取方法当中所有的参数注解，在这里也就是@Field("lang")、@Field("since")。
* 第二步，我们通过 takeIf 来判断，参数注解数组的数量与参数的数量相等，也就是说@Field("lang")、@Field("since")的数量是 2，那么["Kotlin", "weekly"]的 size 也应该是 2
* 这里的 mapIndexed，其实就是 map 的升级版，它本质还是一种映射的语法，“注解数组类型”映射成了“Pair 数组”，只是多了一个 index 而已。, Pair中 it 是注解，args[index]是第几个实参
* 使用 fold 与 parseUrl() 这个方法，拼接出完整的 URL，使用了函数引用的语法“::parseUrl”。而 fold 这个操作符，其实就是高阶函数版的 for 循环。
* 第五步，我们构建出 OkHttp 的 Request 对象，并且将 URL 传入了进去，准备做网络请求。
* 第六步，我们通过 okHttpClient 发起了网络请求，并且拿到了 String 类型的 JSON 数据。
* 最后，我们通过 Gson 解析出 JSON 的内容，并且返回 RepoList 对象。

看看用于实现 URL 拼接的 parseUrl() 是如何实现的。

```kotlin
private fun parseUrl(acc: String, pair: Pair<Array<Annotation>, Any>) =
    pair.first.filterIsInstance<Field>()
        .first()
        .let { field ->
            if (acc.contains("?")) {
                "$acc&${field.value}=${pair.second}"
            } else {
                "$acc?${field.value}=${pair.second}"
            }
        }
```

* 首先，我们从注解的数组里筛选出 Field 类型的注解；
* 通过 first() 取出第一个 Field 注解，这里它也应该是唯一的；
* 判断当前的 acc 是否已经拼接过参数，如果没有拼接过，就用“?”分隔，如果已经拼接过参数，我们就用“&”分隔。

2.0 版本的代码就完成了，完整的代码如下：

```kotlin
object KtHttpV2 {

    private val okHttpClient by lazy { OkHttpClient() }
    private val gson by lazy { Gson() }
    var baseUrl = "https://baseurl.com" // 可改成任意url

    inline fun <reified T> create(): T {
        return Proxy.newProxyInstance(
            T::class.java.classLoader,
            arrayOf(T::class.java)
        ) { proxy, method, args ->

            return@newProxyInstance method.annotations
                .filterIsInstance<GET>()
                .takeIf { it.size == 1 }
                ?.let { invoke("$baseUrl${it[0].value}", method, args) }
        } as T
    }

    fun invoke(url: String, method: Method, args: Array<Any>): Any? =
        method.parameterAnnotations
            .takeIf { method.parameterAnnotations.size == args.size }
            ?.mapIndexed { index, it -> Pair(it, args[index]) }
            ?.fold(url, ::parseUrl)
            ?.let { Request.Builder().url(it).build() }
            ?.let { okHttpClient.newCall(it).execute().body?.string() }
            ?.let { gson.fromJson(it, method.genericReturnType) }


    private fun parseUrl(acc: String, pair: Pair<Array<Annotation>, Any>) =
        pair.first.filterIsInstance<Field>()
            .first()
            .let { field ->
                if (acc.contains("?")) {
                    "$acc&${field.value}=${pair.second}"
                } else {
                    "$acc?${field.value}=${pair.second}"
                }
            }
}
```





​	



























