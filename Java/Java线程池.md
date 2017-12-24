### Java线程池

#### 1、简介

线程池的使用可以避免重复创建线程和销毁线程带来的CPU开销，线程池可以重复利用线程池中的线程。

基本使用：

```java
ExecutorService cachedThreadPool = Executors.newCachedThreadPool();

//调用execute方法
cachedThreadPool.execute(new Runnable() {
  public void run() {  
    System.out.println("running task via execute");  
  }  
});

//调用submit方法
cachedThreadPool.submit(new Runnable() {
  public void run() {  
    System.out.println("running task via submit");  
  }  
});
```

#### 2、原理分析

接口：Executor、ExecutorService；

```java
public interface Executor {
  void execute(Runnable command);
}

//定义了操作线程池的一些常用方法
public interface ExecutorService extends Executor{
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
//实现了一些方法
public abstract class AbstractExecutorService implements ExecutorService{
  
}

//线程池的真正实现类
public class ThreadPoolExecutor extends AbstractExecutorService {
  
  	//任务队列，BlockingQueue 接口的某个实现（常使用 ArrayBlockingQueue 和 LinkedBlockingQueue），由TheadPoolExecutor构造函数传入，线程池从该队列中获取要执行的任务(调用getTask方法)
	private final BlockingQueue<Runnable> workQueue;
	//保存线程池中的Worker，一个Worker对应会开启一个线程，也就是会记录线程池中的线程数
	private final HashSet<Worker> workers = new HashSet<Worker>();
}
```

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
        /*
         1.当前线程数小于corePoolSize时，直接添加一个 worker 来执行任务，调用addWorker方法；
         2.如果线程池处于 RUNNING 状态，把这个任务添加到任务队列 workQueue 中；
         3.如果 workQueue 队列满了，以 maximumPoolSize 为界创建新的 worker。
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

在addWorker方法中，真正创建线程是在Worker（ThreadPoolExecutor的内部类）的构造方法中调用 ThreadFactory 来创建一个新的线程，把command包装为Worker之后，加入到workers队列中。待创建的线程start之后，调用到Worker类的run方法，该法直接调用了ThreadPoolExecutor的runWorker方法。在runWorker方法中，就直接调用commad的run方法了，这个run方法就已经运行在刚才创建的线程里了。

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

关于executor和submit的区别：

- 接收的参数不一样，executor只能接收Runnable；

- submit有返回值（submit接收Runnable的任务也没有返回值），而execute没有；

- submit方便Exception处理，如果在task里会抛出checked或者unchecked exception

  调用者可以通过future.get()进行捕获。