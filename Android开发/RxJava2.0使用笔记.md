RxJava2.0使用笔记

最近项目迁移到RxJava2.0，简单记下使用过程。

使用过程中发现，2.0和1.0差别还是挺大的。从一个简单的栗子看起：

```java
  Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> e) throws Exception {
      e.onNext("hello");
      e.onNext("world");
      e.onComplete();
    }
  }).subscribe(new Observer<String>() { // Observer是一个接口，要实现所有方法
    @Override
    public void onSubscribe(Disposable d) { // onSubscribe的回调有参数Disposable
      d.dispose();	// 调用该方法可停止接收 Observable发送的事件
    }

    @Override
    public void onNext(String s) {
      System.out.println("get data=" + s);
    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onComplete() {

    }
  });
```

和Rxjava1.0的用法一样，多了个`onSubscribe的回调`。我们可以同样简化一下：

```java
  Disposable disposable = Observable.just("hello").subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
      System.out.println("get data=" + s);
    }
  });

  // 合适的事件断开连接，停止接收Observable(被观察者)发送的事件
  // Disposable 由 Observable的subscribe()方法传入接口Consumer时返回
  disposable.dispose();
```

其中，`Consumer`也是一个接口，用来接收单个消息。

Observable 和 Observer的使用方式和RxJava1.0差不多。在RxJava2.0中引入了`Flowable`用来处理背压。

先看个栗子：

```java
  Disposable disposable = Flowable.create(new FlowableOnSubscribe<String>() {
    @Override
    public void subscribe(FlowableEmitter<String> e) throws Exception {
      System.out.println("observable thread:" + Thread.currentThread().getName());
      e.onNext("hello");
      e.onComplete();
    }
  }, BackpressureStrategy.DROP) // create的第二个参数指定背压策略
    .compose(new FlowableTransformer<String,String>() { // compose方法，做线程切换
      @Override
      public Flowable<String> apply(Flowable<String> observable) {
        return observable.subscribeOn(Schedulers.io())
          .observeOn(AndroidSchedulers.mainThread());
      }
    })
    // Flowable有对应的Subscriber接口，和Observer一样。为简单起见，此处调用接收单个事件的Consumer接口
    .subscribe(new Consumer<String>() { 
      @Override
      public void accept(String o) throws Exception {
        System.out.println("observer thread:" + Thread.currentThread().getName());
        System.out.println("get the data: " + o);
      }
    });

  // 合适的时机断开连接，停止接收Flowable(被观察者)发送的事件
  disposable.dispose();
```

所谓背压，就是当生产者所生产的数据超过消费者所能消费的能力时，大量数据在生产者内存中堆积，导致OOM。背压的现象**只出现在异步**的情况下的。

在RxJava1.0中，背压策略在Observable中指定，到了RxJava2.0放到了新类`Flowable`中了。

当出现背压的情况，事件源先将事件放到缓存区(默认大小为128)，观察者可以通过`Subscription.request(long n)` 指定可接收的事件数：

```java
.subscribe(new Subscriber<Integer>() {
    @Override
    public void onSubscribe(Subscription s) {
        Log.d(TAG, "onSubscribe");
        s.request(150); // 观察者接收15个事件
    }
	// ...onNext()、onError()
}

```

除了在构造事件源时指定背压策略，`RxJava 2.0`内部提供 封装了背压策略模式的方法：

- `onBackpressureBuffer()`
- `onBackpressureDrop()`
- `onBackpressureLatest()`

> 默认采用`BackpressureStrategy.ERROR`模式

```java
Flowable.interval(1, TimeUnit.MILLISECONDS)
                .onBackpressureBuffer()
    // ...
```

