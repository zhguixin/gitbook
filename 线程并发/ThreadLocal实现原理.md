ThreadLocal实现原理

## 基本使用
为了保证多个线程中的数据不相互影响，我们可以使用ThreadLocal来存储只供当前线程显示对其他线程隐藏的变量：

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();

threadLocal.set("main-thread");
ExecutorService singleThreadPool = Executors.newSingleThreadExecutor();
singleThreadPool.execute(()-> {
    threadLocal.set("thread-1");
});

try {
    Thread.sleep(1000);
} catch (Exception e) {
    // TODO: handle exception
}

System.out.println(threadLocal.get());
System.out.println("end!!");
```

## 内部实现

如果让我来实现一个类似的功能，我可能会使用ConcurrentHashMap，以thread为key，每个线程有一个属于自己的变量值。但是即使用了ConcurrentHashMap，也不能完全阻止并发，毕竟ConcurrentHashMap解决并发使用的是降低锁的粒度(Java7 分段加锁机制)。

接下来，看一看ThreadLocal是如何实现的呢。

总体架构上，ThreadLocal更像是一个工具类，提供`get` 、`set`、`remove` 等方法。ThreadLocal的内部类：**ThreadLocalMap**真正维护数据(虽然叫Map，并不保存key-value对)，每个线程都会有一个自己的实例。

```java
// ThreadLocal的 set 方法
public void set(T paramT)
  {
    Thread localThread = Thread.currentThread();
    ThreadLocalMap localThreadLocalMap = getMap(localThread);
    // 当前线程没有创建 ThreadLocalMap，就创建之
    if (localThreadLocalMap != null) {
      localThreadLocalMap.set(this, paramT);
    } else {
      createMap(localThread, paramT);
    }
  }
```

ThreadLocalMap由Entry数组构成的，Entry构造函数传入key和value。key是：ThreadLocal；value存储的是线程独有的数据，为了防止内存泄漏，其中key是指向ThreadLocal的弱引用。

```java
    ThreadLocalMap(ThreadLocal<?> paramThreadLocal, Object paramObject) {
      table = new Entry[16]; // 默认table数组长度为16
      int i = threadLocalHashCode & 0xF;
      table[i] = new Entry(paramThreadLocal, paramObject);
      size = 1;
      setThreshold(16);
    }

	// Entry是 ThreadLocalMap 的静态内部类
    static class Entry
      extends WeakReference<ThreadLocal<?>> {
      Object value;
      
      Entry(ThreadLocal<?> paramThreadLocal, Object paramObject)
      {
        super();
        value = paramObject;
      }
    }
```

在ThreadLocalMap中维护的Entry数组——table，它的**数组下标是threadLocal的hash code**，这种做法也是增大了数组的查找速率。根据ThreadLocal可以很快的得到Entry。

```java
    private Entry getEntry(ThreadLocal<?> paramThreadLocal) {
      // i 为数组table下标
      int i = threadLocalHashCode & table.length - 1;
      Entry localEntry = table[i];
      if ((localEntry != null) && (localEntry.get() == paramThreadLocal)) {
        return localEntry;
      }
      return getEntryAfterMiss(paramThreadLocal, i, localEntry);
    }
```

得到Entry之后，就可以得到存储在Entry中本地变量值value。

