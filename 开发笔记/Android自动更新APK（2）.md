### Android自动更新APK（2）

####断点下载、续传的实现

上一节说到从服务器下载APK的过程，说OkHttp不支持断点下载，`DownloadManager`支持断点下载。通过配置OkHttpClient添加拦截器`interptcor`也是可以支持断点下载，再此之前，我们先看看`DownloadManager`的实现原理，看看`DownloadManager1`怎么支持断点下载的。

#### DownloadManager的实现原理分析

在上一节讲到构造一个Request交给DownloadManager，然后会自动下载：

```java
  downloadManager = (DownloadManager) mContext.getSystemService(Context.DOWNLOAD_SERVICE);
  downloadId = downloadManager.enqueue(request);
```

其中`enqueue`的源码：

```java
// android/app/DownloadManager.java

    public long enqueue(Request request) {
        ContentValues values = request.toContentValues(mPackageName);
      	// 直接调用到DownloadProvider中的insert方法
        Uri downloadUri = mResolver.insert(Downloads.Impl.CONTENT_URI, values);
        long id = Long.parseLong(downloadUri.getLastPathSegment());
        return id;
    }

```

我们在Android源码中的**providers**目录下找到DownloadProvider.java中查看insert的主要代码如下：

```java
    final long token = Binder.clearCallingIdentity();
    try {
      // 通过JobSchedule来启动DownloadJobService
      Helpers.scheduleJob(getContext(), rowID);
    } finally {
      Binder.restoreCallingIdentity(token);
    }
```

在DownloadJobService中启动DownloadThread，开启下载线程：

```java
	@Override
    public boolean onStartJob(JobParameters params) {
        final int id = params.getJobId();

        // Spin up thread to handle this download
        final DownloadInfo info = DownloadInfo.queryDownloadInfo(this, id);
        if (info == null) {
            Log.w(TAG, "Odd, no details found for download " + id);
            return false;
        }

        final DownloadThread thread;
        synchronized (mActiveThreads) {
            thread = new DownloadThread(this, params, info);
            mActiveThreads.put(id, thread);
        }
        thread.start();

        return true;
    }
```

在DownloadThread中，开启网络连接，进行下载，直接看DownloadThread的run方法：

```java
    @Override
    public void run() {
      // ...
      try {
        // ...
        mInfoDelta.mStatus = STATUS_RUNNING;
        // 向download.db中写入一些数据，包括下载状态，文件大小，已下载大小等
        mInfoDelta.writeToDatabase(); 
        
        // ...
        executeDownload();// 网络连接实现真正下载的地方
        
        // 异常捕获，失败重试等操作
      }

    private void executeDownload() throws StopRequestException {
      // 当前已下载的字节数不为0，在构造request的时候，传入已下载的字节数，告诉服务器下载
      final boolean resuming = mInfoDelta.mCurrentBytes != 0;
      // ...
      
      // 在重定向次数不超过5的时候，连接网络进行下载，通过HttpURLConnection连接
      int redirectionCount = 0;
      while (redirectionCount++ < Constants.MAX_REDIRECTS) {
        // ...
        
        HttpURLConnection conn = null;
        try {
          checkConnectivity();
          conn = (HttpURLConnection) mNetwork.openConnection(url);
          conn.setInstanceFollowRedirects(false);
          conn.setConnectTimeout(DEFAULT_TIMEOUT);
          conn.setReadTimeout(DEFAULT_TIMEOUT);
          if (conn instanceof HttpsURLConnection) {
            ((HttpsURLConnection)conn).setSSLSocketFactory(appContext.getSocketFactory());
          }
          // 构造请求头，设置代理，已下载大小等。
		  addRequestHeaders(conn, resuming);
          
          final int responseCode = conn.getResponseCode();
          switch (responseCode) {
            case HTTP_OK:
              if (resuming) {
                throw new StopRequestException(
                  STATUS_CANNOT_RESUME, "Expected partial, but received OK");
               }
               // 根据response，写入文件
               parseOkHeaders(conn);
               // 存储到数据库
               transferData(conn);
               return;

               case HTTP_PARTIAL:
                 if (!resuming) {
                   throw new StopRequestException(
                                    STATUS_CANNOT_RESUME, "Expected OK, but received partial");
                  }
                 transferData(conn);
                 return;
                    // 其他的状态处理
                }
            }
        }

    }
```

