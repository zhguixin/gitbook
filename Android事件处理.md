Android事件处理

在上一节中分析了Android事件分发的过程，这一节分析一下具体的事件处理过程。

#### 基本事件

Android将所有的输入事件都放在了MotionEvent中：

`base\core\java\android\view\MotionEvent.java` 

事件包括：ACTION_DOWN、ACTION_MOVE、ACTION_UP、ACTION_CANCEL、ACTION_OUTSIDE。

* ACTION_CANCEL：后续事件不再向View传递了，View会收到该事件
* ACTION_OUTSIDE：WindowManager设置的布局参数(WindowManager.LayoutParams) flag 指定属性`FLAG_WATCH_OUTSIDE_TOUCH`，点击事件发生在视图之外时，可以收到该事件

事件相关的方法：getX()、getY()、getRawX()、getRawY()、getAction()

其中，getX和getY是相对于View的坐标位置，getRaw是相对于屏幕坐上角的位置。

基本处理过程：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    // 使用 getAction()，获取事件类型
    switch (event.getAction()){
        case MotionEvent.ACTION_DOWN:
        	// 手指按下
            break;
        case MotionEvent.ACTION_MOVE:
            // 手指移动
            break;
        case MotionEvent.ACTION_UP:
            // 手指抬起
            break;
        case MotionEvent.ACTION_CANCEL:
            // 事件被拦截 
            break;
        case MotionEvent.ACTION_OUTSIDE:
            // 超出区域 
            break;
    }
    return super.onTouchEvent(event);
}
```

#### 多点触控

新增了两个事件：

* ACTION_POINTER_DOWN，有非主要的手指按下(**即按下之前已经有手指在屏幕上**)
* ACTION_POINTER_UP，有非主要的手指抬起(即抬起之后仍然有手指在屏幕上)

新增事件相关方法：

* getActionMasked()，**多点触控时必须使用 getActionMasked() 来获取事件类型**

