Android开发小技巧

### 1、获取状态栏高度

```java
    public static int getStatusBarHeight(Context context) {
        Class<?> c = null;
        Object obj = null;
        Field field = null;
        int x = 0, statusBarHeight = 0;
        try {
            c = Class.forName("com.android.internal.R$dimen");
            obj = c.newInstance();
            field = c.getField("status_bar_height");
            x = Integer.parseInt(field.get(obj).toString());
            statusBarHeight = context.getResources().getDimensionPixelSize(x);
        } catch (Exception e1) {
            e1.printStackTrace();
        }
        return statusBarHeight;
    }
```

### 2、应用间文件共享

Android 7.0以后，规定：如果一项包含文件 URI 的 Intent 离开您的应用，应用失败，并出现 FileUriExposedException异常。

应当使用`FileProvider`进行文件共享。在AndroidManifest中配置FileProvider

```xml
<provider
          android:name="android.support.v4.content.FileProvider"
          android:authorities="site.zhguixin.previewuitest.provider"
          android:exported="false"
          android:grantUriPermissions="true">
    <meta-data
               android:name="android.support.FILE_PROVIDER_PATHS"
               android:resource="@xml/provider_paths" />
</provider>
```

通过FileProvider，得到文件的URL：

```java
File image = new File(Environment.getExternalStorageDirectory(), "DCIM/haha.jpg");
Log.d(TAG, "fileShare: image's path=" + image.getPath());
Uri imageUri = FileProvider.getUriForFile(this, "site.zhguixin.previewuitest.provider", image);
Log.d(TAG, "fileShare: uri=" + imageUri.toString());

Intent cropImageIntent = new Intent("com.android.camera.action.CROP");
cropImageIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
cropImageIntent.setDataAndType(imageUri, "image/*");
cropImageIntent.putExtra("crop", "true");
cropImageIntent.putExtra("outputX", 80);
cropImageIntent.putExtra("outputY", 80);
cropImageIntent.putExtra("return-data", false);
startActivityForResult(cropImageIntent, 1);
```



#### Root原理

手机root的原理简单说就是：把su对应的二进制文件放到/system/bin或者/system/xbin下面。

因为Android手机的底层操作系统是Linux系统，Linux系统切换到管理员权限的命令就是su,一般手机生产厂商都会把su对应的二进制文件删除掉防止用户获得超级管理员权限. 但是把su二进制文件放到对应的目录下需要管理员权限,这就进入了死循环。

 现在root方案都是基于系统bug的,比如一直卸载挂载sd卡,然后造成系统在某一时间不再检查权限,然后”偷偷”地把su二进制文件放到对应文件夹达到root的目的,或者是在系统备份还原的时候把su二进制文件放进去。

判断手机是否已Root：

```java
public static boolean isPhoneRoot() {
    if (!(new File("/system/bin/su")).exists() && 
        !(new File("/system/xbin/su")).exists()) {
        return false;
    } else {
        Log.d("have_root", new Object[0]);
        return true;
    }
}
```



