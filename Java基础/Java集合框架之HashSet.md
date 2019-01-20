HashSet实现了`Set`接口，表明存放在HashSet中的元素不会出现重复的数值。(*允许一个null值*)

### HashSet中如何保持元素不重复的呢？

如果我们有这样一个数组：[2,5,7,9]，再向数组中插入一个元素`8`的时候，为了保证没有重复元素，需要遍历一遍列表。这样的话，当数组长度很长，这样的效率是比价低的，时间复杂度为O(n)。

什么方式可以实现O(1)时间复杂度的插入呢，就要借助于散列表（哈希函数）的方式。

具体实现过程大致如下：

1. 我们事先开辟好一个数组，长度为10；插入元素`i`的时候，我们根据：`i%10` ，即元素值对数组长度取余，得到应该存储在数组中的位置；

2. 按如上方式，再次插入一个元素`j` 的时候，根据`j%10` ，判断该位置是否已经有元素：没有元素直接插入，有元素在该位置就是重复的吗？

   显然不是，比如1和11对10取余，得到的都是1。出现了这种情况，我们就说出现了**哈希碰撞**。对于对象来说，就是他们的`hashCode` 值是一样的，要判断两个对像是否是同一个对象，就要借助于`equals`方法了。

HashSet的实现方式就是借助于这种思想，我们看看HashSet的实现源码。

### HashSet的实现

先看看构造函数（无参构造函数）：

```java
private transient HashMap<E,Object> map;

public HashSet() {
    map = new HashMap<>();
}
```

新建了一个HashMap对象。HashMap的`key`值，就是HashSet的元素值。看看`add`方法：

```java
// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

HashMap的`key`值，就是HashSet的元素值，`value` 值则是固定为Object：PRESENT。

因为HashMap的key值是不重复的，所以HashSet借助于HashMap的`key`值来存储元素值，从而实现HashSet的元素不重复。

### 关于equals与hashCode方法

我们经常说，重写`equals` 方法尽量要重写`hashCode`方法，对象的hashCode在HashTable、HashMap、HashSet等hash结构的集合时，有着举足轻重的地位。

#### equals()相等，hashCode一定要相等

因为equals相等，表明两个对象就是`相等`的了，hashCode必须要确保相同。反之则不成立，因为设计再好的hash函数，也会出现hash碰撞导致hashCode相同。

#### 什么时候重载equals方法

equals方法在Object类中是有默认实现的：比较两个对象的引用，是否指向同一块内存地址。

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

比如有个学生类：Student，我希望学号ID相同，两个对象相同，这就需要覆写`equals`方法了。

> Object类中的hashCode方法是一个native方法，值则是对象的内存地址值

### String类hashCode的实现

String类hashCode计算得到的hash值由该公式计算得到：

```
s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
```

其中，s为String存放字符串的final char数组，n数组长度。

代码实现：

```java
int hash;
for (int i=0;i<n;i++) {
    hash = 31 * hash + s[i];
}
```

