OkHttp之自定义拦截器实现下载进度监听

在配置OkHttpClient中，实现添加自定义的拦截器：

```java
  Interceptor interceptor = new Interceptor() {
    // 实现 intercept方法
    @Override
    public Response intercept(Chain chain) throws IOException {
      Response originalResponse = chain.proceed(chain.request());
      return originalResponse.newBuilder()
        .body(new ProgressResponseBody(originalResponse.body(), chain.request().url().toString()))
        .build();
    }
  };
  mHttpClient = new OkHttpClient.Builder()
    .addInterceptor(interceptor)
    .build();
```

接下来实现`ProgressResponseBody`，这个类继承自`ResponseBody`：

```java
public class ProgressResponseBody extends ResponseBody {

    public interface ProgressListener {
        void onPreExecute(long contentLength);
        void update(long totalBytes, boolean done);
    }

    private final ResponseBody responseBody;
    private final ProgressListener progressListener;
    private BufferedSource bufferedSource;

    public ProgressResponseBody(ResponseBody responseBody, String url
                                /*ProgressListener progressListener*/) {
        this.responseBody = responseBody;
//        this.progressListener = progressListener;
      	// 为了保证不同的url，有不同的回调，加入了ProgressListenerContainer
        this.progressListener = ProgressListenerContainer.getListener(url);
        if(progressListener!=null){
            progressListener.onPreExecute(contentLength());
        }
    }

    @Override
    public MediaType contentType() {
        return responseBody.contentType();
    }

    @Override
    public long contentLength() {
        return responseBody.contentLength();
    }

    @Override
    public BufferedSource source() {
        if (bufferedSource == null) {
            bufferedSource = Okio.buffer(source(responseBody.source()));
        }
        return bufferedSource;
    }

    private Source source(Source source) {
        return new ForwardingSource(source) {
            long totalBytes = 0L;
            @Override
            public long read(Buffer sink, long byteCount) throws IOException {
                long bytesRead = super.read(sink, byteCount);
                // read() returns the number of bytes read, or -1 if this source is exhausted.
                totalBytes += bytesRead != -1 ? bytesRead : 0;
                if (null != progressListener) {
                    progressListener.update(totalBytes, bytesRead == -1);
                }
                return bytesRead;
            }
        };
    }
}
```

其中，`ProgressListenerContainer`定义如下：

```java
public class ProgressListenerContainer {

    private static final Map<String, ProgressResponseBody.ProgressListener> LISTENER_MAP = new HashMap<>();

    public static void addListener(String url, ProgressResponseBody.ProgressListener listener) {
        LISTENER_MAP.put(url, listener);
    }

    public static ProgressResponseBody.ProgressListener getListener(String url) {
        return LISTENER_MAP.get(url);
    }

    public static void removeListener(String url) {
        LISTENER_MAP.remove(url);
    }
}
```

使用如下：

```java
ProgressListenerContainer.addListener(mUrl, new ProgressResponseBody.ProgressListener(){
    @Override
    public void onPreExecute(long contentLength) {}
  
  	@Override
    public void update(long totalBytes, boolean done) {}
});
```

