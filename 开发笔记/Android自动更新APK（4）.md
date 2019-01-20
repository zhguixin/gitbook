---
title: Android自动更新APK(四)
date: 2017-10-13
tags: Android HTTP
categories:
---

这一节研究一下断点续传的原理和实现。

#### Http断点续传原理

关于断点续传，关键点主要有两个：

- 1、客户端知道当前的文件和上一次加载的文件是不是内容发生了变化，如果有变化，需要重新从offset 0 的位置开始下载；
- 2、客户端记录好上次成功下载到的offset，告诉服务器，服务器支持从特定的offset 开始传输数据。

**通常情况下，Web服务器(如Apache)会默认开启对断点续传的支持。验证服务器是否支持断点续传，通过Http分析工具，查看服务器的响应header中是否包含“Accept-Ranges: bytes”（按字节下载）或者查看响应码：**

> HTTP/1.1 200 Ok（不使用断点续传方式） 
> HTTP/1.1 206 Partial Content（使用断点续传方式）

针对第一个点，我们可以通过Http协议的ETAG方案实现：HTTP 的ETAG来标识是否文件已经修改。从`DownloadManager`的下载源码中可以看到，`DownloadManager`也用到了ETAG这种方案。关于ETAG方案：

> ETAG原理：如果URL上的资源内容改变，一个新的不一样的ETag就会被分配。用这种方法使用ETag即类似于**指纹**，并且他们能够被快速地被比较，以确定两个版本的资源是否相同。ETag的比较只对同一个URL有意义——不同URL上的资源的ETag值可能相同也可能不同，从他们的ETag的比较中无从推断。

我们要做的就是收到响应的时候存储这个ETag，以便下次下载的时候从断点下载处再把这个ETag发送给服务器。

```java
    // 构造Request的是，加上header信息
	Request.Builder builder = new Request.Builder();
	if (mETag != null) {
      builder.header("If-Match", mETag);
    }
	
	// 收到响应的时候，获得ETag信息
	@Override
    public void onResponse(Call call, Response response) {
      mETag = response.header("ETag");
    }

// 剩下的就是交给服务器去做校验了
```



针对第二个点，我们可以通过HTTP头Range字段，指定下载文件的某一段大小，及其单位。格式如下：

> Range: bytes=0-499 下载第0-499字节范围的内容
>
> Range: bytes=500-999 下载第500-999字节范围的内容
>
> Range: bytes=-500 下载最后500字节的内容
>
> Range: bytes=500- 下载从第500字节开始到文件结束部分的内容

```java
    Request.Builder builder = new Request.Builder();
	// 下载从 currentBytes 开始到文件结束部分的内容
    builder.header("RANGE", "bytes=" + currentBytes + "-");// Http header不区分大小写
```

在收到服务器的时候，在header信息中会有一个**Content-Range**字段，形如：

> Content-Range: bytes 1024-7283259/7283259

**客户端发请求时对应的是 Range ，服务器端响应时对应的是 Content-Range**

#### 本地文件操作

关于断点下载和服务器的交互，已经能够保证了。接下来要考虑的就是对本地已下载的部分文件的操作。在Android中可以使用**RandomAccessFile**来随机的操作文件（这块还要考虑磁盘空间是否够用等，此处暂不考虑）。

```java
    RandomAccessFile randomAccessFile;
	try {
      File file = new File(mContext.getExternalCacheDir(),"MFVKeyguard.apk");
      randomAccessFile = new RandomAccessFile(file, "rwd");
      if (file.exists()) { // 如果文件存在
        mCurrentBytes = randomAccessFile.length(); // 获得当前文件大小
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
```

在OkHttp的`onResponse`回调中，实现如下：

```java
    @Override
    public void onResponse(Call call, Response response) {
      int len;
      byte[] buf = new byte[2048];
      InputStream inputStream = null;
      FileChannel channelOut = null;
      try {
        // mETag = response.header("ETag");
        // mLastModified = response.header("Last-Modified");

        inputStream = response.body().byteStream();
        Log.d(TAG, "ETag= " + mETag + " Last-Modified=" + mLastModified +
              " code=" + response.code());
        Log.d(TAG, "Content-Range= " + response.header("Content-Range"));

        channelOut = randomAccessFile.getChannel();
        MappedByteBuffer mappedBuffer = channelOut.map(FileChannel.MapMode.READ_WRITE,
             mCurrentBytes, response.body().contentLength());

        while ((len = inputStream.read(buf)) != -1) {
          mappedBuffer.put(buf, 0, len);
        }

      } catch (Exception e) {
        Log.d(TAG, "Got an exception, download failure");
        mListener.onDownloadAppFailed();
      } finally {
        closeInputStream(inputStream);
        closeChannelOut(channelOut);
        closeRandomAccessFile(randomAccessFile);
      }
    }
```

此处，没有使用randomAccessFile的seek方法来移动到写入位置，对于大文件来说这种速度比较慢，这里采用了NIO包中的`FileChannel`。通过`fileChannel`得到的MappedByteBuffer是ByteBuffer的子类，通过`MappedByteBuffer`的put方法来写入到缓存中：

```java
int len;
byte[] buf = new byte[2048];
InputStream inputStream = response.body().byteStream();
while ((len = inputStream.read(buf)) != -1) {
  mappedBuffer.put(buf, 0, len);
}
```

*IO与NIO直接最大的一个区别就是：IO是面向流的；NIO是面向缓存区的。关于IO与NIO可参考：[Java IO与NIO](http://ifeve.com/java-nio-vs-io/)*

然后randomAccessFile通过FileChannel从缓存中读取：

```java
// 将此通道的文件区域直接映射到内存中;
// 必须指明，它是从文件的哪个位置开始映射的【mCurrentBytes】，
// 映射的范围又有多大【response.body().contentLength()】
FileChannel channelOut = randomAccessFile.getChannel();
MappedByteBuffer mappedBuffer = channelOut.map(FileChannel.MapMode.READ_WRITE,
          mCurrentBytes, response.body().contentLength());
```

**注意：response.body().contentLength()，下次再从断点出下载时，表示剩余未下载的大小**

