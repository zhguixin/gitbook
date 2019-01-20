### Android自动更新APK实现

通过Service的方式（推荐使用**JobService**），在后台获取服务器的APK信息，然后下载到本地，通过调用系统安装程序，来安装最新的APK。

其中，调用系统安装程序的主要代码如下：

```java
File apkFil = new File(mContext.getExternalCacheDir(),"MFVKeyguard.apk");//APK存储路径
Intent intent = new Intent(Intent.ACTION_VIEW);
Uri apk;

// Android N之后的版本通过FileProvider的方式获得Uri
if(Build.VERSION.SDK_INT>=24) {
  apk= FileProvider.getUriForFile(this,"sto.com.stocourier.fileprovider",apkFil);
  intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
  intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
} else {
  apk=Uri.fromFile(apkFil);//低于7.0
}

intent.setDataAndType(apk, "application/vnd.android.package-archive");
startActivity(intent);
```

#### APK的下载实现

将服务器上的APK下载到本地，可以通过Android自带的`HttpURLConnection`也可以通过第三方网络访问的开源库，比如：`OkHttp`，也可以通过系统自带的`DownloadManager`。三种下载的方式各有利弊。

##### 1、通过OkHttp下载

```java
/*** 
	做了简单的封装，使用的时候可以直接调用：
	DownLoaderManager.getInstance().startDownload(this, url, callback);
	通过传递URL来下载，通过callback回调函数中做下载成功或者失败的处理。
***/
public class MyDownLoaderManager {

    private final static String TAG = "MyDownLoaderManager";
    private static MyDownLoaderManager manager;
    private AppUpdateResultCallback mListener;
    private Context mContext;

    private final OkHttpClient mHttpClient;

  	// 单例函数，确保OkHttpClient的唯一
    public synchronized static MyDownLoaderManager getInstance() {
        if(manager == null) {
            manager = new MyDownLoaderManager();
        }
        return manager;
    }

    private MyDownLoaderManager() {
        mHttpClient = new OkHttpClient();
    }

    public void startDownload(Context context, String downUrl, AppUpdateResultCallback listener) 	{
        mContext = context;
        mListener = listener;

        final Request request = new Request.Builder()
                .url(downUrl)
                .build();

        mHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.d(TAG, "Download failure");
                mListener.onDownloadAppFailed();
            }

            @Override
            public void onResponse(Call call, Response response) {
              	// 下载文件到SD卡上的Android/data/<package-name>/cache/目录下
              	// 注意异常捕获操作
                int len;
                byte[] buf = new byte[2048];
                InputStream inputStream = null;
                FileOutputStream fileOutputStream = null;
                try {
                    inputStream = response.body().byteStream();
                  
                  	// 通过mContext.getExternalCacheDir()拿到cache目录，文件名为：MFVKeyguard.apk
                  	// 该目录下的读写是不需要读写权限的，应用卸载后该目录被自动删除
                    File file = new File(mContext.getExternalCacheDir(),"MFVKeyguard.apk");
                    fileOutputStream = new FileOutputStream(file);

                    while ((len = inputStream.read(buf)) != -1) {
                        fileOutputStream.write(buf, 0, len);
                    }
                    fileOutputStream.flush();
                    Log.d(TAG, "Download success");
                    mListener.onDownloadAppSuccess();
                } catch (Exception e) {
                    Log.d(TAG, "Got an exception, download failure");
                    mListener.onDownloadAppFailed();
                } finally {
                    closeInputStream(inputStream);
                    closeOutputStream(fileOutputStream);
                }
            }
        });
    }

    private void closeInputStream(InputStream inputStream) {
        try {
            if (inputStream != null)
                inputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void closeOutputStream(OutputStream outputStream) {
        try {
            if (outputStream != null)
                outputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

有了整体流程，可以做些扩展，比如显示下载进度等。通过OkHttp下载不支持断点续传。要想支持断点续传，可以考虑使用系统自带的**DownloadManager**。

#### 2、通过DownloadManager下载

Android系统在很早的版本就已经集成了`DownloadManager`，通过`DownloadManager`方式下载可以良好的支持断点续传，即使此次下载中断了，下载网络良好的情况下可以继续下载。并且通过系统自带的APP——下载管理器，可以看到每一次的下载记录。一个简单的示例如下：

```java
public class DownLoaderManager {
    private static final String TAG = "DownLoaderManager";
    private Context mContext;
    private static DownLoaderManager manager;
    private String mDownloadUrl;
    private AppUpdateResultCallback mListener;
    private long mDownloadId;
    private DownloadManager mDownloadManager;

