为了充分利用多核CPU的优势，我们可以在自己的进程中开启多个线程来**并行**的处理任务，这些线程可以共享进程的资源，又可以独立的调度，实现更高的效率。

首先介绍一下，线程相关的基本知识，包括：

* 线程的创建
* 线程的实现原理
* 线程生命周期
* 线程同步

### 线程的创建 

Java中提供了`Thread` 类，我们通过`Thread`类来开启一个线程。

1. 继承Thread类，实现run方法

   ```java
   class MyThread extends Thread {
       
       @Override
       public void run() {
           System.out.println("worker thread");
       }
   }
   
   // 开启线程
   new MyThread().start();
   ```

2. 直接向线程`Thread` 类构造函数的传入一个实现了`Runnable` 接口的类

   ```java
   new Thread(()->{
       System.out.println("worker thread");
   }, "mythread-1").start();
   ```

   > 这里使用了Lambda表达式，给线程一个名字：mythread-1


创建线程时，我们可以给线程设置优先级。

#### 线程优先级

线程的优先级介于1 (MINPRIORITY)到10 (MAXPRIORITY)之间，主线程默认是5（NORM_PRIORITY）。每个新线程都默认继承父线程的优先级，因此如果你没有设置过的话，所有线程的优先级都是5。 

#### 守护线程

如果进程不需要等待一个线程终止才能结束，则这个线程可以设置为守护线程。对应的是**用户线程**。



可以看到线程的创建与`Thread` 类密不可分，我们通过`Thread`类来看看线程的实现原理。

### 线程的实现原理

上述实现的两种开启线程的方式，本质都是一样的：给`Thread` 类的`target`赋值，在`Thread` 类的`run` 方法中调用`target`的**run**方法。

```java
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

此时的线程，还并未启动，只有调用**start**方法时，才真正启动线程。在Thread类中，start方法调用了native方法：`start0`

```java
private native void start0();
```

Java并不创建和管理线程，而是交给了操作系统去管理。在Android系统中，因为使用的Linux内核，所以线程的创建实现依赖于Linux的`pthread`：其中**/android/art/runtime/native/java_lang_Thread.cc**

![thread_create](.\imgs\thread_create.JPG)



### 线程生命周期

在`Thread`类中定义了线程的生命周期的**6个状态** ：

- NEW：创建状态，线程创建之后，但是还未启动。
- RUNNABLE：运行状态，处于运行状态的线程，但有可能处于等待状态，例如等待CPU、IO等。
- WAITING：等待状态，一般是调用了wait()、join()、LockSupport.spark()等方法。
- TIMED_WAITING：超时等待状态，也就是带时间的等待状态。一般是调用了wait(time)、join(time)、LockSupport.sparkNanos()、LockSupport.sparkUnit()等方法。
- BLOCKED：阻塞状态，等待锁的释放，例如调用了synchronized增加了锁。
- TERMINATED：终止状态，一般是线程完成任务后退出或者异常终止。



### 线程同步

Object类中提供的有关线程同步的方法：

- wait()的作用是让当前线程进入等待状态，同时，wait()也会让当前线程释放它所持有的锁；

- notify()和notifyAll()的作用，则是唤醒当前对象上的等待线程；notify()是唤醒单个线程，而notifyAll()是唤醒所有的线程。

  > 被唤醒的线程也需要获取到锁才可以得到执行

Thread类提供有关线程同步的方法：

- sleep()持有线程锁，进入阻塞状态？Thread类的静态方法；


- yield()是让线程由“运行状态”进入到“就绪状态”，从而让其它具有相同优先级的等待线程获取执行权；但是注意：并不能保证在当前线程调用yield()之后，其它具有相同优先级的线程就一定能获得执行权。

  > yield 是告诉操作系统的调度器：我的cpu可以先让给其他线程。注意，调度器可以不理会这个信息。这个方法几乎没用。


- join()把指定的线程加入到当前线程，可以将两个交替执行的线程合并为顺序执行的线程。

  ```java
  t.join();      //调用jon方法，等待线程t执行完毕
  ```


- interrupt()，如果在线程t2上调用线程t1的interrupt，线程t1的中断状态被设置为true，会影响如下操作:

  > 1.如果线程t1阻塞在wait(), wait(long), wait(long, int), join(), join(long), join(long, int), sleep(long), sleep(long, int)，线程t1被中断后，马上从这些方法中返回，并且会抛出InterruptedException异常；
  >
  > 2.如果线程阻塞在 InterruptibleChannel 类的 IO 操作中，那么这个 channel 会被关闭；
  >
  > 3.如果线程阻塞在一个 Selector 中，那么 select 方法会立即返回。

#### 多线程控制实例——消费者和生产者模型

```java
package com.zgx.test;

import java.util.LinkedList;
import java.util.List;

public class Storage {
    private int MAX = 10;
    private LinkedList list = new LinkedList<Object>();
    private Object lock = new Object();

    public void produce() {
        synchronized (lock) {
            while(list.size() > MAX) {
                System.out.println("the list is full");
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
            list.add(new Object());
            System.out.println("this list is not empty");
            lock.notifyAll();
        }
    }
    
    private void consume() {
        synchronized (lock) {
            while(list.isEmpty()) {
                System.out.println("the list is empty");
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
            Object data = list.remove();
            System.out.println("this list is not full, data=" + data);
            lock.notifyAll();
        }
    }
}
```

通过Conditon的await和signal来同步线程，实现生产者和消费者模型：

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    // condition 依赖于 lock 来产生
    final Condition notFull = lock.newCondition();
    final Condition notEmpty = lock.newCondition();
    final Object[] items = new Object[100];
    int putptr, takeptr, count;
    // 生产者
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();  // 队列已满，等待，直到 not full 才能继续生产
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            notEmpty.signal(); // 生产成功，队列已经 not empty 了，发个通知出去
        } finally {
            lock.unlock();
        }
    }
    // 消费者
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await(); // 队列为空，等待，直到队列 not empty，才能继续消费
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            notFull.signal(); // 被我消费掉一个，队列 not full 了，发个通知出去
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

> 1.condition 是依赖于 ReentrantLock 的，不管是调用 await 进入等待还是 signal 唤醒，都必须获取到锁才能进行操作；
>
> 2.一个 ReentrantLock 实例可以通过多次调用 newCondition() 来产生多个 Condition 实例，上面的例子对应于notFull、notEmpty；
>
> 3.每个 condition 有一个关联的**条件队列**，如线程 1 调用 condition1.await() 方法即可将当前线程 1 包装成 Node 后加入到条件队列中，然后阻塞在这里，不继续往下执行，条件队列是一个单向链表；
>
> 4.调用 condition1.signal() 会将condition1 对应的**条件队列**移到**阻塞队列**的队尾，等待获取锁，获取锁后 await 方法返回，继续往下执行。