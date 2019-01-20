1、改写`if-else`  表达式

```kotlin
if (forecast != null) dataMapper.convertDayToDomain(forecast)
```

改写为：

```kotlin
forecast?.let {dataMapper.convertDayToDomain(it)}
```

这样的话，`forecast` 不为空执行`let` 函数，为空直接跳过该语句。

其中`let` 函数的原型：

```kotlin
inline fun <T, R> T.let(f: (T) -> R): R = f(this)
```

其中`let` 函数为**对象T**的扩展函数，返回值类型就是扩展函数中最后一个语句的的返回值。类似的高阶函数还有：with、apply、also等。

2、with函数

函数原型为：

```kotlin
inline fun <T, R> with(receiver: T, f: T.() -> R): R = receiver.f()
```

with函数接收两个值，第一个值是对象，第二个值为该对象的扩展函数。

with函数赋值给一个函数，函数式编程：

```kotlin
fun getInfo(str:String) = with(str) {
    val strUpper = str.toUpperCase()
    strUpper.length // 等同于 return@with strUpper.length
}
```

函数的返回值，就是with函数中最后一个语句的返回值

3、apply函数

函数原型为：

```kotlin
inline fun <T> T.apply(f: T.() -> Unit): T { f(); return this }
```

可以看出，apply函数的返回值是固定的，一直返回的是`T` 对象。接收的`T` 的扩展函数的返回值固定为空。这种类型可以替代在java中建造者模式中创建的类。

```kotlin
// 建造者类
class BUilder {
    lateinit var title:String
    lateinit var url:String
    // ...
}

// 调用该类进行构造的时候，直接用apply的函数
val builder = Builder().apply {
    title = "kotlin学习"
    url = "zhguixin.site"
}
```

