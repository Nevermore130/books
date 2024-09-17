### 泛型基础 ###

在定义泛型的时候，其实还可以为它的泛型参数增加一些边界限制，比如说，强制要求传入的泛型参数，必须是 TV 或者是它的子类。这叫做泛型的上界。

```kotlin
//               差别在这里
//                   ↓
class Controller<T: TV> {
    fun turnOn(tv: T) {}
    fun turnOff(tv: T) {}
}
```

我们直接在 fun 关键字的后面加上用尖括号包起来的 T，就可以为函数增加泛型支持

```kotlin
//     函数的泛型参数
//   ↓             ↓
fun <T> turnOn(tv: T){ ... }
fun <T> turnOff(tv: T){ ... }

fun turnOnAll(mi1: XiaoMiTV1, mi2: XiaoMiTV2) {
//      泛型实参自动推导
//          ↓
    turnOn(mi1)
    turnOn(mi2)
}
```

### 型变（Variance） ###

简单来说，它就是为了解决泛型的不变性问题。事实上，型变讨论的是：在已知 Cat 是 Animal 的子类的情况下，MutableList与MutableList之间是什么关系。

```kotlin
// 需要父类集合，传入子类集合

foo(list: MutableList<Animal>) {
    // 出错，Cat集合不能存Dog对象
    list.add(Dog())
    // 通过
    val animal: Animal = list[0] // 取出的Cat对象
}

fun main() {
    // 需要MutableList<Animal>，实际传MutableList<Cat>
    foo(mutableListOf<Cat>(Cat()))
    // 实际上，编译器在这里就会提示错误，我们现在假设编译器不阻止我们，会出什么问题
}
```

从这段代码的注释中，我们能看到，当程序需要 Animal 的集合时，如果我们传入的是 Cat 的集合，我们就可以往 list 里添加其他类型的动物，比如 Dog。然而，Dog 是无法存入 Cat 的集合的。

所以，在默认情况下，编译器会认为MutableList与MutableList之间不存在任何继承关系，它们也无法互相替代，这样就不会出现前面提到的两种问题。这就是**泛型的不变性**

### 逆变 ###

```kotlin
open class TV {
    open fun turnOn() {}
}

class XiaoMiTV1: TV() {
    override fun turnOn() {}
}

class Controller<T> {
    fun turnOn(tv: T)
}
```

来设想一个买遥控器的场景：

```kotlin
//                      需要一个小米电视1的遥控器
//                                ↓
fun buy(controller: Controller<XiaoMiTV1>) {
    val xiaoMiTV1 = XiaoMiTV1()
    // 打开小米电视1
    controller.turnOn(xiaoMiTV1)
}
```

在函数的内部，我们需要打开一台小米电视机。那么，当我们需要打开一台小米电视机的时候，我们是否可以用一个“万能的遥控器”呢？当然可以！所以，我们可以写出下面这样的代码：

```kotlin
fun main() {
//                             实参
//                              ↓
    val controller = Controller<TV>()
    // 传入万能遥控器，报错
    buy(controller)
}
```

由于我们传入的泛型实参是 TV，它是所有电视机的父类。因此，Controller 内部将会处理所有电视机型号的开机、关机。这时候，它就相当于一个万能遥控器，万能遥控器当然也可以打开小米电视 1。

从道理上来讲，我们的推理是没有错的，不过 Kotlin 编译器会报错，报错的内容是说“类型不匹配”，需要的是小米遥控器```Controller<XiaoMiTV1>```，你却买了个万能遥控器``` Controller<TV> ```

第一种做法，是修改泛型参数的使用处代码，它叫做使用处型变。具体做法就是修改 buy 函数的声明，在 XiaoMiTV1 的前面增加一个 in 关键字

```kotlin
//                         变化在这里
//                             ↓
fun buy(controller: Controller<in XiaoMiTV1>) {
    val xiaoMiTV1 = XiaoMiTV1()
    // 打开小米电视1
    controller.turnOn(xiaoMiTV1)
}
```

第二种做法，是修改 Controller 的源代码，这叫声明处型变。具体做法就是，在泛型形参 T 的前面增加一个关键字 in：

```kotlin
//            变化在这里
//               ↓
class Controller<in T> {
    fun turnOn(tv: T)
}
```

### 协变（Covariant） ###

```kotlin
open class Food {}

class KFC: Food() {}

//还有一个饭店的角色：

class Restaurant<T> {
    fun orderFood(): T { /*..*/ }
}
```

点外卖方法

```kotlin
//                      这里需要一家普通的饭店，随便什么饭店都行
//                                     ↓
fun orderFood(restaurant: Restaurant<Food>) {
    // 从这家饭店，点一份外卖
    val food = restaurant.orderFood()
}

fun main() {
//                  找到一家肯德基
//                        ↓
    val kfc = Restaurant<KFC>()
// 需要普通饭店，传入了肯德基，编译器报错
    orderFood(kfc)
}
```

如果我们直接运行上面的代码，会发现编译器提示最后一行代码报错，报错的原因同样是：“类型不匹配”，我们需要的是一家随便类型的饭店```Restaurant<Food>```，而传入的是肯德基```Restaurant<KFC>```

第一种做法，还是修改泛型参数的使用处，也就是使用处型变。具体的做法就是修改 orderFood() 函数的声明，在 Food 的前面增加一个 out 关键字：

```kotlin
//                                变化在这里
//                                    ↓
fun orderFood(restaurant: Restaurant<out Food>) {
    // 从这家饭店，点一份外卖
    val food = restaurant.orderFood()
}
```

第二种做法，是修改 Restaurant 的源代码，也就是声明处型变。具体做法就是，在它泛型形参 T 的前面增加一个关键字 out：

```kotlin
//            变化在这里
//                ↓
class Restaurant<out T> {
    fun orderFood(): T { /*..*/ }
}
```

### 实战与思考 ###

```kotlin
//              逆变
//               ↓
class Controller<in T> {
//                 ①
//                 ↓
    fun turnOn(tv: T)
}

//               协变
//                ↓
class Restaurant<out T> {
//                   ②
//                   ↓
    fun orderFood(): T { /*..*/ }
}
```

传入 in，传出 out。或者，我们也可以说：泛型作为参数的时候，用 in，泛型作为返回值的时候，用 out。

Kotlin 源码当中型变的应用。首先，是逆变的应用。

```kotlin
//                          逆变
//                           ↓
public interface Comparable<in T> {
//                                   泛型作为参数
//                                       ↓
    public operator fun compareTo(other: T): Int
}
```

再来看看协变在 Kotlin 源码当中的应用。

```kotlin
//                        协变
//                         ↓
public interface Iterator<out T> {
//                         泛型作为返回值
//                              ↓    
    public operator fun next(): T
    
    public operator fun hasNext(): Boolean
}
```





























