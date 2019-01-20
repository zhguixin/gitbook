介绍通过WindowManager添加View的操作流程。

#### WindowManager

在应用开发中通过如下语句来添加一个窗口：

```java
WindowManager mManager = mConext.getSystemService(Context.WINDOW_MANAGER);
mManager.addView(myView, mLayoutParams);
```

其中，WindowManger是一个接口，继承自ViewManager这一个接口：

```java
public interface ViewManager {
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

WindowManagerImpl实现了WindowManager接口，在`WindowManagerImpl` 又将具体的实现交给了**WindowManagerGlobal** ，在`WindowManagerGlobal`中，窗口的添加、删除、布局更新都交给了ViewRootImpl。

> WindowManager承载了用户在APP开发中对窗口的主要操作（addView、updateViewLayout、removeView这三个方法定义在ViewManager接口中，WindowManager继承自该接口）。WindowManager是一个接口，他的实现类是WindowManagerImpl，但是在WindowManagerImpl中主要工作交给了WindowManagerGlobal。

ViewRootImpl在WindowManagerGlobal 的`addView` 方法中被实例化：

```java
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    
    root = new ViewRootImpl(view.getContext(), display);
    view.setLayoutParams(wparams);
    
    // 分别添加到三个ArrayList的数组中
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
    
    root.setView(view, wparams, panelParentView);
}
```

ViewRootImpl的生命周期从`setView` 方法开始。

#### ViewRootImpl

先来看一下，实例化ViewRootImpl的构造函数中初始化的类变量：

```java
public ViewRootImpl(Context context, Display display) {
    // IWindowSession类的实例，通过它实现ViewRootImpl与WMS进行通信
    mWindowSession = WindowManagerGlobal.getWindowSession();
    // 当前线程，即UI线程，只允许UI线程操作UI，就要与此值进行判断
    mThread = Thread.currentThread();
    // 窗口中的无效区域
    mDirty = new Rect();
    // 描述当前窗口的位置和尺寸
    mWinFrame = new Rect();
    
    mWindow = new W(this);
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
}
```

紧接着看看`setView` 方法：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            /** 主要流程 **/
            // 触发一次控件树的遍历
            requestLayout();
            // 向WMS添加窗口
            res = mWindowSession.addToDisplay(/**/);
            // ViewRootImpl作为所有的父View
            view.assignParent(this);
        }
    }
}
```

其中，requestLayout调用过程如下：

```java
requestLayout() -->> scheduleTraversals() -->> doTraversal() -->> performTraversals()
```

`doTraversal` 由Choreographer根据VSYNC信号来调用。

#### performTraversals方法

`performTraversals` 方法包含了控件树ew的测量、布局和绘制，与WMS进行交互，最终确定View的大小、位置和绘制。主要可以分为五个阶段：

* 预测量阶段，
* 布局窗口阶段，向WMS请求，对应于WMS的`relayout` 方法
* 最终测量阶段，受到WMS的影响，需要重新测量一番
* 布局控件树阶段，对控件进行布局
* 绘制阶段，对控件进行绘制

```java
private void performTraversals() {
    
    // View预测量阶段期望的大小
    int desiredWindowWidth;
    int desiredWindowHeight;
    // 是否首次调用
    if (mFirst) {
        // 屏幕尺寸大小
    } else {
        // 由WMS更新后的mWinFrame值
    }
    // 调用requestLayout方法，该值为true，需要进行重新测量
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if(layoutRequested) {
        // ...
        // 返回值为true表明了WMS在对窗口布局时，更改了尺寸，与测量结果有出入，需要调用WMS的
        // reLayoutWindow方法
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
    }
    
    // 如果要再次触发测量，需要调用requestLayout方法为mLayoutRequested赋值为true
    if (layoutRequested) {
        mLayoutRequested = false;
    }
}
```

其中，`measureHierarchy` 方法：

```java
private boolean measureHierarchy(/**/) {
    // 测量能否满足控件树充分显示内容的要求
    boolean goodMeasure = false;
    // lp值是应用程序调用addView的时候赋值的
    // 对WRAP_CONTENT的值要进行3次协商测量，不断放大宽高值
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
    }
    
    if (!goodMeasure) {
        // MATCH_PARENT：测量值为desiredWindowWidth，测量模式为EXACTLY
        // 精确的dp值：测量值为dp值，测量模式为EXACTLY
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        // 调用View的measure方法，回调onMeasure
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }    
}
```

