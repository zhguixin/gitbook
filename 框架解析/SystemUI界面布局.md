基于Android7.1。

#### UI布局

在PhoneStatusBar中的`createAndAddWindows` 完成SystemUI窗口以及布局的添加。

```java
@Override
public void createAndAddWindows() {
    addStatusBarWindow();
}

private void addStatusBarWindow() {
    makeStatusBarView();// 创建控件树，mStatusBarWindow会被inflate出来，即StatusBarWindowView
    mStatusBarWindowManager = new StatusBarWindowManager(mContext);
    mRemoteInputController = new RemoteInputController(mStatusBarWindowManager,
                                                       mHeadsUpManager);
    // 实际上是调用WindowManager来添加状态窗口，窗口类型是TYPE_STATUS_BAR
    mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
}

public void add(View statusBarView, int barHeight) {

    // Now that the status bar window encompasses the sliding panel and its
    // translucent backdrop, the entire thing is made TRANSLUCENT and is
    // hardware-accelerated.
    mLp = new WindowManager.LayoutParams(
        ViewGroup.LayoutParams.MATCH_PARENT,
        barHeight,
        WindowManager.LayoutParams.TYPE_STATUS_BAR,
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
        | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
        | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
        | WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS,
        PixelFormat.TRANSLUCENT);
    mLp.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
    mLp.gravity = Gravity.TOP;
    mLp.softInputMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
    mLp.setTitle("StatusBar");
    mLp.packageName = mContext.getPackageName();
    mStatusBarView = statusBarView;
    mBarHeight = barHeight;
    mWindowManager.addView(mStatusBarView, mLp);
    mLpChanged = new WindowManager.LayoutParams();
    mLpChanged.copyFrom(mLp);
}
```

接着看看控件树的创建过程。首先是`super_status_bar.xml`被inflate出来，**StatusBarWindowView**是整个SystemUI的顶层View。

```xml
<StatusBarWindowView>
    <BackDropView/> <!--gone-->
    <ScrimView/><!--behind-->
    <AlphaOptimizedView/>
    <include layout="@layout/status_bar"/><!--height：读取配置文件8dp-->
    <include layout="@layout/brightness_mirror" />
    
    <ViewStub android:id="@+id/fullscreen_user_switcher_stub"
              android:layout="@layout/car_fullscreen_user_switcher"/>
    
    <include layout="@layout/status_bar_expanded" /><!--gone，height：match_parent-->
    <ScrimView /> <!--front-->
</StatusBarWindowView>
```

StatusBarWindowView是整个状态栏布局的根布局，它继承自FrameLayout。正常情况下，只有status_bar的布局是可见的。`status_bar.xml` ：

```xml
<com.android.systemui.statusbar.phone.PhoneStatusBarView>
	<ImageView/> 低辨识度模式下的View
	<LinearLayout>
        <include layout="@layout/system_icons" /> 系统图标
        //...
	</LinearLayout>
                                                         </com.android.systemui.statusbar.phone.PhoneStatusBarView>
```

PhoneStatusBarView的继承关系如下：

**FrameLayout <-- PanelBar <-- PhoneStatusBarView**

在通知栏下拉展开后对应的布局文件为：`status_bar_expanded.xml` ：

```xml
<com.android.systemui.statusbar.phone.NotificationPanelView>
    <com.android.keyguard.CarrierText/>
    <include // 锁屏界面上的时钟
        layout="@layout/keyguard_status_view" 
        android:layout_height="wrap_content"
        android:visibility="gone" />
    // 下拉通知快捷设置界面
	<com.android.systemui.statusbar.phone.NotificationsQuickSettingsContainer
        <com.android.systemui.AutoReinflateContainer
            android:id="@+id/qs_auto_reinflate_container"
            android:layout="@layout/qs_panel" />
    	// 通知显示区域
		<com.android.systemui.statusbar.stack.NotificationStackScrollLayout/>
         <include
            layout="@layout/keyguard_status_bar"
            android:visibility="invisible" />
	</com.android.systemui.statusbar.phone.NotificationsQuickSettingsContainer>
	// 锁屏界面底部显示图标，相机、紧急电话
    <include
            layout="@layout/keyguard_bottom_area"
            android:visibility="gone" />
</<com.android.systemui.statusbar.phone.NotificationPanelView>                                               
```

`NotificationPanelView` 的继承关系：

**FrameLayout <-- PanelView <-- NotificationPanelView**

#### 导航栏添加过程

在`makeStatusBarView()`方法中：

```java
boolean showNav = mWindowManagerService.hasNavigationBar();
if (DEBUG) Log.v(TAG, "hasNavigationBar=" + showNav);
// 决定是否需要添加虚拟导航栏，有framework层的config.xml中的值决定
if (showNav) {
    createNavigationBarView(context);
}
```

导航栏对应的布局文件为：`navigation_bar.xml`，即**NavigationBarView**，该自定义View继承自LinerLayout。

inflate出来**NavigationBarView**后赋值给`mNavigationBarView`，执行`addNavigationBar()`函数：

