ListView是Android中非常常用的控件，用来展示大量的数据信息。

- 作为一个信息展示控件，如何与数据解耦
- 如何做到加载大量数据不会出现OOM
- ListView所体现的设计思想



ListView本质是一个自定义ViewGroup，支持垂直信息的展示，继承结构如下：

在Android Studio使用快捷键F4查看。

![](F:\gitbook\images\listview.JPG)



RecycleBin机制，这个机制也是ListView能够实现成百上千条数据都不会OOM最重要的一个原因。其实RecycleBin的代码并不多，只有300行左右，它是写在AbsListView中的一个内部类，所以所有继承自AbsListView的子类，也就是ListView和GridView，都可以使用这个机制。

##### 加载第一屏数据

从AbsListView的onLayout开始阅读，具体实现为ListView的layoutChildren方法。

fillDown()方法中的while循环，会让子元素View将整个ListView控件填满然后就跳出，也就是说即使我们的Adapter中有一千条数据，ListView也只会加载第一屏的数据，剩下的数据反正目前在屏幕上也看不到，所以不会去做多余的加载工作，这样就可以保证ListView中的内容能够迅速展示到屏幕上。



两次layout过程？第一次通过inflate的方式加载View，第二次通过加载缓存attach view。

##### 滑动加载更多数据

滑动的处理过程是通用的，具体处理在AbsListView中，onTouchEvent方法：

```java
onTouchMove()  ---> scrollIfNeeded()
/**
在 scrollIfNeeded 中：
deltaY表示从手指按下时的位置到当前手指位置的距离；
incrementalDeltaY则表示据上次触发event事件手指在Y方向上位置的改变量；*/
int incrementalDeltaY =
        mLastY != Integer.MIN_VALUE ? y - mLastY + scrollConsumedCorrection : deltaY;
// 我们就可以通过incrementalDeltaY的正负值情况来判断用户是向上还是向下滑动的了。
```

当ListView向下滑动的时候，就会进入一个for循环当中，从上往下依次获取子View。

如果该子View的bottom值已经小于top值了，就说明这个子View已经移出屏幕了，所以会调用RecycleBin的addScrapView()方法将这个View加入到废弃缓存当中，并将count计数器加1，计数器用于记录有多少个子View被移出了屏幕。

当滑动过程中需要加载新数据时，即如果ListView中最后一个View的底部已经移入了屏幕，或者ListView中第一个View的顶部移入了屏幕，就会调用fillGap()方法，最终调用到RecyleBin的getScrapView()方法来尝试从废弃缓存中获取一个View。

这也就说明为什么ListView即使加载再多的数据也不会出现OOM。

#####ListView异步加载图片乱序的问题

RecycleBin的回收机制是对View控件的重复利用。比如，在`getView()`中进行异步网络请求去下载图片，网络下载属于耗时操作，此时快速滑动ListView，此时又会调用`getView()` ，重新下载图片，被移除的ImageView又被重复利用，位置不同，但是ImageView确是同一个，并且此时该ImageView才收到上一次下载成功的响应。

导致的现象就是，图片跳动、乱序。

其中一种方案：

* 给ImageView设置Tag，每次调用`getView` 的时候，给ImageView设置一个Tag（比如以url地址作为tag）。我们获取Imageview实例设置下载好的图片时，通过findViewByTag来拿到ImageView。只有当ImageView不为null时，才设置图片。

  原因就是，快速滑动ListView时，调用`getView` 方法，会重新设置Tag。图片异步下载完成后，调用`findViewByTag` 传入的老Tag已经被新的Tag给覆盖掉了，拿到的ImageView为null，不会设置图片了。

参考：[**Android ListView异步加载图片乱序问题，原因分析及解决方案**](http://blog.csdn.net/guolin_blog/article/details/45586553) 