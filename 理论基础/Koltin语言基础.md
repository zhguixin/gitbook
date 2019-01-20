Koltin语言基础

 1、`?` 和`!!` 

* `?` 加在变量后面，表示该变量可能为空。

  ```kotlin
  str?.length() // 如果str为null，则不执行该语句，避免了空指针异常
  ```

* `!!` 加在变量后面，从编写值角度认为该值一定不为null

  ```kotlin
  str!!.length()// 如果str为null，会抛异常
  ```

2、with函数，接收一个对象和一个扩展函数作为参数。对象会执行这个扩展函数。

```kotlin
with(weekForecast.dailyForecast[position]) {
    holder.textView.text = "$date - $description - $high/$low"
}
```

3、扩展函数，定义在类之外的函数，可以作为这个类的成员被调用。

```kotlin
package strings

fun String.lastChar():Char = get(length - 1)

// 调用的时候
import strings.*
println("Kotlin".lastChar())
```

4、类声明

```kotlin
class User(val name:String)
```

这个类的声明包含着两个信息，构造函数接收一个参数，并将这个参数赋值给`name` 。等价于：

```kotlin
class User constructor(_name:String) {
    val name:String
    init {
        name = _name
    }
}
```

5、延迟加载

```kotlin
var mCached by lazy {mutableMapOf()}
```

在单例模式的实现时，会使用到`by lazy` 这个代理属性。比如：

```kotlin
class Singleton {
    private constructor()
    companion object {
        val INSTANCE by lazy { Singleton() }
    }
}
```

单例模式的要点就是：构造函数私有、懒加载、线程安全（可以做Double-Check）

6、Lambda表达式

对于在Android中实现onClickListener接口：

```kotlin
view.setOnClickListener(object:onClickListener {
    override fun onClick(v:View) {
        // do something
    }
})

// 简化一，类似的只接受一个函数
view.setOnClickListener({view -> /*do something*/})
// 简化二，如果这个函数入参没有被用到，可以去掉函数入参 view
view.setOnClickListener({/*do something*/})
// 简化三，如果这个lambda表达式是函数的最后一个参数，可以将这个Lambda表达式移到圆括号外面
view.setOnClickListener(）{view -> /*do something*/}
// 简化四，如果函数只有这一个Lambda表达式作为函数参数，可以直接将圆括号去掉
view.setOnClickListener {
    /*do something*/
}
```

其中，对于Lambda表达式中，只有一个参数时，可以省略掉这个参数，用`it`表示：

```kotlin
ForecastListAdapter(result) { forecast -> toast(forecast.date) }
// 等价于
ForecastListAdapter(result) { toast(it.date) }
```

7、扩展函数（Extension functions）

扩展函数是一个函数，为一个类添加一个新的特性（Feature）。通过这种方式，对于一个我们无法直接修改的类，我们可以扩展一些有用的方法。当然这种方式并不会修改这个原始的类。

举个栗子，我们访问网络，并得到它的响应：

```kotlin
import java.net.URL

fun run(url:String) {
    val requestStr = URL(url).readText()
}
```

其中，`readText` 就是URL这个java类的扩展函数，在Kotlin的标准库中定义如下：

```kotlin
@kotlin.internal.InlineOnly
public inline fun URL.readText(charset: Charset = Charsets.UTF_8): String = readBytes().toString(charset)
```

`readText` 作为扩展函数，被赋值为：`this.readBytes().toString(charset)` 。其中的**this**表示的就是URL的实例，可以被省略掉。

`readText` 可以接受一个参数，指定编码格式，不指定 的话默认值就是：UTF_8。

8、尾递归优化

使用关键字`tailrec` 修饰要递归调用的函数，优化尾递归（递归语句之后没有其他语句的调用，称为尾递归）。实际上是编译器的优化，将递归调用变为迭代调用。

9、静态类

静态类中的方法都是静态的，通过关键字`object` 声明，类似于在Java中经常使用的`Utils` 工具类：

```kotlin
object Utils {
    fun getIMEI() {}
}
// 调用的时候
Utils.getIMEI()
```

10、泛型上下界限

在Java中， ? super T，指定泛型为T的父类型；？extens T，指定泛型为T的子类型。

在Kotlin中，分别通过关键字`in`  、`out` 指定。

> MutableList<out T> in Kotlin means the same as MutableList<? extends T> in Java. The in-projected MutableList<in T> corresponds to Java’s MutableList<? super T>.    

参考Medium上的一篇文章：[In and out type variant of Kotlin](https://link.jianshu.com/?t=https%3A%2F%2Fandroid.jlelse.eu%2Fin-and-out-type-variant-of-kotlin-587e4fa2944c) 

11、泛型固化

再使用Retrofit时，调用Retrofit的`create` 方法时，需要传递一个接口的class对象：

```kotlin
//我们封装了一个RetrofitHelper类，其中的createApiCall方法
fun <T> createApiCall(clazz: Class<T>):T {
    return createRetrofit().create(clazz)
}
// 调用的时候
createApiCall(Api::class.java)
```

进一步简化：

```kotlin
fun <T> createUserApiCall(clazz: Class<T>):T = createRetrofit().create(clazz)
```

使用`reified` 固化泛型，这样的话，直接可以使用：`T::class.java` ：

```kotlin
inline fun <reified T> createApiCall():T = createRetrofit().create(T::class.java)
```

去掉了函数参数，调用的时候直接指定泛型就可以了。

```kotlin
createApiCall<Api>()
```

>任何被声明称inline的函数都会把函数体内的所有代码直接复制到每一个被调用的地方，而由于泛型参数的不同，所以每一个调用inline函数的位置都会因为泛型的不同而有所不同，这样做其实就是间接的保留了泛型的运行时信息 

