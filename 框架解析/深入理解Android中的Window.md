### 深入理解Android中的Window

#### 1、概述

Android中的Window用一句话概述，就是管理Android中各个控件接受用户的响应并做出相应的处理。

#### 2、通过WindowManager添加一个控件

```java
// 获取WindowManager的实例
WindowManager wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);

// 新建一个按钮
Button btn = new Button(mContext);
btn.setText("hello");

// 新建一个WindowManager.LayoutParams，描述窗口的类型与位置信息
WindowManager.LayoutParams lp = new WindowManager.LayoutParams();
lp.type = WindowManager.LayoutParams.TYPE_PHONE;
lp.gravity = Gravity.CENTER;
lp.width = WindowManager.LayoutParams.WRAP_CONTENT;
lp.height = WindowManager.LayoutParams.WRAP_CONTENT;
lp.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |
  WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;

// 通过WindowManager的addView，将按钮作为一个窗口添加到系统中
wm.addView(btn,lp);

// 通过WindowManager的removeView，将按钮从系统中移除
//wm.removeView(btn);
```

注意：要在AndroidManifest中加入权限申请：

```java
<uses-permission  android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

在Android6.0以后，这个权限是危险权限，需要动态去申请，提示用户“该应用会悬浮于其他窗口之上”。

在入口Activity的onCreate方法中加入如下代码，引导用户去勾选允许应用弹出浮窗：

```java
    if (Build.VERSION.SDK_INT >= 23) {
      if (!Settings.canDrawOverlays(this)) {
        Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivityForResult(intent, 1);// 跳转到设置界面
      } else {
        // TODO do something you need
      }
    }
```



由于Button继承自View，通过WindowManager添加到系统中。（任何继承自View的空间都可以作为窗口添加到系统中）

WindowManager可以看作是WMS在客户端的代理类，其实，通过通过WindowManager已经看不到WMS的影子了。WindowManager精简了大量WMS的接口，便于客户端调用。

WindowManagerService负责Window的管理：

- 窗口的创建和销毁
- 窗口的显示与隐藏
- 窗口的布局
- 窗口的Z-Order管理
- 焦点的管理
- 输入法和壁纸管理