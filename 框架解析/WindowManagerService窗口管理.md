### 概念

在WindowManagerService中，有几个重要的成员变量：

- mTokenMap，HashMap<IBinder, WindowToken>

  应用组件要想向WMS添加窗口(调用addWindow方法)，必须申请一个token，IBinder类型。WMS为Activity特定了AppWindowToken。

- mWindowMap，HashMap<IBinder, WindowState>

  WindowState表示了一个窗口的所有属性，向WMS添加一个窗口时，就会将新建了WindowState添加到mWindowMap中，该集合是整个系统所有窗口的一个全集。

WindowToke将属于同一应用组件的窗口整合在一起。比如，一个Activity上弹出的对话框，这两个窗口属于同一个Token，但是是两个不同的WindowState。

> 两个集合的key值都是IBinder类型，但是也是有区别的：
>
> mTokenMap的key值可以是IAppWindowToken的Bp端，也可以是任意一个Binder的Bp端；
>
> mWindowMap的key值一定是IWindow的Bp端。

### 窗口显示次序

窗口的显示次序，又称为Z序。屏幕Z轴为垂直于屏幕指向屏幕外。

显示次序又由：主序mBaseLayer和次序mSubLayer决定。这两个值定义在WindowState类中。

* 主序决定父窗口位置，主序越大越靠前；
* 子序决定兄弟窗口位置，子序越大越靠前

在WMS中，`addWindow()`方法中：会依次调用`addWindowToListInOrderLocked` 确定主序与子序，放入到DisplayContent的mWindows列表中；再调用`assignLayersLocked` 方法(位于WindowLayersController类中)最终确定显示次序。

### 窗口布局

窗口都是绘制在一块Surface上的，从WMS来看，它的入口就在`relayoutWindow` 这个方法中。

```java
public int relayoutWindow(/**/) {
    // 修改WindowState中的一些值
    
    // 有必要的话创建Surface
    result = createSurfaceControl(outSurface, result, win, winAnimator);
    
    // 重新确定显示次序
    mLayersController.assignLayersLocked(win.getWindowList());
    
    // 告知DisplayContent窗口需要布局
    win.setDisplayLayoutNeeded();
    // 重新布局
    mWindowPlacerLocked.performSurfacePlacement();
    
    // 将布局结果通过传过来的函数参数，传递到应用进程
    // ...
}
```

#### 窗口的创建时机和过程

窗口的创建对开发者来说，一个是通过调用`WindowManager`的`addView()`方法显示创建，另外一个就是启动一个新的Activity、Dialog、Toast隐式的创建（这种隐式的创建也是通过调用`addView()`方法实现的）。

每调用一次`addView()`方法，就会新建一个**ViewRootImpl**，最终通过其内部的成员变量`mWindowSession`（Session类的Bp端）调用`addToDisplay()`方法，调用到WMS的`addWindow()`方法。

在WMS的`addWindow()`方法中，对窗口的添加主要经历了三个大的过程：

- 第一个过程，前置处理，检查窗口添加的合法性，比如根据token判断窗口添加的合法性
- 第二个过程，添加窗口相关数据，更新相关成员变量，包括：`mWindowMap`、`mTokenMap`等
- 第三个过程，处理窗口添加完毕后，可能引起的状态变化，比如：输入法重新获取焦点窗口、重新确定窗口的层序

> 详细需要参考《Android内核剖析》第十四章

