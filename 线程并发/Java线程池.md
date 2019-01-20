### Java线程池

线程池的使用可以避免重复创建线程和销毁线程带来的CPU开销，线程池可以重复利用线程池中的线程。

#### 1、简介

线程池的实现借助于`ThreadPoolExecutor`来实现，对于线程池来说它有**五种**运行状态：

* RUNNING：线程池处在这个运行态时，可以接受新任务和处理队列中的任务
* SHUTDOW：不再接受新任务，但会处理队列中的任务
* STOP：不再接受新任务，不再处理队列中的任务，中断正在处理的任务
* TIDYING：所有的任务都终止，已经没有有效线程，马上调用`terminated()`方法
* TERMINATED：调用了`terminated()`方法，线程池已终止

ThreadPoolExecutor是用一个AtomicInteger来记录线程池状态和线程池里的线程数量的：

- 低29位：用来存放线程数
- 高3位：用来存放线程池状态



基本使用：

```java
ExecutorService cachedThreadPool = Executors.newCachedThreadPool();

//调用execute方法
cachedThreadPool.execute(new Runnable() {
  public void run() {  
    System.out.println("running task via execute");  
  }  
});

//Lambda表达式的写法更加简洁
cachedThreadPool.execute(() -> System.out.println("running task via execute"));

//调用submit方法
cachedThreadPool.submit(new Runnable() {
  public void run() {  
    System.out.println("running task via submit");  
  }  
});
```

Java通过`Executors`这个工具类提供了四种线程池，分别为：

* **newCachedThreadPool** 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程（60s不活动则自动终止），若无可回收，则新建线程。适用于生命 周期比较短的异步任务。
*  **newFixedThreadPool** 创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。适用于很稳定、很正规的并发线程，多用于服务器。
*  **newScheduledThreadPool** 创建一个定长线程池，支持定时及周期性任务执行。
*  **newSingleThreadExecutor** 创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行

以上四种线程池分别对`ThreadPoolExecutor`填充了不同的参数，实现不同的功能。

#### 2、原理分析

接口：Executor、ExecutorService；

```java
public interface Executor {
  void execute(Runnable command);
}

//定义了操作线程池的一些常用方法
public interface ExecutorService extends Executor {
  void shutdown();
  // ...
  <T> Future<T> submit(Callable<T> task);
  <T> Future<T> submit(Runnable task, T result);
  Future<?> submit(Runnable task);
  // ...
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
  // ...
}
```

类：AbstractExecutorService、ThreadPoolExecutor

```java
//实现了接口中的一些方法
public abstract class AbstractExecutorService implements ExecutorService{
  
}

//线程池的真正实现类
public class ThreadPoolExecutor extends AbstractExecutorService {
 
  	//任务队列，BlockingQueue 接口的某个实现（常使用 ArrayBlockingQueue 和 LinkedBlockingQueue），由TheadPoolExecutor构造函数传入，线程池从该队列中获取要执行的任务(调用getTask方法)
	private final BlockingQueue<Runnable> workQueue;
	//保存线程池中的Worker，一个Worker对应会开启一个线程，也就是会记录线程池中的线程数
	private final HashSet<Worker> workers = new HashSet<Worker>();
    
    // 构造函数
    public ThreadPoolExecutor(int corePoolSize,// 核心线程数
                              int maximumPoolSize,// 线程池最大线程数
                              long keepAliveTime,// 非核心线程池闲置超时时长
                              TimeUnit unit,
                              // 阻塞队列。核心线程满时，任务在此队列等待，不同的队列表现不一样
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,// 用户指定的线程创建工厂
                              // 发生异常时的捕获处理
                              RejectedExecutionHandler handler) {
    }
}
```

> 阻塞队列主要有如下四种类型：（任务添加到阻塞队列的前提都是线程池线程数已超过核心线程数）
>
> SynchronousQueue，同步队列，该队列的任务接收到任务时，不保存任务，直接交给线程处理
>
> LinkedBlockingQueue，添加到队列，等待核心线程空闲，并且没有队列上限制
>
> ArrayBlockingQueue，可以设置长度，当达到数组长度时，再创建线程处理
>
> DelayQueue，延迟队列，实现Delayed接口，指定的延迟时间到达后才去执行

