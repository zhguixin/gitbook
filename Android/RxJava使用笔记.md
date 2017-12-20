### RxJava使用笔记

####简单使用

RxJava，就是观察者模式的一种实现。涉及到的概念有：

- Observable，数据源，英文被译为“被观察者”，发射数据的源头；

- Observer，接收器，英文翻译为“观察者”，可接收`Observable`数据源发送过来的数据；

- Subscriber，“订阅者”，也是一个接收器，可接收`Observable`数据源发送过来的数据；

  >它跟Observer有什么区别呢？`Subscriber`实现了`Observer`接口，比Observer多了一个最重要的方法unsubscribe( )，用来取消订阅，当你不再想接收数据了，可以调用`unsubscribe( )`方法停止接收，Observer 在 subscribe() 过程中,最终也会被转换成 Subscriber 对象，一般情况下，建议使用Subscriber作为接收源

- Subscription，这个是`Observable`调用`subscribe()`方法返回的对象，该对象同样有`unsubscribe()`方法用来取消订阅事件；



一个简单的栗子（RxJava2会有变动，跑不通）：

```java
// 创建数据源
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("world");
        subscriber.onCompleted();
    }
});

// 创建接收器
Subscriber<String> subscriber = new Subscriber<String>() {
  @Override
  public void onSubscribe(Subscription s) {

  }

  @Override
  public void onNext(String s) {
    // 直接打印接收到的数据
    System.out.print(s);
  }

  @Override
  public void onError(Throwable t) {

  }

  @Override
  public void onComplete() {

  }
};

// 关联数据源 和 接收器
observable.subscribe(subscriber);

// 还记得 subscribe 的返回的对象为Subscription 吧，我们拿到返回值，来使用完成后取消订阅
Subscription subscription = observable.subscribe(subscriber);
subscription.unsubscribe();
```

以上代码只为理清思路，我们看看简化后的代码：

```java
Observable.just("hello", "world")
    .subscribe(new Action1<String>() {
        @Override
        public void call(String str) {
            System.out.println("str = " + str);
        }
    });
```

这里有几个知识点：

- `just()`方法会创建一个`Observable`对象，自动调用`onNext()`进行数据发送；
- `subscribe()`方法有多个重载函数，这里传入`Action1`对象，是 RxJava 的一个接口，它只有一个方法 `call()`， 该对象在这里的作用是将 `onNext(obj)` 和 `onError(error)` 打包起来传入`subscribe()` ，所以在`call()` 方法里我们可以打印数据源发送过来的数据；

> 简单说一下，RxJava2的不同：
>
> 在RxJava2中，数据源有两个类，一个Observable，一个是Flowable，区别在于Flowable能处理背压，而Obserable没有处理背压的能力。
>
> 接收器也有两个，一个是Observer，适用于Observable。一个是Subscriber，适用于Flowable。
>
> **RxJava中Subscriber是实现了Observer接口的一个抽象类，RxJava2则是两个不相关的接口**

参考资料：[给Android开发者的RxJava详解](http://gank.io/post/560e15be2dca930e00da1083#toc_27)

#### RxJava常见的使用场景

RxJava的几个使用场景，附代码：

```java
/******************************************
 * RxJava 简单示例
******************************************/
Random random = new Random();

Observable.create(emit -> { // the observable implementation is a thread that emits random integer 

    new Thread(() -> {
        while (true) {
            emit.onNext(random.nextInt(40));
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();

}, Emitter.BackpressureMode.LATEST)
.subscribe(o -> System.out.println(o.toString())); // subscriber to the obserable

/************************************************
 * RxJava的变换操作示例：
 * 扫描某一目录下所有以.png结尾的图片，转换为Bitmap
************************************************/
Observable.from(folders)
    .flatMap(new Func1<File, Observable<File>>() { // flatMap进行一对多的转换
        @Override
        public Observable<File> call(File file) {
            return Observable.from(file.listFiles());
        }
    })
    .filter(new Func1<File, Boolean>() {
        @Override
        public Boolean call(File file) {
            return file.getName().endsWith(".png");
        }
    })
  	/*Func1 和 Action 的区别在于， Func1 包装的是有返回值的方法;
  	 map进行一对一的变换操作，通过Func1这个类，从File指定的路径中读取转换为Bitmap对象*/
    .map(new Func1<File, Bitmap>() {
        @Override
        public Bitmap call(File file) {
            return getBitmapFromFile(file);
        }
    })
    .subscribeOn(Schedulers.io()) // 指定事件产生的线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定事件消费的线程
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) {
            imageCollectorView.addImage(bitmap);
        }
    });

/******************************************
 * 打印 list 数组中的内容
******************************************/
List<String> list = Arrays.asList("One", "Two", "Three", "Four", "Five",null);
for (String str :list) {
    System.out.println("str = " + str);
}

Observable.from(list).subscribe(System.out::println);
Observable.from(list).subscribe(str -> System.out.println("str = " + str));

Observable.from(list)
    .subscribe(new Action1<String>() {
        @Override
        public void call(String str) {
            System.out.println("str = " + str);
        }
    });

/****************************************************
 * RxJava的线程切换示例：
 * 在IO线程对drawable进行处理，在UI线程对drawable进行显示
*****************************************************/
// 创建Observable时创建了一个OnSubscribe的对象，这个OnSubscribe对象会存储在ObServable对象中，
// 当Observable被订阅的时候，OnSubscribe的call方法会被自动调用，并依次调用事件序列。
Observable.create(new Observable.OnSubscribe<Drawable>() {
    @Override
    public void call(Subscriber<Drawable> subscriber) {
        // 此处运行在IO线程，可以对drawable进行耗时处理
        Drawable drawable = getTheme().getDrawable();
        subscriber.onNext(drawable);
        subscriber.onCompleted();
    }
})
/* subscribeOn进行的线程切换发生在OnSubscribe之前的操作，可以调用多次，即使多次调用，只有第一个起作用;指定Observable.OnSubscribe 被激活时所处的线程,或者叫做事件产生的线程 */
.subscribeOn(Schedulers.io)
/* observeOn() 指定的是它之后的操作所在的线程，可以执行多次;指定 Subscriber 所运行在的线程,或者叫做事件消费的线程 */
.observeOn(AndroidSchedulers.mainThread())
.subscribe(new Subscriber<Drawable>() { // 这里也可以使用Observer，使用Observer的话，也总是会先被转换成一个 Subscriber 再使用
    @Override
    public void onNext(Drawable drawable) {
        imageView.setImageDrawable(drawable);
    }

    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }
});
```

#### RxJava与Retrofit联合使用

```java
Retrofit retrofit = new Retrofit.Builder()
  .baseUrl("https://github.com/")
  .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
  .build();
```

通过这句话`.addCallAdapterFactory(RxJavaCallAdapterFactory.create())`，RxJava就和Retrofit完美的关联在了一起。

我们在定义网络访问请求接口时，返回值直接返回Observable：

```java
public interface IUser {
  @GET("users/{user}/followers")
  Observable<List<UserBean>> followers(@Path("user") String usr);
}
```

最后拿到该接口：

```java
IUser userService = retrofit.create(IUser.class);
Observable<List<UserBean>> mObserver = userService.followers("zhguixin");
```

拿到了Observable，就可以通过RxJava的流式操作了。