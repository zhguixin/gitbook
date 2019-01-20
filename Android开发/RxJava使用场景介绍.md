RXJava使用场景介绍

### RxJava基本操作

RxJava提供按功能不同提供了各种操作符：创建操作符、功能操作符、组合(合并）操作符、变换操作符。

#### 创建操作符

- 基础创建：`create()` ，它允许为每个订阅者精确控制事件的发送

  ```java
  public Observable<String> valueObservable() {
    return Observable.create(new Observable.OnSubscribe<String>() {
      @Override
      public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext(value);
        subscriber.onCompleted();
      }
    });
  }
  ```

- 快速创建：`jsut()` 、`fromArray()`，`fromIterable()` 

  ```java
  String[] arrs = {"A", "B", "C", "D"};

  Observable.just(arrs)
      .subscribe(s -> System.out.println(s)); // s 是String[] 类型的

  Observable.fromArray(arrs)
      .subscribe(s -> System.out.println(s)); // s 是String 类型的，分别输出 A/B/C/D
  ```

- 延迟创建：`defer()` ，只有观察者订阅后，才动态的创建事件源，确保观察者拿到的事件是最新的；

  ```java
  Observable<Integer> observable = Observable.defer(new Callable<ObservableSource<? extends Integer>>() {
              @Override
              public ObservableSource<? extends Integer> call() throws Exception {
                  return Observable.just(i);
              }
  });
  ```

  `timer()` ，延迟指定的时间，发送一个0。兼顾创建和发送事件 ，不会阻塞主线程（这是异步的）

  ```java
  Observable.timer(2, TimeUnit.SECONDS) 
  ```

  `interval`，每间隔一段时间发送一个事件，事件序列为从0开始、无限递增1的的整数序列

  `intervalRange`，可以限定事件数

  ```java
  // 参数1 = 第1次延迟时间；
  // 参数2 = 间隔时间；
  // 参数3 = 时间单位；
  Observable.interval(3,1,TimeUnit.SECONDS)
      
  // 参数1 = 事件序列起始点；
  // 参数2 = 事件数量；
  // 参数3 = 第1次事件延迟发送时间；
  // 参数4 = 间隔时间数字；
  // 参数5 = 时间单位
  Observable.intervalRange(3,10,2, 1, TimeUnit.SECONDS)

  ```

#### 功能操作符

- delay()，事件源延迟一段时间在进行事件发送

  ```java
  Observable.just(1, 2, 3)
  	.delay(3, TimeUnit.SECONDS) // 延迟3s再发送
  ```


- do操作符，比如`doOnNext()` 、`doAfterNext()`，分别在观察者的OnNext之前和之后调用

#### 组合/合并操作符

将各个事件源的事件进行组合或者合并，一起发送给观察者。

- 按发送事件组合：` contact()`
- 按发送事件组合：`merge()`
- 合并多个事件： `zip()` 

#### 变换操作符

- map()，将事件源的每一个事件都通过指定的函数进行变换处理，转变为另一种事件传递给观察者

  ```java
  List<String> arrs = Arrays.asList("A", "B", "C", "D");
  
  Observable.just(arrs)
      .map(list -> list.subList(1, 3)) // 返回值是一个 list
      .subscribe(s -> System.out.println(s));
  // 输出 
  // [B,C]
  ```

- flatMap()，将事件源中的每个事件单独拆分**分别**处理后，再**合并**为新的事件序列进行发送(此时新的事件序列顺序与旧的事件序列顺序没有关系)，flatMap返回的值是**Observable**类型

  ```java
  List<String> arrs = Arrays.asList("A", "B", "C", "D");
  // just 直接发送整个list集合，可以通过flatMap对list集合进行迭代输出
  Observable.just(arrs)
      .flatMap(list -> Observable.fromIterable(list)) // 返回值类型是 Observable 类型
      .subscribe(s -> System.out.println(s));
  // 输出
  // A
  // B
  // C
  // D
  ```

- concatMap()，不同于flatMap，该变换保持旧事件序列的顺序

### RxJava使用

#### 1、网络请求出错重连

结合`Retrofit` ，主要借助于observable 的 `retryWhen`方法。可以指定出错重试的次数，以及出错抛出异常的类型来进行重试。

#### 2、网络请求轮询

- 无限次轮询，调用`interval`方法，指定时间间隔后，每一个时间间隔内触发`doOnNext(new Consumer<Long>())` 方法，执行轮询时的操作。

  *覆写 Consumer 接口的accept方法进行每次轮询时的具体业务实现*

- 有限次的轮询，调用`intervalRange`方法。通`interval`方法。

#### 3、有条件的网络请求轮询

借助于`repeatWhen` 方法。

#### 4、嵌套网络请求

再进行了一次网络请求后，紧接着进行下一次网络请求。比如，用户注册完成后进行登录操作。

主要借助于**flatMap**这个函数，进行事件的切换。

#### 5、多级缓存功能实现

借助于`firstElement()` ，`contact()`两个操作符对不同的事件源进行合并。

```java
// 1. 通过concat（）合并memory、disk、network 3个被观察者的事件（即检查内存缓存、磁盘缓存 & 发送网络请求）
//    并将它们按顺序串联成队列
Observable.concat(memory, disk, network)
        // 2. 通过firstElement()，从串联队列中取出并发送第1个有效事件（Next事件），即依次判断检查memory、disk、network
        .firstElement()
        // 即本例的逻辑为：
        // a. firstElement()取出第1个事件 = memory，即先判断内存缓存中有无数据缓存；由于memoryCache = null，即内存缓存中无数据，所以发送结束事件（视为无效事件）
        // b. firstElement()继续取出第2个事件 = disk，即判断磁盘缓存中有无数据缓存：由于diskCache ≠ null，即磁盘缓存中有数据，所以发送Next事件（有效事件）
        // c. 即firstElement()已发出第1个有效事件（disk事件），所以停止判断。
        
        // 3. 观察者订阅
        .subscribe(new Consumer<String>() {
            @Override
            public void accept( String s) throws Exception {
                Log.d(TAG,"最终获取的数据来源 =  "+ s);
            }
        });
```

#### 6、表单验证

基本思路是将每一个EditText作为事件源：

```java
/*
* 说明：
* 1. 此处采用了RxBinding：RxTextView.textChanges(name) = 对对控件数据变更进行监听（功能类似TextWatcher），需要引入依赖：compile 'com.jakewharton.rxbinding2:rxbinding:2.0.0'
 * 2. 传入EditText控件，点击任1个EditText撰写时，都会发送数据事件 = Function3（）的返回值（下面会详细说明）
* 3. 采用skip(1)原因：跳过 一开始EditText无任何输入时的空值
**/
Observable<CharSequence> nameObservable = RxTextView.textChanges(name).skip(1);

/*
* 步骤3：通过combineLatest（）合并事件 & 联合判断
**/
Observable.combineLatest(nameObservable,ageObservable,jobObservable,new Function3<CharSequence,
                         CharSequence, CharSequence,Boolean>() {
            @Override
            public Boolean apply(@NonNull CharSequence charSequence, @NonNull CharSequence
                           charSequence2, @NonNull CharSequence charSequence3) throws Exception {
                // 具体验证操作
            }
});
```

#### 7、防抖功能

```java
Observable.throttleFirst(2, TimeUnit.SECONDS)  // 2s内只发送一次事件
    .subscribe(new Observer<Object>() {};
```

#### 8、优化搜索请求

在`EditText` 中，根据输入值实时的发送网络请求来搜索结果。可以利用RxBinding的`debounce` 操作符来延迟一段时间再发送请求：

```kotlin
RxTextView.textChanges(editview)
    .debounce(500, TimeUnit.MILLISECONDS, Schedulers.io())
    .switchMap { searchApi.search(it.toString()) }
	.observeOn(AndroidSchedulers.mainThread())
    .subscribe { updateUI() }
```

其中，`debounce` 操作符确保延迟500ms在触发后续操作，并且将`switchMap` 操作符中的操作放到工作线程；

`switchMap` 操作符，将会接收最新的一个事件。

#### 避免RxJava内存泄漏

对于放在Activity内的周期性的操作，应该在`onDestroy` 中解除订阅。

