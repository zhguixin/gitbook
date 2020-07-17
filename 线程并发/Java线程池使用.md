---
title: Java线程池使用
date: 2017-6-17 23:42:44
tags: Java 多线程
categories: 
---

[TOC]

我们不可能把事情都放在主线程上执行，这样会造成严重卡顿（ANR），那么这些事情应该交给子线程去做，但对于一个系统而言，创建、销毁、调度线程的过程是需要开销的，所以我们并不能无限量地开启线程，那么对线程的了解就变得尤为重要了。首先了解下Java中的多线程编程。

#### 一、Java中的多线程编程

JDK1.4之前，线程的创建还是依靠Thread、Runnable和Callable（新加入）对象的实例化；Concurrency包出现之后，线程的执行则靠Executor、ExecutorService的对象执行execute()方法或submit()方法；线程的调度则被固化为几个具体的线程池类，如ThreadPoolExecutor、ScheduledThreadPoolExecutor、ExecutorCompletionService等等。这样表面上增加了复杂度，而实际上成功将线程的**创建、执行和调度**的业务逻辑分离，使程序员能够将精力集中在线程中业务逻辑的编写，大大提高了编码效率，降低了出错的概率，而且大大提高了性能。

#### 1、线程的执行者

这个功能主要由三个接口和类提供，分别是： 

- Executor：执行者接口，所有执行者的父类，只包含一个方法：

```jav
void execute(Runnable command);
```

- ExecutorService：执行者服务接口，继承自Executor接口，具体的执行者类都继承自此接口：

```java
public abstract class AbstractExecutorService implements ExecutorService

public class ThreadPoolExecutor extends AbstractExecutorService
//ThreadPoolExecutor的构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue)
```

- Executors：执行者工厂方法类，大部分执行者的实例以及线程池都由它的工厂方法创建。

```java
    ExecutorService service = Executors.newCachedThreadPool();
    service.execute(new Task1());//Task1是Runnable接口的实现类
    service.shutdown();//阻止继续提交其他线程，并等待执行中的线程结束

    class Task1 implements Runnable{
        @Override
        public void run() {
            // TODO Auto-generated method stub
            System.out.println("task1");
    }
```

如果要改变默认的线程创建方式，自定义`ThreadFactory`传入：

```java
Executors.newFixedThreadPool(2, new ThreadFactory() {

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setDaemon(true);
        return t;
    }
});
```



##### 2、开启线程的方式

- 继承Thread类，实现run方法；
- 实现Runnable接口
- 实现Callable接口，可以得到异步线程的执行结果

```java
/**
在Java Concurrency中，得到异步结果有了一套固定的机制，即通过Callable接口、Future接口和ExecutorService的submit方法来得到异步执行的结果
Callable：泛型接口，与Runnable接口类似，它的实例可以被另一个线程执行，内部有一个call方法，返回一个泛型变量V;
Future：泛型接口，代表依次异步执行的结果值，调用其get方法可以得到一次异步执行的结果，如果运算未完成，则阻塞直到完成；调用其cancel方法可以取消一次异步执行；
**/
     ExecutorService service = Executors.newCachedThreadPool();
        //
        Future<Integer> future = service.submit(new Callable<Integer>() {

            @Override
            public Integer call() throws Exception {
                // TODO Auto-generated method stub
                return 11;
            }
        });
        service.shutdown();
        try {
            //获取异步执行的结果，并捕获异常
            System.out.println("the value is=" + future.get());
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } catch (ExecutionException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
```

关于`ExecutorService`的`submit`和`execute`的区别：

- `submit`可以接收更多的参数，`execute`只能接收Runnable的实例
- `submit`有返回值，而`execute`没有
- `submit`方便Exception处理

#### 3、线程的执行控制

开发过程中会有对线程，延期执行或重复执行的需求。这个时候Java Concurrency包中的ScheduledExecutorService就派上了用场。

```java
/**
ScheduledExecutorService：可以将提交的任务延期执行，也可以将提交的任务反复执行。 
ScheduledFuture：与Future接口类似，代表一个被调度执行的异步任务的返回值。 
**/
ScheduledExecutorService service2 = new ScheduledThreadPoolExecutor(2);
ScheduledFuture<?> future2 = service2.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
           System.out.println("test");
       }}, 1, 1, TimeUnit.SECONDS);//延迟1秒执行，然后每隔1秒打印一次”test”
        
    service2.schedule(new Runnable() {
        @Override
        public void run() {
            System.out.println("cancel task");
            future2.cancel(true);
            service2.shutdown();
        }
    }, 10, TimeUnit.SECONDS);//延迟10秒执行，取消打印任务
```

其中，`ScheduledThreadPoolExecutor`类继承关系如下：

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService
        
public interface ScheduledExecutorService extends ExecutorService

public interface ScheduledFuture<V> extends Delayed, Future<V>
```

#### 4、Fork-Join框架

在JDK1.7引入了一种新的并行编程模式“fork-join”，它是实现了“分而治之”思想的Java并发编程框架。
