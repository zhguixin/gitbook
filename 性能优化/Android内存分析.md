Android内存分析

每个应用对应一个虚拟机实例，虚拟机默认分配的堆内存空间，可通过如下查看系统属性得到：

```bash
adb shell getprop dalvik.vm.heapgrowthlimit
```

在我的手机上输出的是`256m`（手机运存为6GB），它表示了单个应用可用的最大内存，超出后就会报OOM。

此外，如果你在应用的AndroidManifest中配置了

```xml
android:largeHeap="true"
```

应用可以使用的最大内存，通过查看系统属性获得：

```bash
adb shell getprop dalvik.vm.heapsize // 512m
```

对普通应用来说不建议使用，并且Android对该属性也做了限制，不是配置就可以获得到更大内存的。



当堆栈内存逐渐增大时，Dalvik虚拟机中管理内存回收的垃圾收集器（garbage collector，GC）将会被触发，进行内存回收操作。每一个进程，**也就是每一个应用程序都有自己对应的虚拟机和垃圾收集器**。尽管如此，应用程序中所分配的对象也能够充满堆栈区却不能及时触发垃圾收集器进行内存回收，从而引起内存泄漏。

内存泄漏是指已经不再使用的对象仍然被GC Root引用着，导致这个不再使用的对象一直无法释放，导致内存泄漏。因此，要解决这种内存泄漏的情况，释放掉该对象根本上就是将path to gc这个链路剪断就可以。

通过adb shell命令，可以粗略看看当前的内存使用情况：

```bash
adb shell dumpsys meminfo com.zgx.testapp
```

![](.\imgs\meminfo.JPG)

可以看到Java堆、Native堆、Graphics层的内存占用情况；还可以查看对象数目，View、Activity、Context等的数目。

### 通过MAT分析堆内存占用情况

#### MAT工具使用

通过`adb` 命令抓取应用堆内存信息：

```bash
ps -e | grep com.zte.mfvkeyguard
u0_a36 11248  1609 1686280  46460 SyS_epoll_wait f2325f94 S com.zte.mfvkeyguard

adb shell am dumpheap 11248  /data/local/tmp/com.zte.mfvkeyguard_A1.hprof
adb pull /data/local/tmp/com.zte.mfvkeyguard_A1.hprof .
```

> 11248为PID进程号

在借助于Eclipse MAT分析之前，先将hporf文件转换一下，通过`hporf-conv`命令(在`sdk/platform-tools` 目录下)：

```shell
hprof-conv com.zte.mfvkeyguard_A1.hprof converted.hprof
```

