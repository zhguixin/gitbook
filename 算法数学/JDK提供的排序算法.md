在JDK集合框架中提供了两个工具类：`Arrays`和`Collections`，分别用来处理数组类型和集合类型。两个关键的操作就是**排序**和**查找**。

### JDK提供的排序算法

在`Arrays`类中提供了静态`sort`方法的多个重载方法，真正进行排序的是交给了`DualPivotQuicksort.sort`

```java
public static void sort(int[] a) {
    DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
}
```

**DualPivotQuicksort**被称作是**双轴快速排序**，这是一种改进的快速排序算法；在这个类的实现内部，会根据待排序元素的数目分别选择插入排序、快速排序和归并排序。

在`Collections`类中提供了静态`sort`方法的多个重载方法，最终经过处理后又调用了`Arrays.sort()`

```java
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}

// List.java
default void sort(Comparator<? super E> c) {
    // 集合类型转换为数组类型
    Object[] a = this.toArray();
    // 调用回Array.sort()方法
    Arrays.sort(a, (Comparator) c);
    ListIterator<E> i = this.listIterator();
    for (Object e : a) {
        i.next();
        i.set((E) e);
    }
}
```

#### Arrays对对象数组类型的

Arrays类中提供了一个方法，来对对象数组进行排序

```java
public static void sort(Object[] a) {
    if (LegacyMergeSort.userRequested)
        legacyMergeSort(a);
    else
        ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
}

// 泛型支持
public static <T> void sort(T[] a, Comparator<? super T> c) {
    if (c == null) {
        sort(a);
    } else {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a, c);
        else
            TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
}
```

其中，`legacyMergeSort`在JDK中标注以后将被移除，先不考虑。在调用者没有显示指定比较器时，使用的是`ComparableTimSort`，自定义了比较器选择的是`TimSort`，本质上是一样的，重点就在于`TimeSort`这个排序算法。

**TimSort**，这种排序在思想是一种归并和二分插入排序（算是对直接插入排序的一种优化）相结合的一种排序算法，思路是先查找数据集中已经排好序的分区（被称为run），然后再合并这些分区来达到排序的目的。

在JDK1.8中引入了并行排序算法`parallelSort()`，底层实现依赖于**fork-join**框架，来充分利用多核CPU提高排序效率。

参考

[JDK提供的默认排序算法](https://blog.csdn.net/lingzhm/article/details/45022385)

[Timsort原理介绍](https://blog.csdn.net/yangzhongblog/article/details/8184707?utm_source=blogxgwz0)