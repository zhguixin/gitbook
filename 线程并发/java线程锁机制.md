##### 自旋锁

关于自旋锁的解释，来自百度百科：

> 何谓**自旋锁**？它是为实现保护**共享资源**而提出一种锁机制。其实，自旋锁与**互斥锁**比较类似，它们都是为了解决对某项资源的互斥使用。无论是**互斥锁**，还是**自旋锁**，在任何时刻，最多只能有一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁。但是两者在调度机制上略有不同。对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。

自旋锁的经典写法：

```java
AtomicReference<Node<T>> head = new AtomicReference<>(null);

for (; ; ) {
    Node<T> tmpHead = head.get();
    if (head.compareAndSet(tmpHead, node)) {
     	node.setNext(tmpHead);
        return;
     }
}
```

*注意自旋锁会引入ABA的问题* 

##### CAS原子操作

`AtomicReference`底层调用的是UnSafe类的compareAndSwapObject方法，即CAS原子操作。

```java
/*
   判断valueOffset地址中的值与except 的值是否相等，如果相等则用update的值替换valueOffset地址的值并返回true，否则不进行替换返回false；
   其中，valueOffset代表的地址值，通过反射调用获得；
*/
public final boolean compareAndSet(V expect, V update) {
    return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
}
```

CAS即CompareAndSwap操作，`compareAndSwapObject`是一个native方法，这个方法最终会演化为一条CPU指令。其中，`compareAndSwapObject`四个参数的含义如下：

* 其中第一和第二个参数代表对象的实例以及地址

* 第三个参数代表期望值

* 第四个参数代表更新值

CAS的语义是，若期望值等于对象地址存储的值，则用更新值来替换对象地址存储的值，并返回true，否则不进行替换，返回false。 

##### Java并发包中锁的实现机制

与synchronize关键字不同，并发包中的锁，其阻塞机制是调用Unsafe类提供的park和unpark方法，结合CAS原子语句，实现了比synchronize更加高效灵活的各种锁。这两个方法都是native方法，定义如下：

```java
public native void park(boolean var1, long var2);
public native void unpark(Object var1);
```

>  park方法用来阻塞一个线程，第一个参数用来指示后面的参数是绝对时间还是相对时间，true表示绝对时间，false表示从此刻开始后的相对时间。调用park的线程就阻塞在此处。 
>
>  upark用来释放某个线程的阻塞，线程用参数var1表示

在Java并发包中，`LockSupport`类对这两个方法做了封装，使得使用更加方便。详情参考LockSupport的源码。

```java
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }

	public static void park() {
        UNSAFE.park(false, 0L);
    }
```



##### Java并发包中AbstractQueueSynchronizer的实现

在Java并发包中，这个类有详细的注释和例子，加上注释和代码总共有2300多行，代码量其实并不大。

Java各种锁的实现（ReentrantLock、ReentrantReadWriteLock等），都是基于`AbstractQueuedSynchronizer`，AQS提供了内部维护一个int型的原子变量，通过`tryAcquire`来设置这个变量表明状态正在被使用，通过`tryRelease`来获取这个状态，如果状态不可用将会发生阻塞，AQS内部有一个FIFO的队列，阻塞的线程被包装成`Node`添加到这个队列（这个队列被称为阻塞队列）。

要想实现一个自定义的锁，该类应该定义一个私有的内部来继承AQS并实现相关方法，最后通过这个调用这个内部类的方法来提供锁的功能。

```java
/**
实现了一个最简单的排他锁，当有线程获得锁时，其他线程只能等待；当这个线程释放锁时，另一个线程可以竞争获取锁
**/
public class MySimpleLock {
    private static class Sync extends AbstractQueuedSynchronizer {//内部类继承AQS

        private static final long serialVersionUID = 1L;

        @Override
        protected boolean tryAcquire(int acquire) {//重写tryAcquire方法
            return compareAndSetState(0, 1);//CAS操作
        }

        @Override
        protected boolean tryRelease(int release) {//重写tryRelease方法
            setState(0);
            return true;
        }

        protected Sync() {
            super();
        }
    }

    private final Sync sync = new Sync();

    public void lock() {//代理模式，外部调用 MySimpleLock.lock() 加锁
        sync.acquire(1);
    }

    public void unlock() {//代理模式，外部调用 MySimpleLock.unlock() 解锁
        sync.release(1);
    }
}
```

#### ReentrantLock

`ReentrantLock`是一个可重入锁，重入的意思就是，如果拥有该锁的线程再次进入这个临界区时，不会阻塞，依然可以进入（内部调用`tryAcquire`将state加一），其他线程会阻塞。被`ReentrantLock`阻塞的线程可以被中断，并抛出一个`InterruptedException`异常。

```java
if (currentThread == getExclusiveOwnerThread()) {state++}
```

`ReentrantLock`使用如下所示：

```java
  class Test {
    private final ReentrantLock lock = new ReentrantLock();
    // ...
 
    public void m() {
      lock.lock();  // 阻塞，直到条件成立；其后紧跟try语句
      try {
        // ... 执行操作
      } finally {
        lock.unlock() //在finally释放锁
      }
    }
  }}
```

`ReentrantLock`支持公平锁，和非公平锁，默认的话是非公平锁。

```java
static final class NonfairSync extends Sync {
  // ...
  final void lock() {
    /*
      先判断state的值是否为0（为0代表没有被占用）：
      为0的话，将state的值设为1，调用setExclusiveOwnerThread，表明当前线程获得锁；
      不为0的话，尝试获取锁，调用acquire;
    */
      if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
      else
        acquire(1);
  }
  // ...
}
```

其中`acquire`方法在AQS中实现：

```java
    /*
      tryAcquire在子类中实现，返回为true的话说明尝试获取锁成功，AQS中的这个acquire方法也就返回了；
      没有获取成功的话，包装当前线程为Node，加入到阻塞队列，主要操作在addWaiter中，头结点不存放线程信息
    */
	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

	/*
	  1. 获取当前节点的前驱节点；
	  需要获取当前节点的前驱节点，而头结点head所对应的含义是当前占有锁且正在运行。
	  2. 当前驱节点是头结点并且能够获取状态，代表该当前节点占有锁；
	  如果满足上述条件，那么代表能够占有锁，根据节点对锁占有的含义，设置头结点为当前节点。
	  3. 否则进入等待状态。
	  如果没有轮到当前节点运行，那么将当前线程从线程调度器上摘下，也就是进入等待状态。
	*/
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
              	//挂起线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```