以上代码只是截取了重要的一部分，在函数`addRequestHeaders(conn, resuming);`中构造请求头，有如下一句话：

```java
  if (resuming) {
    if (mInfoDelta.mETag != null) {
      conn.addRequestProperty("If-Match", mInfoDelta.mETag);
    }
    conn.addRequestProperty("Range", "bytes=" + mInfoDelta.mCurrentBytes + "-");
  }
```

关于ETag的作用，就是标识资源是否已经被修改过了，ETag值的变更说明资源状态已经被修改，具体的操作由服务器来处理。详细可以查询相关资料。

在方法`parseOkHeaders`中主要获取一些连接信息，包括：文件名、MimeType、文件总大小等，保存在`DownloadInfoDelta`（DownloadThread的内部类）中，然后写入数据库，这个函数很简单：

```java
    private void parseOkHeaders(HttpURLConnection conn) throws StopRequestException {
        if (mInfoDelta.mFileName == null) {
            final String contentDisposition = conn.getHeaderField("Content-Disposition");
            final String contentLocation = conn.getHeaderField("Content-Location");

            try {
                mInfoDelta.mFileName = Helpers.generateSaveFile(mContext, mInfoDelta.mUri,
                        mInfo.mHint, contentDisposition, contentLocation, mInfoDelta.mMimeType,
                        mInfo.mDestination);
            } catch (IOException e) {
                throw new StopRequestException(
                        Downloads.Impl.STATUS_FILE_ERROR, "Failed to generate filename: " + e);
            }
        }

        if (mInfoDelta.mMimeType == null) {
            mInfoDelta.mMimeType = Intent.normalizeMimeType(conn.getContentType());
        }

        final String transferEncoding = conn.getHeaderField("Transfer-Encoding");
        if (transferEncoding == null) {
            mInfoDelta.mTotalBytes = getHeaderFieldLong(conn, "Content-Length", -1);
        } else {
            mInfoDelta.mTotalBytes = -1;
        }

        mInfoDelta.mETag = conn.getHeaderField("ETag");

        mInfoDelta.writeToDatabaseOrThrow();

        // Check connectivity again now that we know the total size
        checkConnectivity();
    }
```

在transferData函数中，为了方便查看，省略了大部分异常处理、DRM处理的代码：

```java
private void transferData(HttpURLConnection conn) throws StopRequestException {
  
  ParcelFileDescriptor outPfd = null;
  FileDescriptor outFd = null;
  InputStream in = null;
  OutputStream out = null;
  
  in = conn.getInputStream();
  outPfd = mContext.getContentResolver()
    .openFileDescriptor(mInfo.getAllDownloadsUri(), "rw");
  outFd = outPfd.getFileDescriptor();
  
  out = new ParcelFileDescriptor.AutoCloseOutputStream(outPfd);
  // 把指针指向已下载字节数的后面，方便继续写入
  Os.lseek(outFd, mInfoDelta.mCurrentBytes, OsConstants.SEEK_SET);
  
  transferData(in, out, outFd);
}
```

在`transferData(in, out, outFd)`函数中（省略了磁盘空间检查等部分的代码）：

```java
private void transferData(InputStream in, OutputStream out, FileDescriptor outFd) {
  final byte buffer[] = new byte[Constants.BUFFER_SIZE];
  
  while（true) {
    len = in.read(buffer);
    if (len == -1) {
      break;
    }
    out.write(buffer, 0, len);
    
    mInfoDelta.mCurrentBytes += len;// 更新当前下载的字节数
  }

}
```

