### Java集合框架源码分析

#### HashMap源码分析

HashMap采用了数组加链表的方式来存储数据。

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

~~HashMap的put方法，其中数组（bucket）就在扩容里面进行创建。最后，检查是否需要扩容(resize方法)。~~



#### 初探LinkedHashMap

该类继承自HashMap，相比较与HashMap通过单向链表来存储碰撞的数据，LinkedHashMap通过双向链表存储数据。设置accessOrder标志位来标识。

- accessOrder为false，表示按插入顺序排序，默认值；
- accessOrder为true，表示按访问顺序排序 ，实现LRU算法需要实现此值。双向链表的尾结点，作为最经常使用的结点，所以每次get操作，都要讲该结点放到链表末尾：

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)//LRU算法，该值置为true
            afterNodeAccess(e);
        return e.value;
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

```

