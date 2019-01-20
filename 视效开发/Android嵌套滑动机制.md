某个View对于事件的处理，是有始有终的。拿到TOUCH_DOWN事件，后续的事件就都会传递给此View。

以ScrollView嵌套ListView为例。为了能够持续滚动。为了确保嵌套的多个View。



CoordinatorLayout 作为根布局，就协调它子控件之间的联动效果。子控件间如何联动，通过对子控件配置 `app:layout_behavior` 属性。该属性指向自定义的Behavior类。

> CoordinatorLayout是android support design推出的新布局，要在build.gradle中配置：
>
> ```groovy
> implementation 'com.android.support:design:26.1.0'
> ```

```xml
<android.support.design.widget.CoordinatorLayout>
	<android.support.design.widget.AppBarLayout/>
	<android.support.v4.widget.NestedScrollView
 	 app:layout_behavior=".AppBarLayout$ScrollingViewBehavior"/>
</android.support.design.widget.CoordinatorLayout>
```

在NestedScrollView中指定的Behavior为AppLayout的内部类ScrollingViewBehavior：

```java
public static class ScrollingViewBehavior extends HeaderScrollingViewBehavior {
    // 第二个参数：NestedScrollView，当前View，即NestedScrollView
    // 第三个参数：AppBarLayout，要关心的那个View，即：AppBarLayout
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency);
    // 当我们关心的那个View发生了变化，当前View需要做哪些改变
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child,View dependency);
    
}
```



**使用Behavior，我们就必须使用CoordinatorLayout布局，并且我们所有希望带Behavior操作的控件都必须要是CoordinatorLayout的直接子View**

联动的实现原理是什么？

CoordinatorLayout 继承自`NestedScrollingParent` 接口。

可滑动的组件比如：RecyclerView、NestedScrollView继承自`NestedScrollingChild`接口。

嵌套滑动的原理就是，可滑动的子View会将事件传递给父View进行消耗处理，然后子View也会处理剩下的事件。

CoordinatorLayout只是将传过来的数据进行了一次封装，分发给了自己子布局里面所有含有Behavior的控件，交由它们进行处理。

当 RecyclerView 或 NestedScrollView 滑动时，CoordinatorLayout 的子控件 Behavior 可以接收到对应的回调：

- onStartNestedScroll，返回值决定是否接收嵌套滑动事件
- onNestedPreScroll，在准备滚动之前调用的，它带有滚动偏移量 `dx`、`dy`；`consumed` 这个参数，它是一个长度为 2 的数组，存放的是要消费掉的 x 和 y 轴偏移量
- onStopNestedScroll，取消了嵌套滑动时的回调
- onNestedPreFling，



在 Behavior 中，通过 layoutDependsOn 方法来建立依赖关系，一个控件可以依赖多个其他控件，但不可循环依赖。当被依赖的控件属性发生变化时，会调用 onDependentViewChanged 方法。



要想让子View的事件传递出去，就要继承`NestedScrollingChild` 并实现相关方法。

在以`CoordinatorLayout ` 为根布局的情况下，要想让另外一个子View联动，需要实现`CoordinatorLayout .Behavior`类。事件由`CoordinatorLayout ` 传递过来。



RecyclerView需要关注HeaderView的位移，实现Behavior的特定方法：

```java
@Override
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
    return dependency instanceof HeaderView;
}

@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
	// ...
    return super.onDependentViewChanged(parent, child, dependency);
}
```



参考资料：

[Android 嵌套滑动机制（NestedScrolling）](https://segmentfault.com/a/1190000002873657)

[CoordinatorLayout中的Behavior介绍和使用系列](https://segmentfault.com/a/1190000006666005)

[Android 优化交互 —— CoordinatorLayout 与 Behavior](https://segmentfault.com/a/1190000005024216)

[Android NestedScrolling全面解析](https://www.jianshu.com/p/f09762df81a5)



