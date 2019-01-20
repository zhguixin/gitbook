#### 初探LinkedHashMap

该类继承自HashMap，相比较与HashMap通过单向链表来存储碰撞的数据，LinkedHashMap通过双向链表存储数据。设置**accessOrder**标志位来标识。

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

#### LRU算法

在`LinkedHashMap`中有个方法：

```java
protected boolean removeEldestEntry(Map.Entry<String, String> eldest)
```

根据这个方法的返回值，确定是否删除链表中的最后一个节点。因此我们在定义了LRUCached最大容量`cacheSize `后，可以按如下代码方式实现一个LRUCache结构：

```java
final int cacheSize = 100;

Map<String, String> map = new LinkedHashMap<String, String>((int) Math.ceil(cacheSize / 0.75f) + 1, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
    return size() > cacheSize;
    }
};
```

