### 深入解析Bitmap

#### 1、Bitmap加载到内存的大小

Bitmap在内存所占大小通过如下方式计算：

```
图片的最终高度 * 图片的最终宽度 *　4（ARGB888）
```

> - ALPHA_8：每个像素占用1byte内存。
> - ARGB_4444:每个像素占用2byte内存
> - ARGB_8888:每个像素占用4byte内存
> - RGB_565:每个像素占用2byte内存

| density | densityDpi(dpi) | 分辨率         | res     |
| ------- | --------------- | ----------- | ------- |
| 1       | 160             | 320 X 533   | mdpi    |
| 1.5     | 240             | 480 X 800   | hdpi    |
| 2       | 320             | 720 X 1280  | xhdpi   |
| 3       | 480             | 1080 X 1920 | xxhdpi  |
| 3.5     | 560             | …           | xxxhdpi |

density：密度，指每平方英寸中的像素数，在DisplayMetrics类中属性density的值为dpi/160 
densityDpi，单位密度下可存在的点。



图片的最终宽高的计算如下：

> scale= targetDensity/density
>
> height = （scale * originalHeight +0.5f)
>
> width= （scale * originalWidth +0.5f)



```java
  DisplayMetrics metrics = new DisplayMetrics();
  getWindowManager().getDefaultDisplay().getMetrics(metrics);

  mBitmap = BitmapFactory.decodeResource(this.getResources(),R.mipmap.smile_face);
  mMyImgView.setImageBitmap(mBitmap);
  Log.d(TAG,"zgx: device`s DPI=" + metrics.densityDpi);
  Log.d(TAG,"zgx: height=" + mBitmap.getHeight());
  Log.d(TAG,"zgx: width=" + mBitmap.getWidth());
  Log.d(TAG,"zgx: byteCount=" + mBitmap.getByteCount());
```

假设一张图片的原始大小为1024\*1024，放到xxhdpi（DPI为480）目录下，手机的分辨率为：2560\*1440（640DPI），因此图片的最终宽高为：

> height = (1024 /* 640/480 + 0.5f) = 1365
>
> width =  (1024 /* 640/480 + 0.5f) = 1365

程序输出结果如下：

>09-07 13:52:45.428 29048-29048/? D/MainActivity: zgx: device`s DPI=640
>09-07 13:52:45.428 29048-29048/? D/MainActivity: zgx: height=1365
>09-07 13:52:45.428 29048-29048/? D/MainActivity: zgx: width=1365
>09-07 13:52:45.428 29048-29048/? D/MainActivity: zgx: byteCount=7452900

对放到不与屏幕匹配下的drawable目录，Android系统会对图片进行缩放。因此建议图片都要放在高DPI的目录下，来减少Bitmap占用的内存。

> 在Android2.3（API10）之前，Bitmap对象所占的内存存放于Dalvik堆上，由GC负责回收，另外一部分的内存，即Bitmap的像素数据存放于Native堆上，需要通过调用：`Bitmap.recycle()`方法回收。目前，所有的内存数据都存放于Dalvlik堆上了。

[关于Android中图片大小、内存占用与drawable文件夹关系的研究与分析](http://www.jianshu.com/p/312511edf94f)

#### Bitmap的常用操作

Bitmap的构造方法是私有的，无法通过new的方式创建，常用的创建Bitmap的方式有：

```java
Bitmap bitmap = BitmapFactory.decodeResource(getResource(), R.drawable.ic_drawable);

Bitmap bitmap = Bitmap.createBitmap(w, h, config);
```

缓存经常使用到的Bitmap，避免每次都进行`decode()`操作。

创建图片时，避免OOM错误，进行捕获处理，返回默认图片资源：

```java
Bitmap bitmap = null;
try {
    bitmap = BitmapFactory.decodeFile("");
} catch (OutOfMemoryError error) {
    error.getMessage();
}
if (bitmap == null) {
    // 如果实例化失败 返回默认的Bitmap对象
    return defaultBitmapMap;

}
```

对图片进行采样压缩后，再进行`decode()`获得Bitmap

```java
BitmapFactory.Options opts = new BitmapFactory.Options();

// 设置inJustDecodeBounds为true
opts.inJustDecodeBounds = true;
// 使用decodeFile方法得到图片的宽和高，并不会真正的分配空间，即解码出来的Bitmap为null
BitmapFactory.decodeFile(path, opts);
// 打印出图片的宽和高
Log.d("example", opts.outWidth + "," + opts.outHeight);

// 计算inSampleSize值
opts.inSampleSize = calculateInSampleSize(options, opts.outWidth, opts.outHeight);
// 使用获取到的inSampleSize值再次解析图片
opts.inJustDecodeBounds = false;
Bitmap bitmap = BitmapFactory.decodeFile(path, opts);
```



#### recycle操作

Bitmap所表示的图片占用的是Native内存，通过触发`recycle()`操作来进行回收，Bitmap类对象是在堆内存中。对于用不到的图片，进行及时触发回收操作：

```java
// 先判断是否已经回收
if(bitmap != null && !bitmap.isRecycled()){
    // 回收并且置为null
    bitmap.recycle();
    bitmap = null;
}

System.gc();
```



### Drawable

Drawable指的是一切可被绘制的资源，Drawable更像是一个功能类，本身并不能进行绘制。可以将Canvas传递给Drawable来进行绘制，通过Drawable触发的绘制，会绘制到该Canvas上。

它的`draw()`方法是抽象方法。

> A Drawable is a general abstraction for "something that can be drawn."

每个具体的Drawable对象仅仅绘制某个特定的图案，SDK中提供的Drawable实现类有：BitmapDrawable、ColorDrawable、ShapeDrawable。Drawable一般都是XML来定义的，定义在`Drawable目录下`：

- BitmapDrawable对应于`<bitmap>`标签，通过`android:src`指定图片资源
- ShapeDrawable对应于`<shape>`标签，表示一个纯色或者渐变的图形

- StateListDrawable对应于`<selector>`标签，它是一个Drawable的集合，根据不同的状态选择不同的Drawable

如果是将Button的背景设置为selector，在初始化Button的时候会将正反选图片都加载在内存中（具体可以查看Android源码，在类Drawable.java的createFromXmlInner方法中对图片进行解析，最终调用Drawable的inflate方法），相当于一个按钮占用了两张相同大小图片所使用的内存。