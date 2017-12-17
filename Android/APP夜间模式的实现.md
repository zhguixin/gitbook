APP夜间模式的实现

夜间模式的实现方案有多种，这里采用重启Activity，重新设置Theme的方式。

首先定义两个Theme，分别为DayTheme、NightTheme。

> 自定义的属性在`attrs.xml`中定义好后，在布局中使用：
>
> ```xml
> android:background="?attr/mainBackground"
> ```
>
> 在DayTheme和NightTheme中，分别对mainBackground设置不同的颜色值（或drawable）

在电影天堂APP中，监听开关切换状态。存储当前的主题值到本地（通过SharedPreference），并发送广播，通知MainActivity重建。

在MainActivity中，注册监听广播主题变化，触发重建：

```java
  skinChangeReceiver = new SkinChangeReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
      searchFragment = null;
      movieFragment = null;
      aboutFragment = null;
      recreate(); // 重建Activity
    }
  };
registerSkinReceiver(MainActivity.this, skinBroadcastReceiver);
```

在Activity重建之后，后调用**onSaveInstanceState(Bundle outState)**保存数据，由于夜间模式的设置是在`AboutFragment`中，因此需要在ManiActivity中的`onCreate`判断**savedInstanceState**是否为null，来设置BottomNavigationView的焦点：

```java
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    // ...
    if (savedInstanceState != null) {
      	mItemId = savedInstanceState.getInt(savedInstanceStateItemId);
      	mTitle = savedInstanceState.getString(savedInstanceStateTitle);
        mBottomView.setChecked(aboutFragmentItem);
    }
    //...
  }

  // 如果有数据保存
  @Override
  protected void onSaveInstanceState(Bundle outState) {
    outState.putInt(savedInstanceStateItemId, navigationCheckedItemId);
    outState.putString(savedInstanceStateTitle, navigationCheckedTitle);
    super.onSaveInstanceState(outState);
  }
```

样式：

```xml
<color name="item">#ffffff</color>
<color name="itemNight">#979696</color>

<item name="itemBg">@color/item</item>
<item name="itemBg">@color/itemNight</item>
```
