ViewDragHelper使用

1、侧滑返回的实现

大体思路：写一个Activity，所有希望支持侧滑返回的Activity继承自该类：

```java
public class SwipeBackActivity extends AppCompatActivity {
  
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      
      this.getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
      this.getWindow().getDecorView().setBackgroundDrawable(null);
      mSwipeBackLayout = (SwipeBackLayout) LayoutInflater.from(this).inflate(
                R.layout.swipeback_layout, null);
      mSwipeBackLayout.attachToActivity(this);
    }
  
}
```

可以看到，要想实现侧滑返回，重点在与实现`SwipeBackLayout`，这是一个自定义的ViewGroup。

首先看一下，在SwipeLayout中的`attachToActivity`方法的实现：

```java
  public void attachToActivity(Activity activity) {
    mActivity = activity;
    TypedArray a = activity.getTheme().obtainStyledAttributes(new int[]{
      android.R.attr.windowBackground
    });
    int background = a.getResourceId(0, 0);
    a.recycle();
	
    // decor表示，setContentView所设置View的父View
    ViewGroup decor = (ViewGroup) activity.getWindow().getDecorView();
    // 得到setContentView设置的View
    ViewGroup decorChild = (ViewGroup) decor.getChildAt(0);
    decorChild.setBackgroundResource(background);
    // 移除掉之前父容器中的布局view
    decor.removeView(decorChild);
    // 添加设置过背景色后的子view到SwipeLayout中
    addView(decorChild);
    setContentView(decorChild);
    // 将SwipeLayout添加到DecorView中
    decor.addView(this);
  }
```

简而言之，就是在原来的ContentView和DecorView之间放入了我们的SwipeLayout。这个SwipeLayout主要处理滑动事件。