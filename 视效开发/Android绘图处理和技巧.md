Android 绘图处理和技巧

新建一个Canvas：

```java
Canvas canvas = new Canvas(bitmap);
```

传入一个Bitmap对象，后续调用`canvas.drwaXXX`所绘制的东西都会存储在`bitmap`对象中。

Canvas的所代表的画布操作方法：

```java
// ... 一系列的绘制操作
canvas.save(); // 保存在当前画布上绘制的内容
canvas.translate(); // 平移当前画布
canvas.rotate(); // 旋转当前画布
// ...平移、旋转后在做一系列操作

canvas.restore();  // 合并两次的操作
```



调用view的requestLayout方法会触发整个View树的重新测量、布局以及重绘。

调用某个View的`requestLayout` 方法时会一直遍历到根节点，并给View（或者ViewGroup）打上**PFLAG_FORCE_LAYOUT** 标记，最终调用到ViewRootImpl的`performTraversals` 方法。进行测量、布局以及重绘。由于layoutRequested为True，三个主要步骤都会进行。



调用View的`invalidate` 方法时，也会一直遍历到根View，打上**PFLAG_DIRTY**标记并求得各个View（或者ViewGroup）的dirty区域的并集。最终同样调用到ViewRootImpl的`performTraversals` 方法。只会触发重绘操作。



对于设置了标志位为：PFLAG_FORCE_LAYOUT的View进行测量，测量