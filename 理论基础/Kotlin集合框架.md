Kotlin集合框架在Java基础上又做了扩展。

新增的接口有：

Iterable

MutableIterable Collection

MutableCollection

List /MutableList

Set/MutableSet

1、创建一个列表

```kotlin
val list = listOf(1,2,3,4,5)
```



集合高阶函数

map：对集合进行map函数，对集合中的每个元素进行操作，返回这个集合；

flatMap：将多个集合进行合并，合并成一个集合，返回这个集合。flat，的扁平之意，可体会一下。举例：

```kotlin
val list = arrayOf(1..10, 20..30, 50..60)
println(list.size) // size为 3
val mergedList = list.flatMap { it }
println(mergedList.size) // size为 32
```

> 同样的在RxJava中也有map、flatMap的操作符。两者返回的类型不一样：map返回一个集合类型，对下游来说，它直接拿到的是整个集合；flatMap中返回Observer，将集合中的每个元素告诉下游，对下游来说拿到的是单个集合中的元素