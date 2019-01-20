LiveData与Retrofit相结合

LiveData是一个持有数据的类，它具有以下特点：

* 数据可以被观察，调用`obsever`
* 能够感知组件的生命周期，只有组件处于激活状态（可见）时，才会通知观察者数据更新（UI刷新）

在与Retrofit中，内置了可被RxJava观察的数据源转换工厂，即`RxJava2CallAdapterFactory`，仿照该转换工厂实现可被LiveDa事件源转换工厂。

定义Service的API接口，返回`LiveData` 类型：

```kotlin
interface GithubService {
    @GET("users/{login}")
    fun getUser(@Path("login") login: String): LiveData<ApiResponse<User>>
}
```

配置Retrofit：

```kotlin
Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
	// 添加LiveDataCallFactory
    .addCallAdapterFactory(LiveDataCallAdapterFactory())
    .build()
    .create(GithubService::class.java)
```

正常情况下，Retrofit接口返回的是`Call` 对象，为了将`Call` 对象转化为`LiveData` 对象，需要增加**CallAdapter** ，仿照RxJava的实现方式。先实现`LiveDataCallAdapterFactory` 类。

```kotlin
class LiveDataCallAdapterFactory : Factory() {
    override fun get(
        returnType: Type,
        annotations: Array<Annotation>,
        retrofit: Retrofit
    ): CallAdapter<*, *>? {
        if (Factory.getRawType(returnType) != LiveData::class.java) {
            return null
        }
        val observableType = Factory.getParameterUpperBound(0, returnType as ParameterizedType)
        val rawObservableType = Factory.getRawType(observableType)
        if (rawObservableType != ApiResponse::class.java) {
            throw IllegalArgumentException("type must be a resource")
        }
        if (observableType !is ParameterizedType) {
            throw IllegalArgumentException("resource must be parameterized")
        }
        val bodyType = Factory.getParameterUpperBound(0, observableType)
        return LiveDataCallAdapter<Any>(bodyType)
    }
}
```

该类继承自`Factory` 类，是一个工厂类，覆写`get` 方法，返回**LiveDataCallAdapter** 。

```kotlin
class LiveDataCallAdapter<R>(private val responseType: Type) :
    CallAdapter<R, LiveData<ApiResponse<R>>> {

    override fun responseType() = responseType

    // 覆写adapt方法，将call 对象 转化为 LiveData 对象
    override fun adapt(call: Call<R>): LiveData<ApiResponse<R>> {
        return object : LiveData<ApiResponse<R>>() {
            private var started = AtomicBoolean(false)
            // 覆写LiveData的onActive方法，当观察此LiveData的观察者从0变为1时触发
            override fun onActive() {
                super.onActive()
                // 原子操作
                if (started.compareAndSet(false, true)) {
                    call.enqueue(object : Callback<R> {
                        override fun onResponse(call: Call<R>, response: Response<R>) {
                            // 通过ApiResponse进行二次封装
                            postValue(ApiResponse.create(response))
                        }

                        override fun onFailure(call: Call<R>, throwable: Throwable) {
                            postValue(ApiResponse.create(throwable))
                        }
                    })
                }
            }
        }
    }
}
```

