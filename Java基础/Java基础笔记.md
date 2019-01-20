Java基础笔记

---
title: Java基础笔记
date: 2017-11-24
tags: Java
categories: 
grammar_cjkRuby: true

---

##### 泛型方法

泛型方法使得该方法能够独立与类而产生变化。泛型方法的指导原则：能够使用泛型的方法的地方就尽量使用泛型方法。泛型方法可以取代整个类泛型化。

对于一个static方法，无法访问泛型类的类型参数，所以，如果static方法需要使用泛型能力，就必须使其成为泛型方法。

```java
  private static List<String> list;

  public static void main(String[] args) {
    list = new ArrayList<>();
    startTest(list);
  }

  // 泛型方法
  public static <T> T startTest(List<T> list){
    return list.get(0);
  }
```

##### 注解

一个简单的注解例子：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Entity {
    String tableName() default "";
}

// 使用的时候
@Entity(tableName = "tasks")
public final class Task {
  // ...
}
```

##### 基本数据类型和包装类

String 和 int 之间的转换：

```java
// string转换成Integer，Integer会自动拆箱，转成int
Integer integer = Integer.valueOf("12334")
// 用上述写法，会有findbugs告警，提示你用 parseInt 会更高效
int i= Integer.parseInt("12334");

// int转换成string
String.valueOf(12)
```

这个地方呗被坑过，之前单纯的想通过：

```java
Integer i = Integer.getInteger("12334");
```

通过分析源码，`Integer`的这个静态方法是**返回系统属性为`12334`的整数值**，系统属性一般都不会有这个叫`12334`的属性，所以返回为null。

> 类似的还有：Boolean.getBoolean("True")，这个返回值一直为false，太坑。

Integer的缓存策略，Integer会缓存-128到127之间的值，在这个范围内的同一个int值是同一个Integer实例。否则的会new一个Integer实例。验证：

```java
  Integer i1 = 100;
  Integer i2 = 100;
  System.out.println(i1 == i2); // 返回 true，i1、i2拥有相同的内存地址
  Integer i3 = 1000;
  Integer i4 = 1000;
  System.out.println(i3 == i4); // 返回 false，i3、i4拥有不同的内存地址
```

#### 内部类

继承含有内部类的子类，在子类中重写父类的内部类。并不会覆盖父类的内部类，两个内部类是完全独立的两个实体。

#### final关键字

* final变量

  final修饰基本类型，不能改变值；修饰对象，不能改变引用。（所以可以改变对象的成员变量值）

  final关键字修饰的变量必须在定义处或者构造函数中赋值。


* final方法，final修饰的方法在子类中不能被改变。private修饰的方法，隐式的表明就是final方法。
* final类，无法被继承

final关键字可以修饰的变量、方法、类。

- 修饰类表示该类不可以被继承扩展
- 修饰变量表示该变量不可以被修改
- 修饰方法表示该方法不可以被重写。

虽然修饰的变量不可以被修改，并不是说该变量就是不可变的，比如：

```java
final List<String> strList = new ArrayList<>();
strList.add("hello");// 依然可以进行add操作
```

final约束的是`strList`不可以再被赋值。