View自身的测量：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    // 设置了PFLAG_FORCE_LAYOUT标记,子控件内容变化向上回溯调用requestLayout方法时会设置此值
    // 或者需要重新测量布局（传进来的值与旧值不相等时）
    if (forceLayout || needsLayout) {
        // ...
        // 具体交由View自身去测量计算
        onMeasure();
        // 便于后续调用layout进行布局
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
    // 便于下次比较，求得needsLayout
    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;
}
```

第二个步骤，布局窗口阶段：

```java
private void performTraversals() {
    // 预测量代码
    
    // 条件判断
    if (mFirst || windowShouldResize || insetsChanged ||
        viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        // 会调用WMS进行窗口布局
        // 在WMS中会申请Surface，保存到mSurface
        relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
    }
}
```

第三个步骤，最终测量阶段：

```java
private void performTraversals() {
    // 1
    // 2
    
    if (!mStopped || mReportNextDraw) {
        if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
            || mHeight != host.getMeasuredHeight() || framesChanged ||
            updatedConfiguration) {
            // mWidth、mHeight由WMS对布局窗口后的调整值
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
            // 无须协商，直接测量
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            // ...
        }
    }
}
```

第四个步骤，控件布局阶段：

```java
private void performTraversals() {
    // 1
    // 2
    // 3
    // 要设置layoutRequested为true
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        // 调用View的layout方法，最终回调onLayout方法
        performLayout(lp, mWidth, mHeight);
    }
    
}
```

第五个步骤，绘制阶段：

见：View的绘制



Window主要就是接受用户的输入、并将一些回调传递给View。比如在Activity中包含了一个PhoneWindow类，在PhoneWindow中又包含了DecorView。这个DecorView是一个继承自FrameLayout的自定义组件，用来展示用户想要显示的内容。

DecorView是顶层View，一般是包含一个垂直方向的LinearLayout，该LinearLayout分为上下两部分，上部分放置title，下面显示内容content(android.R.id.content)，我们在Activity中调用`setContentView`就是将一个View放入到了内容区域content中。

##### 关于Window与View的关系

先来了解下，Window的绘制。Window的绘制就是在系统的分配的一块Surface区域中的矩形区域内进行操作。当Window绘制的时候，先对Surface加锁，并在Surface返回的Canvas对象中绘制，绘制完成后，Surface解锁。解锁后，将绘制的内容放到缓存中，由Surface Flinger来组织。

其中，对触发Window进行绘制依靠的就是View（当View调用invalidate时），具体的绘制操作就是将Canvas传递给View，View对Canvas中进行操作来实现具体的绘制操作的。

可见，对Android系统底层来说，View的概念是不存在的。底层只知道有Window这么个东西（这也可以解释框架里为什么是WindowManagerService而不是ViewManagerService了）。

##### Activity与Window的关系

在Activity中有一个`setContentView`的方法，源码如下：

```java
public void setContentView(View view) {
    getWindow().setContentView(view);
    initWindowDecorActionBar();
}
```

其中`getWindow`返回了成员变量`mWindow`，赋值的地方在Activity的attach方法中：

```java
mWindow = new PhoneWindow(this, window);
```

PhoneWindow继承Window这个抽象类，看看PhoneWindow的`setContentView`方法的实现：

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    }
    // ...
}
```

创建DecorView，将DecorView添加到Window（即PhoneWindow），通过调用getViewRootImpl（DecorView继承自FrameLayout，getViewRootImpl是View类中的**hide**方法）与ViewRootImpl相关联。



主要的操作流程都在`ViewRootImpl`中的`performTraversals`方法中。涉及到：`measure`、`layout`、`draw`。

##### View的measure过程

ViewGroup中放置一个View，是先测量自己还是测量自身呢。ViewGroup的onMeasure是让用户自己实现的。

根据测量模式，调用measureChildren测量子View。计算每个子View的宽高，最终根据测量模式决定自己的宽高：

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        // 获得父容器为它设置的测量模式和大小
        int sizeWidth = MeasureSpec.getSize(widthMeasureSpec);
        int sizeHeight = MeasureSpec.getSize(heightMeasureSpec);
        int modeWidth = MeasureSpec.getMode(widthMeasureSpec);
        int modeHeight = MeasureSpec.getMode(heightMeasureSpec);
        
        int cCount = getChildCount();
        // 遍历每个子元素
        for (int i = 0; i < cCount; i++) {
            View child = getChildAt(i);
            // 测量每一个child的宽和高
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
            // 得到child的lp
            MarginLayoutParams lp = (MarginLayoutParams) child
                    .getLayoutParams();
            // 当前子空间实际占据的宽度
            int childWidth = child.getMeasuredWidth() + lp.leftMargin
                    + lp.rightMargin;
            // ...
        }
        
        // width、height是根据子 view 的宽高计算出来的
        setMeasuredDimension((modeWidth == MeasureSpec.EXACTLY) ? sizeWidth
                : width, (modeHeight == MeasureSpec.EXACTLY) ? sizeHeight
                : height);        
    }
```

widthMeasureSpec是一个32位的int值，高两位存储测量模式：

* UNSPECIFIED
* EXACTLY，对应于LayoutParams的match_parent和精确dp值
* AT_MOST，对应于LayoutParams的wrap_content

低30位存储某种测量模式下的大小。

我们在定义View时，是指定LayoutParams，系统会根据父容器的MeasureSpec和我们指定的LayoutParams来共同决定该View的MeasureSpec。MeasureSpec一旦确定，在onMeasure中就可以确定View的测量宽/高。



##### View的刷新流程

View的刷新，比如调用了`invalidate` 方法，会层层找到parent，计算得到需要重新绘制的区域，最终到ViewRootImpl（View树的根），在ViewRootImpl中控制了整个View数的刷新，只需要绘制需要重绘的视图（只调用draw方法）。不会设置**mLayoutRequested **的值。

scheduleTraversals() ---> doTraversal() ---> performTraversals()

View调用`requestLayout` 方法会层层传递到ViewRootImpl的`requestLayout` 方法，设置**mLayoutRequested ** 为true，调用`scheduleTraversals` 方法，触发一次控件树变量测量和布局。