通过MAT分析，下载安装链接见[官网](https://www.eclipse.org/mat/downloads.php)。MAT查看堆内存情况，常用的几个视图：

![](./imgs/MAT.JPG)

从左到由的方向：

第一个图标是【Over View】视图，通过饼图的形式，展示了各个对象占用总应用内存的情况

第二个是【Histogram】视图，按照包名顺序列出包相关对象的数量、Shallow Heap 和 Retained Heap

> 我常用的是：Group by package，可以按照应用内的包名排序查看

第三个是【Dominator Tree】视图，对象支配树视图，该视图的主要作用就是寻找占用内存最多的对象，默认按照对象展示的

> 观察这个视图，发现每个对象前面有个文件形式的图标，有的图标上面有红点、有的则没有。有红点的表示可以被GC Root引用得到，否则不会被GC Root引用得到。当然那些带红点的并不都是泄漏对象，因为系统中确实需要不能被回收掉的对象，要常驻内存的。

第四个是【QQL】查询视图，MAT还提供了查询某个对象的QQL查询语句，类似于SQL语句：

1. 查找某个类的实例

```sql
select * from com.zgx.Test
```

2. 查找所有的Activity：

```sql
select * from instanceof android.app.Activity
```

3. 查询size=0并且未使用的ArrayList；

```sql
select * from java.util.ArrayList where size=0 and modCount=0
```

#### 分析步骤

分析内存泄漏的基本步骤，确保应用冷启动，启动后抓取一次堆内存信息；运行一段时间Monkey测试后（两个多小时）在内存最高点时，再抓取一次堆内存信息。

通过MAT对比两次hprof文件，MAT还支持查看两次dump对象数目的增减情况。

点击Histogram查看，各个类的内存占用情况，其中有两个表示内存的列：**Shallow Heap** 和 **Retained Heap**

* Shallow Head，表示对象本身占用的内存空间；
* Retained  Heap，表示对象本身的Shallow Size + 对象能直接或间接访问到的对象的Shallow Size，也就是说Retained Size就是该对象被gc之后所能回收内存的总和。

查看某个对象的引用关系，在表示对象的那一行，点击右键：

**Merge shortest paths to GC root -> exclude phatom/weak/soft etc. references**

到GC root的路径，只查看强引用。

#### 使用Android Studio分析

MAT工具很强大，但是并非是为分析Android App设计的，在后来的Android版本中，将Bitmap对象放到Native侧，这个是使用MAT工具分析不到的。因此需要借助于Android Studio（后续简称AS）来分析Bitmap内存占用情况。

可以将转换之前的hprof文件直接用AS打开，不需要通过hprof-conv转换；并且AS3.0提供了Profile，可以实时动态的分析内存、CPU、网络的使用情况，详细可参考：[Android Studio 3.0 利用 Android Profiler 测量应用性能 - 掘金](https://juejin.im/post/5b7cbf6f518825430810bcc6)



### 常见的导致内存泄漏情况

##### Handler导致的内存泄漏

在Android开发中，Handler的使用频率是非常高的，但是使用不当就会导致内存泄漏。

我们都知道：**非静态内部类或者匿名内部类会持有外部类的强引用** ，比如：在Activity的`onCreate`中创建一个Handler：

```java
private  Handler mHandler;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    mHandler = new Handler() { // mHandler为匿名内部类对象
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };

    Message message = Message.obtain();
    message.what = 1;
    mHandler.sendMessageDelayed(message,10*60*1000);
}
```

mHandler延迟10分钟发送一个消息，10分钟之内退出Activity的话，由于该延迟消息还在MessageQueue中，引用链如下：msg---Handler---Activity，导致Activity无法被回收，出现内存泄漏。

解决方案：

1、匿名内部类改为静态内部类；

2、在该静态内部类中增加一个成员变量弱引用外部类对象实例，在静态内部类的构造函数中传入外部类实例

```java
private  Handler mHandler;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    mHandler = new MyHandler(this);

    Message message = Message.obtain();
    message.what = 1;
    mHandler.sendMessageDelayed(message,10*60*1000);
}

// 重新定义 MyHandler 为静态内部类
private static class MyHandler extends Handler {
    // 增加一个成员变量弱引用外部类对象实例
    private WeakReference<MyActivity> mActivity;
    
    public MyHandler(MyActivity activity){
        mActivity = new WeakReference<>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
    }
}
```

2、在Activity的onDestroy中，移除当前Handler的所有消息和回调：

```java
mHandler.removeCallbacksAndMessages(null);
```



##### Glide内存占用过大分析

在模块内有一个借助于ViewPager+Glide提供的壁纸预览Activity，在这个Activity中占用的内存相当高。默认情况下，Glide自动就是开启内存缓存的，所以这个滑动过程中有多少张壁纸就会有多少缓存。ViewPager缓存三种图片的机制也被替代了。

我们可以调用skipMemoryCache()方法并传入true，来禁用掉Glide的内存缓存功能。

在RecycleView中使用Glide的时候，在自定义Adapter中，重写`onViewRecycled`，清除Glide缓存

```java
 @Override
 public void onViewRecycled(WallpaperNodeHolder holder) {
     super.onViewRecycled(holder);
     Glide.get(this).clearMemory(); // 清除缓存
 }
```

在ViewPager的自定义Adapter中，覆写`destroyItem` ，清除ViewPager的缓存

```java
public void destroyItem(ViewGroup container, int position, Object object) {
       super.destroyItem(container, position, object);
       ViewPagerFragment fragment = (ViewPagerFragment) object;
       if (fragment == null) {
           return;
       }
       ((ViewPager) container).removeView(fragment.getView());
   }
}
```

自定义Glide组件，指定Glide的缓存大小：

```java
 public class GlideConfiguration implements GlideModule {
     @Override
    public void applyOptions(Context context, GlideBuilder builder) {
         builder.setBitmapPool(new LruBitmapPool(40 * 1024*1024))
                 .setMemoryCache(new LruResourceCache(40 * 1024 * 1024))
                 .setDecodeFormat(DecodeFormat.PREFER_ARGB_8888);
     }
 
     @Override
     public void registerComponents(Context context, Glide glide) {
 
    }
 }
```

### 可导致内存泄漏的常见原因

1. 非静态内部类的生命周期长于外部类

   只有非静态内部类没有被回收，那么他的外部类也不会被回收。

2. 静态类对象引用context，导致Activity无法释放

3. 注册对象后，没有在销毁函数中反注册

4. 资源对象为关闭

   数据库连接、cursor对象、资源IO没有及时关闭

5. 集合中的对象没有清理

   当这个集合对象是**某个对象的静态成员变量**或者是**单例对象的成员变量**时，随着集合越来越大，非常容易发生内存泄漏。因为静态变量和单例对象的生命周期是比较长的。

   ```java
   public class SingleManager {
       private static SingleManager INSTANCE;
       
       private final ArrayList<Listener> mListeners = new ArrayList<>();
   }
   ```

   `SingleManager` 是个单例对象，在进程中只有一个实例，因此成员对象mListeners也是只有一个对象实例。当某个对象`add` 进了这个`mListeners` 集合中，但不需要改对象时，又没有及时被`remove` ，随着`add` 进来的对象越来越多，就会造成内存泄漏。

   那什么时候，会出现add对象进去，没有remove掉呢。举个栗子：

   ```java
   @Override
   public void onWindowFocusChanged(boolean hasWindowFocus) {
       if (hasWindowFocus) {
           mSingleManager.addListener(this);
       } else {
           mSingleManager.removeListener(this);
       }
   }
   ```

   上述代码的功能是当窗口得到焦点时，把当前对象加入到集合对象中；失去焦点时，移除集合对象。

   当时当屏幕发生旋转时，在`onDestory` 方法前后没有调用`onWindowFocusChanged(false)` ，Activity重建后，在`onResume` 方法后调用了`onWindowFocusChanged(true)` 。如此一来，该集合中的对象就没有被移除掉，`add` 和`remove` 没有成对出现。

   修改方案可以分别将`add` 和` remove` 操作放到：`onResume` 和`onPause` 中。