```java
protected void addNavigationBar() {
    if (DEBUG) Log.v(TAG, "addNavigationBar: about to add " + mNavigationBarView);
    if (mNavigationBarView == null) return;
    // 为各个虚拟按键注册onTouchListener、onLongClick事件
    prepareNavigationBarView();
    // 添加导航栏窗口
    mWindowManager.addView(mNavigationBarView, getNavigationBarLayoutParams());
}
// 窗口类型TYPE_NAVIGATION_BAR
private WindowManager.LayoutParams getNavigationBarLayoutParams() {
    WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
        LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT,
        WindowManager.LayoutParams.TYPE_NAVIGATION_BAR,
        0
        | WindowManager.LayoutParams.FLAG_TOUCHABLE_WHEN_WAKING
        | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
        | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
        | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
        | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH,
        PixelFormat.TRANSLUCENT);
    // this will allow the navbar to run in an overlay on devices that support this
    if (ActivityManager.isHighEndGfx()) {
        lp.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
    }

    lp.setTitle("NavigationBar");
    lp.windowAnimations = 0;
    return lp;
}
```

> Android的三大虚拟按键，home键、back键并没有注册onClick事件。这是因为这两个虚拟按钮在`back.xml` 和`home.xml`中分别指定了Keycode为：4、3，这两个xml文件由**KeyButtonView**（继承自ImageView的自定义View）实现，在**KeyButtonView**中最终封装成KeyEvent发送给InputManger处理

#### 事件分发

既然`StatusBarWindowView` 作为一个根布局，对事件的分发就显得尤为重要。

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    mFalsingManager.onTouchEvent(ev, getWidth(), getHeight());
    if (mBrightnessMirror != null && mBrightnessMirror.getVisibility() == VISIBLE) {
        // Disallow new pointers while the brightness mirror is visible. This is so that you
        // can't touch anything other than the brightness slider while the mirror is showing
        // and the rest of the panel is transparent.
        // 亮度条显示时，第二根手指的事件完全忽略掉
        if (ev.getActionMasked() == MotionEvent.ACTION_POINTER_DOWN) {
            return false;
        }
    }
    // 是否需要关闭通知栏
    if (ev.getActionMasked() == MotionEvent.ACTION_DOWN) {
        mStackScrollLayout.closeControlsIfOutsideTouch(ev);
    }

    return super.dispatchTouchEvent(ev);
}

// 每个事件都会调用 onInterceptTouchEvent
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean intercept = false;
    // 拦截事件的条件:状态栏展开，通知栏显示。锁屏状态，BouncerView没有显示
    if (mNotificationPanel.isFullyExpanded()
        && mStackScrollLayout.getVisibility() == View.VISIBLE
        && mService.getBarState() == StatusBarState.KEYGUARD
        && !mService.isBouncerShowing()) {
        intercept = mDragDownHelper.onInterceptTouchEvent(ev);
        // wake up on a touch down event, if dozing
        if (ev.getActionMasked() == MotionEvent.ACTION_DOWN) {
            mService.wakeUpIfDozing(ev.getEventTime(), ev);
        }
    }
    // 上面判断不需要拦截的话，交给父view即FrameLayout判断
    if (!intercept) {
        super.onInterceptTouchEvent(ev);
    }
    // 需要拦截事件的话，下发一个ACTION_CANCEL事件
    if (intercept) {
        MotionEvent cancellation = MotionEvent.obtain(ev);
        cancellation.setAction(MotionEvent.ACTION_CANCEL);
        mStackScrollLayout.onInterceptTouchEvent(cancellation);
        mNotificationPanel.onInterceptTouchEvent(cancellation);
        cancellation.recycle();
    }
    return intercept;
}

// 事件被拦截后的具体处理
@Override
public boolean onTouchEvent(MotionEvent ev) {
    boolean handled = false;
    if (mService.getBarState() == StatusBarState.KEYGUARD) {
        // 上面就是在锁屏状态下拦截的，再次确保还是在锁屏状态下，交给DragDownHelper处理
        // DragDownHelper会在锁屏上下拉时，展开通知
        handled = mDragDownHelper.onTouchEvent(ev);
    }
    if (!handled) {
        handled = super.onTouchEvent(ev);
    }
    final int action = ev.getAction();
    if (!handled && (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL)) {
        mService.setInteracting(StatusBarManager.WINDOW_STATUS_BAR, false);
    }
    return handled;
}
```

接下来看`status_bar` ，它的根布局是`PhoneStatusBarView` 对事件的处理不多，作为顶部一个高度固定的显示区域，显示的信息有：时间、电量、信号等；

再看`status_bar_expanded` ，它的根布局是`NotificationPanelView` ，这个自定义View可谓是包罗万象。包括锁屏界面信息、下拉通知栏快捷设置、通知条信息展示等。

PanelBar是PhoneStatusBarView的父View，PanelView是NotificationPanelView的父View。

PanelBar持有PanelView的实例，在收到TouchDown事件后，会调用PanelView进行事件处理。

在`PanelBar` 的onTouchEvent中：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    // Allow subclasses to implement enable/disable semantics
    if (!panelEnabled()) {
        return false;
    }

    if (event.getAction() == MotionEvent.ACTION_DOWN) {
        final PanelView panel = mPanel;
        if (panel == null) {
            // panel is not there, so we'll eat the gesture
            return true;
        }
        boolean enabled = panel.isEnabled();
        if (!enabled) {
            // panel is disabled, so we'll eat the gesture
            return true;
        }
    }
    // 调用PanelView的onToucheEvent方法
    return mPanel == null || mPanel.onTouchEvent(event);
}
```

在PanelView的onTouchEvent方法中，会根据手机下拉的距离来慢慢展开下拉通知栏。