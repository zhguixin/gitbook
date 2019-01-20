### Java集合框架之TreeMap

#### 1、概述

TreeMap是基于红黑树实现的一种带排序功能的集合。继承结构如下：

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```

TreeMap中的元素的有序的，排序的依据是存储在其中的键的natural ordering（自然序，也就是数字从小到大，字母的话按照字典序）或者根据在创建TreeMap时提供的Comparator对象，这取决于使用了哪个构造器。

- 继承 `AbstractMap`类，实现了`Map<K, V>`接口，实现了基本的操作，减少直接继承Map接口要重写更多的方法
- 实现 `NavigableMap` 接口，支持一系列的导航方法，实现了这个接口的类支持一些navigation methods，比如lowerEntry（返回小于指定键的最大键所关联的键值对），floorEntry（返回小于等于指定键的最大键所关联的键值对），ceilingEntry（返回大于等于指定键的最小键所关联的键值对）和higerEntry（返回大于指定键的最小键所关联的键值对）。一个NavigableMap支持对其中存储的键按键的递增顺序或递减顺序的遍历或访问
- 实现 `Cloneable` 接口，重写 `clone()`，实现浅拷贝(shallow copy of this map)
- 实现 `java.io.Serializable` 接口，支持序列化

TreeMap的插入和查找操作的时间复杂度都为O(logN)，这就得益于底层的实现的数据结构——红黑树。

> This implementation provides guaranteed log(n) time cost for the `containsKey`, `get`, `put` and ` remove` operations. 

#### 2、TreeMap源码分析

TreeMap提供了四个构造函数。如下：

```java
public TreeMap() //Key按照自然序排序，Key必须有自己的比较器（实现Comparable接口），key的类型要相同，否则抛出将ClassCastException异常
public TreeMap(Comparator<? super K> comparator)//传入自定义比较器
public TreeMap(Map<? extends K, ? extends V> m)//传入一个Map类型的集合
public TreeMap(SortedMap<K, ? extends V> m)//传入一个有序的Map类型集合，比较器为该SortedMap的
```

TreeMap中的存储的每个节点为TreeMapEntry，也就是红黑树的一个节点，其为TreeMap的内部类：

```java
static final class TreeMapEntry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    TreeMapEntry<K,V> left = null;
    TreeMapEntry<K,V> right = null;
    TreeMapEntry<K,V> parent;
    boolean color = BLACK;

    //左右子节点以及红黑属性的设置不在构造器阶段完成，而是在旋转和着色阶段完成
    TreeMapEntry(K key, V value, TreeMapEntry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
...
```

下面看一下，TreeMap的put操作：

```java
//返回值V：相同的key，更新新的value值前的旧value值
public V put(K key, V value) {
    TreeMapEntry<K,V> t = root;
    if (t == null) {//根结点位null，TreeMap还为包含任何key-value对，JDK1.8对key为null的情况作了限定，之前TreeMap是不允许key为null的
        if (comparator != null) {
            if (key == null) {
                comparator.compare(key, key);
            }
        } else {
            if (key == null) {
                throw new NullPointerException("key == null");
            } else if (!(key instanceof Comparable)) {
                throw new ClassCastException(
                    "Cannot cast" + key.getClass().getName() + " to Comparable.");
            }
        }

        root = new TreeMapEntry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
```

当TreeMap含有元素时，也就是root不为null的时候，插入操作：

```java
int cmp;
TreeMapEntry<K,V> parent;
// split comparator and comparable paths
Comparator<? super K> cpr = comparator;
//自定义了比较器的情况
if (cpr != null) {
    do {//二分遍历查找key所在的节点，赋值为parent
        parent = t;
        cmp = cpr.compare(key, t.key);//使用自定义比较强比较key值
        if (cmp < 0)
            t = t.left;
        else if (cmp > 0)
            t = t.right;
        else
            return t.setValue(value);//key值已经存在，更新value值，返回更新之前的value值
    } while (t != null);
} else {//使用key的自然序排序
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
    Comparable<? super K> k = (Comparable<? super K>) key;
    do {
        parent = t;
        cmp = k.compareTo(t.key);
        if (cmp < 0)
            t = t.left;
        else if (cmp > 0)
            t = t.right;
        else
            return t.setValue(value);
    } while (t != null);
}
//新增节点，插入该节点，作为parent的子节点
TreeMapEntry<K,V> e = new TreeMapEntry<>(key, value, parent);
if (cmp < 0)
    parent.left = e;
else
    parent.right = e;
fixAfterInsertion(e);//旋转操作保证还是一个红黑树
size++;
modCount++;
return null;
```

红黑树的插入操作很简单，变态的就是关于维持红黑树特性的旋转操作，操作重点在fixAfterInsertion方法内。这个方法在深入理解了红黑树原理下，再阅读会事半功倍。由于对红黑树还是浅尝辄止的状态，暂且搁置。



