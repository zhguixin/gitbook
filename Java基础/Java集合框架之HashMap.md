---
title: HashMap源码分析
date: 2017-08-13 22:47:44
tags: Java集合框架
categories: 
---

## HashMap源码分析

HashMap采用了数组加链表的方式来存储数据。

### hash计算

在HashMap中，并没有直接使用对象的`hashCode`值，而是在hashCode值得基础上，又做了一次运算，进一步降低存放在哈希桶的数据碰撞：

```java
// 该方法称之为扰动函数
static final int hash(Object key) {   //jdk1.8
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 hashCode右移16位，再与hashCode异或，让高位参与运算，以此来加大低位的随机性
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`hash`函数的返回值，才是HashMap要用到的真正哈希值，得到哈希值后，确定数组下标：

```java
// length为数组长度
length = (tab = resize()).length;
// i为数组下标
i = (length - 1) & hash;
```

HashMap的得到数组下标并没有采用根据**hash值对数组长度取余**的处理方式，而是通过位运算的过程来进行，位运算的效率是高于取余运算的。但是要保证数组的长度要始终是**2的n次方**。代码中`resize()`过程。

> hash值是一个int型的值，对于32位有符号int值的取值范围：-2147483648~2147483648。前后有40亿的映射空间。如果直接拿来取模运算(其实就是**与运算**)的话，由于哈希桶数组不会很大，这就导致一直是hashCode的低位一直在参与运算，高为没有参与。

### 扩容过程

下面简单分析一下，向HashMap中push一个数据的过程，超过原数组容量进行扩容的过程。（JDK1.7与JDK1.8差异很大，先以1.7为源码描述）

```java
 Map<Integer,String> hashMap = new HashMap<>(2,0.75f);
 hashMap.put(3, "value1");
 hashMap.put(5, "value2");
 hashMap.put(7, "value3");
```

指定HashMap的容量为2，负载因子为0.75。阈值由下面函数给出，超过后push过的数量超过该阈值时就进行扩容操作：

```java
this.threshold = tableSizeFor(initialCapacity);

static final int tableSizeFor(int cap) {
   	int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

计算得到的阈值是2，也就是说当向该hashMap put第三个元素时，会触发扩容操作。

JDK1.7 将原来存储在小数组中的值拷贝到新数组中去：

```java
 void transfer(Entry[] newTable) {
   	 Entry[] src = table;                   //src引用了旧的Entry数组
     int newCapacity = newTable.length;
     for (int j = 0; j < src.length; j++) { //遍历旧的Entry数组
         Entry<K,V> e = src[j];             //取得旧Entry数组的每个元素
         if (e != null) {
             src[j] = null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）
             do {
                 Entry<K,V> next = e.next;
                 int i = indexFor(e.hash, newCapacity); //！！重新计算每个元素在数组中的位置
                 e.next = newTable[i]; //标记[1]
                 newTable[i] = e;      //将元素放在数组上
                 e = next;             //访问下一个Entry链上的元素
             } while (e != null);
         }
     }
 }
```

![jdk1.7扩容例图](https://tech.meituan.com/img/java-hashmap/jdk1.7扩容例图.png)

上图描述了hashMap在扩容过程中，链表的重建过程采用的是头插法，因此新的链表会出现链表倒置的情况。

注意：由于HashMap是非线程安全的，在多线程下进行push操作，当超过指定容量进行扩容时，容易出现环形链表，造成死锁。

在JDK1.8中数组桶(table[])的初始化采用lazy-load，在put元素时才会初始化，即`resize()`操作，其中也包含了扩容操作。默认初始数组的长度为16，每次扩容时增加为原来两倍的长度。


