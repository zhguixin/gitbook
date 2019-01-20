Glide图片加载框架使用

简单使用：

```java
Glide.with(mContext).load(mUrl).into(imgView);
```

`into`方法，还有一个重载方法：*(GenericRequestBuilder)*

```java
public <Y extends Target<TranscodeType>> Y into(Y target)
```

使用方法如下：

```java
  Glide.with(mContext)
    .load(mUrl)
    .into(new GlideDrawableImageViewTarget(image) {
      @Override
      public void onLoadStarted(Drawable placeholder) {
        super.onLoadStarted(placeholder);
        // 图片开始加载的时候，增加处理逻辑
      }

      @Override
      public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> animation) {
        super.onResourceReady(resource, animation);
        // 图片加载完毕，可以 对resource进行任何操作，这里简单的加载到ImageView中
        imgView.setImageDrawable(resource);
      }
    });
```



配置Glide，自定义模块的使用，定义一个我们自己的模块类，并让它实现GlideModule接口： 

```java
public class MyGlideModule implements GlideModule {
    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
      // Glide加载图片的默认格式是RGB_565，我们在这里更改为ARGB_8888
      builder.setDecodeFormat(DecodeFormat.PREFER_ARGB_8888);
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
    }
}
```

并在AndroidMaifest的Application标签中配置`meta-data`：

```xml
  <application>

    <meta-data
               android:name="com.example.myapplication.MyGlideModule"
               android:value="GlideModule" />

    ...

  </application>
```

如此一来，我们就将Glide默认加载图片的格式从`RGB_565`更改为`RGB_8888`了。

Glide支持自定义的配置选项及默认配置如下：

- setMemoryCache() 
  用于配置Glide的内存缓存策略，默认配置是LruResourceCache。
- setBitmapPool() 
  用于配置Glide的Bitmap缓存池，默认配置是LruBitmapPool。
- setDiskCache() 
  用于配置Glide的硬盘缓存策略，默认配置是InternalCacheDiskCacheFactory。
- setDiskCacheService() 
  用于配置Glide读取缓存中图片的异步执行器，默认配置是FifoPriorityThreadPoolExecutor，也就是先入先出原则。
- setResizeService() 
  用于配置Glide读取非缓存中图片的异步执行器，默认配置也是FifoPriorityThreadPoolExecutor。
- setDecodeFormat() 
  用于配置Glide加载图片的解码模式，默认配置是RGB_565。

以上的配置选项，我们都可以通过在`applyOptions`方法通过builder进行配置。



更改Glide的组件

借助于`registerComponents`这个方法，我们可以自定义组件来替换Glide的一些默认组件。

比如，Glide在加载图片时，是通过**HttpURLConnection**的方式，将url变换为InputStream的方式，最终显示出来。如果希望在加载大图的时候，想要加载的同时显示加载进度，可以通过配置Glide的组件：用OkHttp替换HttpURLConnection，借助于OkHttp强大的拦截器机制来显示加载进度。



#### 图片下载显示

在`into` 方法中，构造好请求后，发起请求。会调用到Request的begin方法，即调用到GenericRequest的`begin` 方法：

```java
public void begin() {
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
    }
    // ...
}

// onSizeReady方法
public void onSizeReady(int width, int height) {
    // ...
    ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
    final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
}
```

其中，`loadProvider` 在GenericRequest的构造函数中传递过来，赋值的地方在**DrawableTypeRequest** 构造函数中，通过`buildProvider` 方法构造：

```java
buildProvider(/**/) {
    // ...
    modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
                fileDescriptorModelLoader);
    return new FixedLoadProvider<A, ImageVideoWrapper, Z, R>(modelLoader, transcoder, dataLoadProvider);
}
```

因此 `getModelLoader`返回的modeLoader为：

```java
ImageVideoModelLoader<A> modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
                                            fileDescriptorModelLoader);
```

调用**ImageVideoModelLoader** 的`getResourceFetcher` 方法，

```java 
public DataFetcher<ImageVideoWrapper> getResourceFetcher(A model, int width, int height) {
    if (streamLoader != null) {
        streamFetcher = streamLoader.getResourceFetcher(model, width, height);
    }
    
    if (streamFetcher != null || fileDescriptorFetcher != null) {
        return new ImageVideoFetcher(streamFetcher, fileDescriptorFetcher);
    }
}
```

看看这个streamFetcher是什么，先来看看streamLoader是什么？

*streamLoader* 在构造函数中传递过来，在**RequestManager** 的 `loadGeneric` 方法中赋值为：

```java
ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
```

在Glide的构造函数中：

```java
register(String.class, InputStream.class, new StreamStringLoader.Factory());
register(Uri.class, InputStream.class, new StreamUriLoader.Factory());
register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
```

在调用**Glide** 类的`buildStreamModelLoader` 方法中，会根据class类型，选择不同的ModelLoader，最终选择到的ModelLoader是**HttpUrlGlideUrlLoader** 。

因此，*streamFetcher* 为：

```java
public DataFetcher<InputStream> getResourceFetcher(GlideUrl model, int width, int height) {
    GlideUrl url = model;
    
    return new HttpUrlFetcher(url);
}
```

在**HttpUrlFetcher** 类的`loadData`  方法中，通过**HttpURLConnection** 进行网络通信，拿到InputStream流，交给DecodeJob，去解码，显示、缓存。



#### 后记

如果想要替换Glide的网络下载工具：

* 自定义DataFetcher组件，实现**DataFetcher** 接口，用作发起网络请求；
* 自定义ModelLoader组件，实现**ModelLoader**接口，重写`getResourceFetcher` 方法，返回自定义的DataFetcher；
* 实现GlideModule接口，在AndroidManifest中注册，使得Glide可以认识自定义的组件

