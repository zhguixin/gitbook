---
title: Android自动更新APK(三)
date: 2017-10-13
tags: Android

categories:

---

上一小节讲到DownloadManager的实现原理，整理梳理了DownloadManager的下载流程。这一小节通过OkHttp向服务器发送请求，观察是否有新版本，返回下载链接，然后交给自己写的下载管理器。

#### 通过OkHttp向服务器发送post请求

发送post请求，通过OkHttp主要是构造一个含有RequestBody的Request，通过建造者模式构建Request：

```java
final Request request = new Request.Builder()
  .url(mUrl)
  .header("Content-type", "application/json") // 添加一些必要的header信息
   // entityData为构造json串的byte数组，以bytes数组的长度作为Content-Length
  .header("Content-Length",  String.valueOf(entityData.length))
  .header(UpdateConstant.HEAD_ATTR_IMSI, mInstanceUtils.getIMSIString())
  .header(UpdateConstant.HEAD_ATTR_IMEI, mInstanceUtils.getIMEIString())
  .header(UpdateConstant.HEAD_ATTR_SYS_VERSION, mInstanceUtils.getSysVersion())
  .header(UpdateConstant.HEAD_ATTR_DEV_NAME, mInstanceUtils.getDeviceName())
  .header(UpdateConstant.HEAD_ATTR_EXTERNAL_DEV_NAME, mInstanceUtils.getExternalDevName())
  .header(UpdateConstant.HEAD_ATTR_DPI, mInstanceUtils.getDevDpi())
  .header(UpdateConstant.HEAD_ATTR_LOCALE, mInstanceUtils.getClentLocale())
  .header(UpdateConstant.HEAD_ATTR_VERSION, mInstanceUtils.getHeadVersion())
  .header(UpdateConstant.HEAD_ATTR_VERSIONCODE, mInstanceUtils.getHeadVersionCode())
  .header(UpdateConstant.HEAD_ATTR_TIMESTAMP, timeStamp)
  .header(UpdateConstant.HEAD_ATTR_TOKEN, mInstanceUtils.getRequestToken(timeStamp))
  // 调用RequestBody的静态函数 create 传入MediaType，传入byte[]
  .post(RequestBody.create(mMediaType, entityData))
  .build();
```

在构造json字符串时，使用了Google开源的Gson库：

```java
Gson mGson = new Gson();
byte[] entityData;

List<AppInfo> list = new ArrayList<>();
list.add(new AppInfo(this));

entityData = mGson.toJson(list).getBytes("utf-8");
```

其中AppInfo定义如下：

```java
public class AppInfo {

    private String appName;
    private String pkgName;
    private String version;
    private int versionCode;
    private String country;
    private String device;
    private String aliveUpdateChannel;
    private String extraAttributes;

    public AppInfo(Context context) {
        UpdateUtils instanceUtils = UpdateUtils.getInstance(context);
        this.versionCode = 92001;
        this.appName = instanceUtils.getAppName();
        this.pkgName = instanceUtils.getPackageName();
        this.version = instanceUtils.getVersionName();
        this.country = instanceUtils.getCountryStr();
        this.device = instanceUtils.getDeviceName();
        this.aliveUpdateChannel = instanceUtils.getAliveUpdateChannel();
        this.extraAttributes = "";
    }
}
```

构造好Request后，向服务器发送，通过OkHttp的`onResonse`回调解析服务器返回来的数据（一般都是json串），得到下载地址。

```java
mOkHttpClient.newCall(request).enqueue(new Callback() {
  @Override
  public void onFailure(Call call, IOException e) {
    // failed, do something...
  }
  
  @Override
  public void onResponse(Call call, Response response) {
    // 解析json串，得到download url
  }
}
```

发送请求的操作主要是要和服务器协商好的，包括header信息和post到服务器的requestbody信息。