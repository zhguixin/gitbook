---
title: Java并发类的使用
date: 2017-09-16 22:47:44
tags: Java 多线程
categories: 
---

[TOC]

#### 1、CountDownLatch的使用

经典使用场景：假设开启N个线程去处理任务，每个线程通过调用CountDownLatch的await() 进行阻塞等待，这些线程将会阻塞在**栅栏**上，只有当条件满足的时候（通过调用CountDownLatch的countDown()来取消阻塞），它们才能同时通过这个栅栏。

```java
class TestCountDownLatch { // ...
    void main() throws InterruptedException {
        // 同时开始任务
        CountDownLatch startSignal = new CountDownLatch(1);
        // 同时结束任务
        CountDownLatch doneSignal = new CountDownLatch(N);
 
        for (int i = 0; i < N; ++i) // create and start threads
            new Thread(new Worker(startSignal, doneSignal)).start();
        // 这边插入一些代码，确保上面的每个线程先启动起来，才执行下面的代码。
        doSomethingElse();            // don't let run yet
        // 因为这里 N == 1，所以，只要调用一次，那么所有的 await 方法都可以通过
        startSignal.countDown();      // let all threads proceed
        doSomethingElse();
        // 等待所有任务结束
        doneSignal.await();           // wait for all to finish
    }
}
class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;
    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }
    public void run() {
        try {
            // 为了让所有线程同时开始任务，我们让所有线程先阻塞在这里
            // 等大家都准备好了，再打开这个门栓
            startSignal.await();
            doWork();
            doneSignal.countDown();//
        } catch (InterruptedException ex) {
        } // return;
    }
    void doWork() { ...}
}
```

#### 2、CyclicBarrier的使用

CyclicBarrier与CountDownLatch作用一样，只是存在以下差别：

- CountDownLatch的计数器只能使用一次。而CyclicBarrier的计数器可以使用reset() 方法重置。所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。
- CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得CyclicBarrier阻塞的线程数量，isBroken方法用来知道阻塞的线程是否被中断。

#### 3、Semaphore的使用

Semaphore（信号量）也是一个线程同步的辅助类，可以维护当前访问自身的线程个数，并提供了同步机制。使用Semaphore可以控制同时访问资源的线程个数，例如，实现一个文件允许的并发访问数。

**Semaphore的主要方法摘要：**

- Semaphore(int permits)：构造函数，指定同时访问的线程个数permits


- void acquire()：从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中
- void release()：释放一个许可，将其返回给信号量
- int availablePermits()：返回此信号量中当前可用的许可数
- boolean hasQueuedThreads()：查询是否有线程正在等待获取