工厂类：Executors，提供已定义好的一些线程池（直接调用相关静态方法），内部通过调用ThreadPoolExecutor实现；

辅助类：FutureTask，通过submit交给线程池的任务都会包装成此类。

```java
//定义获取异步执行结果的方法
public interface Future<V> {
  boolean isCancelled();
  boolean isDone();
  V get() throws InterruptedException, ExecutionException;
  V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

//接口RunnableFuture，继承了Runnable和Future接口
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}

//实现了Runnable和Future接口
public class FutureTask<V> implements RunnableFuture<V> {
  
}
```



我们调用ThreadPoolExecutor的execute方法，执行过程如下：

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        // 当前线程数小于corePoolSize时，直接添加一个 worker 来执行任务，调用addWorker方法；
        if (workerCountOf(c) < corePoolSize) {
            // 核心线程
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 将任务加入阻塞队列，即使加入成功也要在进行一次检查操作，防止在此期间线程池关闭
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 线程池在运行，但是没有活动的线程，再调用addWorker创建线程从队列执行任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        } // 如果任务不能添加到队列，调用addWorker创建线程，去处理队列中的任务
        else if (!addWorker(command, false))
            reject(command);
    }
```

在`addWorker`方法中，真正创建线程是在**Worker**（ThreadPoolExecutor的内部类）的构造方法中调用 ThreadFactory 来创建一个新的线程，把`command`包装为Worker之后，加入到workers队列中。待创建的线程start之后，调用到Worker类的run方法，该法直接调用了ThreadPoolExecutor的runWorker方法。在runWorker方法中，就直接调用commad的run方法了，这个run方法就已经运行在刚才创建的线程里了。

如果我么调用线程的submit方法，直接调用到的是`AbstractExecutorService`的submit方法，执行过程如下：

```java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);//包装成FutureTask后，调用ThreadPoolExecutor的execute方法
        return ftask;
    }

	//newTaskFor的定义如下
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```

submit的几个重载方法定义如下：

```java
    //如果想要获得，异步线程的执行结果，传入result参数，执行结果将放在这个变量中
	public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

	//submit还可以传入Callable的接口，
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

	//Callable的接口定义如下：
    public interface Callable<V> {
     //它和 Runnable 的区别在于 run() 没有返回值，而 Callable 的 call() 方法有返回值，同时，如果运行出现异常，call() 方法会抛出异常。
      V call() throws Exception;
  }
```

虽然submit可以接受不同的传入参数，但是对于ThreadPoolExecutor来说，并没有区别，最终都包装成了FutureTask。

##### 关于executor和submit的区别：

- 接收的参数不一样，executor只能接收Runnable；

- submit有返回值（submit接收Runnable的任务也没有返回值），而execute没有；

- submit方便Exception处理，如果在task里会抛出checked或者unchecked exception

  调用者可以通过future.get()进行捕获。



#### 2、阻塞队列

在线程池中，当线程数量超过corePoolSize时，新添加的任务会被添加到阻塞队列中去，这个阻塞队列由用户定义传入。让我们先了解下阻塞队列。

阻塞队列（BlockingQueue） 对插入操作、移除操作、获取元素操作提供了四种不同的方法用于不同的场景中使用：1、抛出异常；2、返回特殊值（null 或 true/false，取决于具体的操作）；3、阻塞等待此操作，直到这个操作成功；4、阻塞等待此操作，直到成功或者超时指定时间。总结如下：

|             | *Throws exception* | *Special value* | *Blocks*         | *Times out*          |
| ----------- | ------------------ | --------------- | ---------------- | -------------------- |
| **Insert**  | add(e)             | offer(e)        | **put(e)**       | offer(e, time, unit) |
| **Remove**  | remove()           | poll()          | **take()**       | poll(time, unit)     |
| **Examine** | element()          | peek()          | *not applicable* | *not applicable*     |

参考文章：

[阻塞队列详解](https://blog.csdn.net/silyvin/article/details/79482885)