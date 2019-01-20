### Android事件分发

View的dispatchTouchEvent在进行事件分发的时候，只有前一个action返回true，才会触发后一个action。

设置视图的 WindowManager 布局参数的 flags为`FLAG_WATCH_OUTSIDE_TOUCH`，这样点击事件发生在这个视图之外时，该视图就可以接收到一个 `ACTION_OUTSIDE` 事件。点击 Dialog 区域外关闭。



##### ViewGroup的事件分发处理

ViewGroup对于事件的处理，有以下方法：

- dispatchTouchEvent，事件分发
- onInterceptTouchEvent，事件拦截，只有ViewGroup有该方法
- onTouchEvent，事件消费

View对于事件的处理有如下方法：

- dispatchTouchEvent，事件分发
- onTouchEvent，事件消费

`dispatchTouchEvent` 和 `onInterceptTouchEvent` 每次事件传递过来都会调用，`onTouchEvent` 只有决定消费后才将后续事件传递过来。

假设我们在一个LinerLayout（ViewGroup）中放置了一个Button（View），触摸这个Button按钮，整个事件的分发流程如下：（暂不考虑Activity的事件分发流程）

ViewGroup的`dispatchTouchEvent`先收到该事件，根据自己的`onInterceptTouchEvent`决定是否拦截该事件：

- 不拦截，调用View的`dispatchTouchEvent` ，交给View处理
- 拦截该事件，调用自身的`onTouchEvent`的消费该事件

伪代码示例如下：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean result = false;             // 默认状态为没有消费过

    if (!onInterceptTouchEvent(ev)) {   // 如果不拦截该事件交给子View处理
        result = child.dispatchTouchEvent(ev);
    }

    if (!result) {                      // 根据View的处理结果，决定是否调用自身的onTouchEvent
        result = onTouchEvent(ev);
    }

    return result;
}
```

由伪代码可见知，ViewGroup自身的`onTouchEvent`是否调用，即取决于自己的`onInterceptTouchEvent` 是否拦截该事件，也取决于子View对事件的分发处理 `dispatchTouchEvent` 结果，如果子View的`dispatchTouchEvent`  要返回false，表示没有消费该事件，就会调用ViewGroup的`onTouchEvent`。

> 子View可以在`dispatchTouchEvent` 通过：
>
> getParent().requestDisallowInterceptTouchEvent(boolean ); 
>
> 来控制ViewGroup对事件的拦截状态，即使ViewGroup拦截了事件，我们可以通过语句：
>
> getParent().requestDisallowInterceptTouchEvent(true);
>
> 来取消ViewGroup对事件的拦截 

##### View的事件分发处理

当事件成功传入到了View中，首先收到该事件的就是`dispatchTouchEvent` 方法，先看一下该方法的源码：

```java
// ...
ListenerInfo li = mListenerInfo;
if (li != null && li.mOnTouchListener != null
         	&& (mViewFlags & ENABLED_MASK) == ENABLED
          	&& li.mOnTouchListener.onTouch(this, event)) {
     result = true;
}

 if (!result && onTouchEvent(event)) {
	 result = true;
}
//...
return result;

```

以上代码有省略，只看我们关注的：

如果用户为View设置了`OnTouchListener`，并在该类的`onTouch`方法中返回了true，表示事件已经被消费了，则就不会执行到View的`onTouchEvent`方法了。



简单分析了，ViewGroup和View的事件分发过程。总结一下：

- 根据触摸的View，事件先由含有该View的ViewGroup进行处理，由ViewGroup决定是否将当事件传递给子View
- 当事件传递到了子View（假设用户没有设置onTouchListener），如果`onTouchEvent`的返回值true，则事件由该子View消费，后续的`down`、`up`事件会传递过来；如果返回false，则不消耗该事件，事件回传到VieGroup，后续的`down`、`up`事件也不会再传递过来

#####View事件的详细分发过程

在View中，有这么几个对事件进行相应的方法：onTouchListener、onTouchEvent、onLongClickListener、onClickListener。这几个方法对事件的响应顺序是怎么样的呢？

1、onTouchListener，由用户设置的事件监听器，监听触摸事件，肯定会第一个被执行；

2、onTouchEvent，View自身的触摸事件回调，第二个被执行；

3、onLongClickListener，用户设置的长按事件监听器，不需要ACTION_UP事件，就要被回调，**要先于`onClickListener`执行**

4、onClickListener，用户设置的单击事件监听器，ACTION_UP后才会回调，最后一个被执行



推荐参考：[这可能是最好的事件分发讲解](http://www.jianshu.com/p/2be492c1df96)