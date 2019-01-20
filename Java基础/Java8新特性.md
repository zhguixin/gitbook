Java8新特性

#### 1、Lambda表达式

Lambda 表达式通常使用 (argument) -> {body} 语法书写

#### 2、函数式编程

函数可以作为参数进行传递使用。



函数式接口就是仅声明了一个方法的接口，比如我们熟悉的Runnable，Callable，Comparable等都可以作为函数式接口。

每个 Lambda 表达式都能隐式地赋值给函数式接口：

```java
Runnable r = () -> System.out.println("hello world");
```

#### 3、流式编程

```java
/**
* 价格大于20的打九折后(乘以0.9)，再将所有价格相加求和
*/
final List<BigDecimal> prices = Arrays.asList(
    new BigDecimal("10"), new BigDecimal("30"), new BigDecimal("17"),
    new BigDecimal("20"), new BigDecimal("15"), new BigDecimal("18"),
    new BigDecimal("45"), new BigDecimal("12"));

BigDecimal totalPrices = prices.stream()
    .filter(price -> price.compareTo(BigDecimal.valueOf(20)) > 0)
    .map(price -> price.multiply(BigDecimal.valueOf(0.9)))
    .reduce(BigDecimal.ZERO, BigDecimal::add);

System.out.println("total prices=" + totalPrices);
```