    private BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            long extractId = intent.getLongExtra(DownloadManager.EXTRA_DOWNLOAD_ID, -1);
            Log.d(TAG, "DownLoad Success!!!" + " mDownloadId=" + mDownloadId +
                    " extractId=" + extractId);
          	// 确保是自己的那条记录被下载完成
            if(mDownloadId == extractId) {
                mListener.onDownloadAppSuccess();
              	// 下载成功后，可以移除这条下载记录，这样通过下载管理器无法查看到，根据个人需求决定
//                mDownloadManager.remove(mDownloadId); // 下载成功后，可以移除这条下载记录
            }
        }
    };

    public synchronized static DownLoaderManager getInstance() {
        if(manager == null) {
            manager = new DownLoaderManager();
        }
        return manager;
    }

    public void startDownload(Context context, String downUrl, AppUpdateResultCallback listener) {
        mContext = context;
        mDownloadUrl = downUrl;
        mListener = listener;
      
      	// 虽然下载的目录仍然是SD卡上的Android/data/<package-name>/cache/目录下，但是需要读写权限
        if (!checkFile()) {
            Log.d(TAG, "Download failed caused by operating directors");
            mListener.onDownloadAppFailed();
            return;
        }
		
      	// 注册监听下载成功的广播
        mContext.registerReceiver(mReceiver,
                new IntentFilter(DownloadManager.ACTION_DOWNLOAD_COMPLETE));

        mDownloadManager = (DownloadManager)mContext.getSystemService(DOWNLOAD_SERVICE);
        DownloadManager.Request request = new DownloadManager.Request(Uri.parse(mDownloadUrl));

        request.setDestinationInExternalPublicDir("Android/data/" + mContext.getPackageName()+ "/cache",
                "MFVKeyguard.apk");
		// 不显示通知，需要 DOWNLOAD_WITHOUT_NOTIFICATION 权限
        request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_HIDDEN);
      	// 加入下载请求队列，DownloadManager会选择合适的时机自动下载，下载成功后发送广播
      	// enqueue的返回值代表了该下载任务的唯一号ID，其实对应于数据库中的行号:rowId
        mDownloadId = mDownloadManager.enqueue(request);
    }

  	// 检测SD卡状态，以及是否有读写权限
    private boolean checkFile() {
        String externalStorageState = Environment.getExternalStorageState();

        if("mounted".equals(externalStorageState) && hasExternalStoragePermission(mContext)) {
            File file = new File(mContext.getExternalCacheDir(),"MFVKeyguard.apk");
            if(file.exists()) {
                file.delete();
            }
            return true;
        }
        return false;
    }

    private boolean hasExternalStoragePermission(Context context) {
        int perm = context.checkCallingOrSelfPermission("android.permission.WRITE_EXTERNAL_STORAGE");
        return perm == 0;
    }
```

`DownloadManager`的内部实现，主要是通过`ContentProvider`实现的，你把一个下载请求插入到数据库中，`DownloadProvider`（在com/android/providers/downloads包下）检测到数据库有变化，启动`DownloadJobService`（继承自JobService），然后开启一个线程`DownloadThread`来进行下载。每一个下载任务对应一个`DownloadThread`线程。

关于DownloadManager，可以通过设置`request.set*`实现不同的需求，比如显示通知等，详细参考API说明。如果要想显示下载进度，需要调用DownloadManager的另一个重要函数**DownloadManager.Query()**。

```java
  // 通过Query的方法setFilterById过滤查询某一下载任务（指定mDownloadId）
  public int[] getBytesAndStatus(long downloadId) {
    int[] bytesAndStatus = new int[] { -1, -1, 0 };
    DownloadManager.Query query = new DownloadManager.Query().setFilterById(mDownloadId);
    Cursor c = null;
    try {
        c = downloadManager.query(query);
        if (c != null && c.moveToFirst()) {
            bytesAndStatus[0] = c.getInt(c.getColumnIndexOrThrow(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR));// 已下载大小
            bytesAndStatus[1] = c.getInt(c.getColumnIndexOrThrow(DownloadManager.COLUMN_TOTAL_SIZE_BYTES)); // 总大小
            bytesAndStatus[2] = c.getInt(c.getColumnIndex(DownloadManager.COLUMN_STATUS)); // 下载状态
        }
    } finally {
        if (c != null) {
            c.close();
        }
    }
    return bytesAndStatus;
}

```

如果我们想要实现下载进度的提示，我们可以定时1秒来获取当前下载的大小，计算出进度，反应到进度条上。

关于`mDownloadId`， 通过：

```java
mDownloadManager.getUriForDownloadedFile(mDownloadId);
```

得到该下载任务的URI，如：content://downloads/my_downloads/1125，其中**1125**就代表**mDownloadId**，也就是数据库中的行号。

#### 尾巴

以上主要介绍了APK下载过程的实现，这其实只是自动更新中的一环，还有向服务请求是否有新版本，以及下载完成后的安装等。接下来慢慢介绍。

