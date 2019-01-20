Kotlin知识点

1、泛型

泛型是Java在1.5之后引入的，不同于C++中的泛型，Java中的泛型在运行时会被擦除；所以在Java语言中：

```java
 public <T> void test() { 
     System.out.println(T.class);
 }
```

这段代码是编译不过的，无法通过泛型类型`T` 得到class对象；

在Kotlin语言中，可以通过关键字：`inline` 和 `reifield` ，实现一下：

```kotlin
inline fun <reified T> Gson.fromJson(json: String): T { 
     return fromJson(json, T::class.java) 
 }
```

同为JVM语言，这里主要是通过`inline` 关键字，在调用处将函数展开，泛型类型`T` 就可以确定了。



型变

型变包括协变、逆变、不变 ；

* 协变：`? extends E ` ，Kotlin：`out E` 

  表示允许传入的参数可以是泛型参数类型为 Number 子类的任意类型 ；

* 逆变：`? super E` ，Kotlin：`in E` 

  表示元素类型为 E 及其父类

2、闭包

一段程序代码通常由常量、变量和表达式组成，然后使用一对花括号“{}”来表示闭合，并包裹着这些代码，由这对花括号包裹着的代码块就是一个闭包。

```kotlin
// 把一个闭包赋值给一个变量：sum，该变量就是一个函数，函数类型为：(Int,Int) -> Int
// 接收两个整型值，返回一个整型值
val sum:(Int,Int)->Int = { i:Int, j:Int -> i+j }

// 调用
println(sum(1,2))
// 或者调用 invoke() 方法
println(sum.invoke(1,2))
```

闭包可以用在许多地方。它的最大用处有两个：

* 一个是可以读取函数内部的变量

  定义嵌套函数，在函数内部在定义一个函数

* 另一个就是让这些变量的值始终保持在内存中

  ```kotlin
  fun justCount(): ()->Unit { // 返回一个函数
      var count = 0
      return {
          println(count++)
      }
  }
  
  // 调用
  val count = justCount() //把返回值赋值给一个变量：count
  count.invoke() // 通过invoke调用
  count() // 通过加括号调用，直接调用闭包函数
  // 输出 0 1
  ```

参考：[Kotlin语法基础，函数与闭包](https://blog.csdn.net/yzzst/article/details/74619101)

:smile: