## Android Fragment使用

### Fragment和Activity的相互通信

为了方便Fragment和Activity(需要继承`FragmentActivity`)之间进行通信，`FragmentManager`提供了一个类似于`findViewById()`的方法，专门用于从布局文件中获取Fragment的实例：

```java
  FragmentManager fm = getFragmentManager();
  Fragment fragment = fm.findFragmentById(R.id.fragment);
```

其中，`fragment_container`是一个FrameLayout,包含一个Fragment：

```xml
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <fragment
        android:id="@+id/fragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</FrameLayout>
```

通过`FragmentManager`的`beginTransaction()`方法获取一个`FragmentTransaction`的实例，开启一个事务来对Fragment进行`add`、`replace`、`remove`等操作：

```java
  if (fragment == null) {
    fragment = createFragment();
    fm.beginTransaction()
      .replace(R.id.fragment_container, fragment)
      .addToBackStack(null) // 避免按下 Back 键程序就会直接退出
      .commit();
  }
```

在每个Fragment中都可以通过调用 getActivity()方法来得到和当前Fragment相关联的Activity的实例：

```java
MainActivity activity = (MainActivity) getActivity();
```

*Fragment之间的通信，可以先获得FragmentA对应的Activity实例，在通过这个Activity实例得到FragmentB，从而间接实现FragmentA和FragmentB之间的相互通信*

### Fragment的生命周期

- onAttach()

  当Fragment和宿主Activity建立关联的时候调用

- onCreate()

  完成Fragment的初始化创建

- onCreateView()

  创建View返回给Fragment时调用。

- onActivityCreated()

  通知Fragment当前Activity的`onCreate` 方法已经调用完成

- onStart：Fragment已经对用户可见时调用，当然这个基于它的宿主Activity的onStart()方法已经被调用
- onResume：Fragment已经开始和用户交互时调用，当然这个基于它的宿主Activity的onResume()方法已经被调用
- onPause：Fragment不再和用户交互时调用，这通常发生在宿主Activity的onPause()方法被调用或者Fragment被修改（replace、remove）
- onStop：Fragment不再对用户可见时调用，这通常发生在宿主Activity的onStop()方法被调用或者Fragment被修改（replace、remove）

- onDestroyView()

  Fragment释放View资源时被调用

- onDetach()

  Fragment与宿主Activity解除关联的时候调用 

### Fragment高级使用

我们可以通过**ViewPager + Fragment**实现在一个Activity中，切换不同的Fragment。

Activity使用的布局如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical">
  
    <android.support.design.widget.TabLayout
        android:id="@+id/tabLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:tabMode="scrollable"
        />
    <android.support.v4.view.ViewPager
        android:id="@+id/viewPager"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:background="#ffffff"
        />
</LinearLayout>
```

其中`TabLayout`是support design库中的一个控件，此处用来指示顶部标签。

Activity的主体代码如下：

```java
    //Fragment+ViewPager+FragmentViewPager组合的使用
    ViewPager viewPager = (ViewPager) findViewById(R.id.viewPager);
    MyFragmentPagerAdapter adapter = new MyFragmentPagerAdapter(getSupportFragmentManager(),
                                                                this);
    viewPager.setAdapter(adapter);

    //TabLayout
    TabLayout tabLayout = (TabLayout) findViewById(R.id.tabLayout);
    tabLayout.setupWithViewPager(viewPager);
```

其中`MyFragmentPagerAdapter`继承自FragmentPagerAdapter：

```java
  class MyFragmentPagerAdapter extends FragmentPagerAdapter {
      private final int COUNT = 5;
      private String[] titles = new String[]{"最新电影", "国内经典", "日韩电影", "欧美电影"};
      private String[] urls = new String[]{"index1.html", "index2.html"}
      private Context mContext;

      public MyFragmentPagerAdapter(FragmentManager fm, Context context) {
          super(fm);
          mContext = context;
      }

      @Override
      public Fragment getItem(int position) {
          return PageFragment.newInstance(urls[position]);
      }

      @Override
      public int getCount() {
          return COUNT;
      }

      @Override
      public CharSequence getPageTitle(int position) {
          return titles[position];
      }
  }
```

其中，`PageFragment`定义如下：

```java
  public class PageFragment extends Fragment {
      public static final String URL = "url";
      private int mUrl;

      public static PageFragment newInstance(String url) {
          Bundle args = new Bundle();

          args.putString(URL, url);
          PageFragment fragment = new PageFragment();
          fragment.setArguments(args);
          return fragment;
      }

      @Override
      public void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          mUrl = getArguments().getString(URL);
      }

      @Override
      public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
          View view = inflater.inflate(R.layout.fragment_container,container,false);
          TextView textView = (TextView) view.findViewById(R.id.textView);
          textView.setText(mUrl);
          return view;
      }
  }
```

> 在这里利用了`Bundle`和这个`setArguments(bundle)`方法，向Fragment传递参数，达到保存参数的目的。因为Fragment是通过反射调用它的无参构造函数实例化的，直接通过构造函数向Fragment传递参数，在Fragment重建时，会造成参数丢失。

PageFragment使用的布局`fragment_container`的定义如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:orientation="vertical">

    <TextView
              android:id="@+id/textView"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:gravity="center" />

</RelativeLayout>
```

至此，结合**TabLayout+ViewPager+Fragment**标签页切换的功能就实现了。

#### 定制TabLayout

关于定制`TabLayout`：

1、更改Tablayout中的标签为自定义View：

```java
	// 在Activity中：
    for (int i = 0; i < mTabLayout.getTabCount(); i++) {
        mTabLayout.getTabAt(i).setCustomView(myCustomView);
    }
```

2、根据标签是否选中更新自定义View的状态：

```java
	//重写Tablayout 的 setSelected()方法
    @Override
    public void setSelected(boolean selected) {
        super.setSelected(selected);
        // 更改文本颜色、图标、背景等等
    }
```

### Fragment懒加载

Fragment实现懒加载的核心是，**Fragment可见时加载**。在使用ViewPager时，由于ViewPager的缓存策略，会提前加载当前显示页的左右两边页。如果Fragment在初始化时，有网络下载等操作，会造成网络拥堵。

主要借助于方法`setUserVisibleHint(boolean isVisibleToUser)` 当`isVisibleToUser`为True时，Fragment可见；反之不可见。

> 这里要注意，Fragment的 `setUserVisibleHint`回调发生在`onViewCreated`之前。因此在`setUserVisibleHint`方法中不能根据可见性来更新UI，只可以在此方法中做些耗时不更新UI的操作。

封装一个`LazyFragment`来实现懒加载，提供抽象回调方法：`onFragmentVisibleChange`、`onFragmentFirstVisible` 供子类实现。

```java
public abstract class LazyFragment extends Fragment {

    private boolean isFragmentVisible;
    private boolean isFirstVisible;
    private View rootView;

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        // rootView在onViewCreated中被赋值，避免在onViewCreated之前做操作
        if (rootView == null) {
            return;
        }
        
        if (isFirstVisible && isVisibleToUser) {
            onFragmentFirstVisible();
            isFirstVisible = false;
        }
        
        if (isVisibleToUser) {
            onFragmentVisibleChange(true);
            isFragmentVisible = true;
        } else {
            if (isFragmentVisible) {
                isFragmentVisible = false;
                onFragmentVisibleChange(false);
            }
        }

    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        init();
    }

    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        //如果setUserVisibleHint()在rootView创建前调用时，那么
        //就等到rootView创建完后才回调onFragmentVisibleChange(true)
        //保证onFragmentVisibleChange()的回调发生在rootView创建完成之后，以便支持ui操作
        if (rootView == null) {
            rootView = view;
            if (getUserVisibleHint()) {
                if (isFirstVisible) {
                    onFragmentFirstVisible();
                    isFirstVisible = false;
                }
                onFragmentVisibleChange(true);
                isFragmentVisible = true;
            }
        }
        super.onViewCreated(view, savedInstanceState);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        init();
    }

    private void init() {
        isFirstVisible = true;
        isFragmentVisible = false;
    }

    /**
     * @param isVisible true  不可见 -> 可见
     *                  false 可见  -> 不可见
     * 在该方法内可根据Fragment可见性，更新UI
     */
    abstract protected void onFragmentVisibleChange(boolean isVisible) {

    }

    /**
     * 在Fragment首次可见时回调，可在这里进行进行网络下载，保证只在第一次打开Fragment时才会加载数据
     * 该方法会在Fragment实例存在时，只被调用一次，后面只回调 onFragmentVisibleChange()
     */
    abstract protected void onFragmentFirstVisible() {

    }

    protected boolean isFragmentVisible() {
        return isFragmentVisible;
    }
}
```

