APP开发记录笔记二

在使用Glide加载图片时，如果要加载的图片宽度很宽，在显示图片时，为了确保比例更加的协调，需要根据图片宽高比，自动确定高度：

ImageView的布局如下（图片要显示的宽度为整个屏幕的宽度）：

```xml
<ImageView
           android:id="@+id/previewImage"
           android:layout_width="match_parent"
           android:layout_height="wrap_content" />
```

使用glide加载时，调用如下代码：

```java
  Glide.with(this)
    .load(detailEntity.getPreviewImage())
    .asBitmap()
    .into(new BitmapImageViewTarget(previewImage) {
      @Override
      protected void setResource(Bitmap resource) {
        super.setResource(resource);
        int width = resource.getWidth();// 获得图片原始宽度
        int height = resource.getHeight(); // 获得图片原始高度
        float ratio = width * 1.0F / height; // 获得原始图片的宽高比
        float targetHeight = Utils.getScreenWidth() * 1.0F / ratio;// 计算出目标宽度

        ViewGroup.LayoutParams params = previewImage.getLayoutParams();
        params.height = (int) targetHeight;
        previewImage.setLayoutParams(params);

        previewImage.setImageBitmap(resource);
      }
    });
```

根据手机分辨率获得屏幕宽高的工具函数：

```java
    private static int mScreenWidth = 0;
    private static int mScreenHeight = 0;

	public static Point getScreenWidthAndHeight(Context cx) {
        Point ret = new Point();

        if (mScreenWidth == 0 && mScreenHeight == 0) {
            WindowManager wm = (WindowManager) cx.getSystemService(Context.WINDOW_SERVICE);
            DisplayMetrics dm = cx.getResources().getDisplayMetrics();
            wm.getDefaultDisplay().getRealMetrics(dm);
            mScreenWidth = dm.widthPixels;
            mScreenHeight = dm.heightPixels/* + getNavigationBarHeight(cx) */;
        }

        ret.x = mScreenWidth;
        ret.y = mScreenHeight;

        return ret;
    }
```

判断手机网络连接状态：

```java
    /**
     * 当前网络是否已连接
     *
     * @param cx 上下文
     * @return {@link NET_TYPE}
     */
    public static boolean netIsConnected(Context cx) {
        boolean ret = false;
        ConnectivityManager connectMgr = (ConnectivityManager) cx.getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo info = connectMgr.getActiveNetworkInfo();
        if (info != null) {
            if (info.getState() == NetworkInfo.State.CONNECTED) {
                ret = true;
            }
        }
        return ret;
    }
```

下拉刷新的实现

Android在support-v4包中提供了一个**SwipeRefreshLayout**来提供下拉刷新的功能，使用如下：

将`RecyclerView`或者`ListView`放在SwipeRefreshLayout中

```xml
    <android.support.v4.widget.SwipeRefreshLayout
        android:id="@+id/refres_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/new_movie_list"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </android.support.v4.widget.SwipeRefreshLayout>
```

代码实现下拉刷新的功能，通过设置`setOnRefreshListener`的回调，并实现`onRefresh`回调来实现下拉刷新的功能，很easy，不是吗。

```java
    mRefreshLayout = findViewById(R.id.refres_layout);
    mRefreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
      @Override
      public void onRefresh() {
        MovieBean movie = new MovieBean("http://whatever.html","2017-10-31",
                                        "动作大片", "需求：我有个想法");
        Log.d(TAG, "onRefresh: movie" + movie.toString());
        mMovies.add(movie);
        mAdapter.notifyDataSetChanged();
        mRefreshLayout.setRefreshing(false);
      }
     });
```

正则表达式：

```java
String regEx_html = "<[^>]+>"; // 定义HTML标签的正则表达式
Pattern p_html = Pattern.compile(regEx_html);
Matcher m_html = p_html.matcher(str);
str = m_html.replaceAll(""); // 过滤html标签
```

ActionBar中配置搜索框，并且设置为默认展开的状态：

```java
	@Override  
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {  
        inflater.inflate(R.menu.contact, menu);  
        MenuItem search = menu.findItem(R.id.search);  
        search.collapseActionView();  
        //是搜索框默认展开  
        search.expandActionView();  
        super.onCreateOptionsMenu(menu, inflater);  
    }
```

图片模糊效果的实现：

```java
    public static Bitmap blur(Context context, Bitmap originalBitmap){

        // 计算图片缩小后的长宽
        int width = Math.round(originalBitmap.getWidth() * BITMAP_SCALE);
        int height = Math.round(originalBitmap.getHeight() * BITMAP_SCALE);
        // 将缩小后的图片做为预渲染的图片。
        Bitmap inputBitmap = Bitmap.createScaledBitmap(originalBitmap, width, height, false);
        // 创建一张渲染后的输出图片。
        Bitmap outputBitmap = Bitmap.createBitmap(inputBitmap);
        // 创建RenderScript内核对象
        RenderScript rs = RenderScript.create(context);
        // 创建一个模糊效果的RenderScript的工具对象
        ScriptIntrinsicBlur blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
        // 由于RenderScript并没有使用VM来分配内存,所以需要使用Allocation类来创建和分配内存空间。
        // 创建Allocation对象的时候其实内存是空的,需要使用copyTo()将数据填充进去。
        Allocation tmpIn = Allocation.createFromBitmap(rs, inputBitmap);
        Allocation tmpOut = Allocation.createFromBitmap(rs, outputBitmap);
        // 设置渲染的模糊程度, 25f是最大模糊度
        blurScript.setRadius(BLUR_RADIUS);
        // 设置blurScript对象的输入内存
        blurScript.setInput(tmpIn);
        // 将输出数据保存到输出内存中
        blurScript.forEach(tmpOut);
        // 将数据填充到Allocation中
        tmpOut.copyTo(outputBitmap);
        return outputBitmap;
    }
```

ActionMode的使用：

```java
getActivity().startActionMode(mCallback);

ActionMode.Callback mCallback = new ActionMode.Callback() {
  @Override
  public boolean onCreateActionMode(ActionMode mode, Menu menu) {
  }
};
```

ActionMode的样式修改：

```xml
  <ImageView android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             android:layout_gravity="center"
             android:scaleType="fitCenter"
             android:src="?android:attr/actionModeCloseDrawable" />
```

`android:src="?android:attr/actionModeCloseDrawable"`其中**？**的意思表示：`android:src`引用值为当前主题下的`actionModeCloseDrawable`的值。在定义样式的时候，可以自定义你的返回箭头：

```xml
    <style name="custom_theme" parent="@android:style/Theme.Material.Light">
      <item name="android:colorAccent">@color/action_bar_icon_color</item>
      <item name="android:listDivider">@drawable/list_divider_material</item>
      // 自定义返回箭头
      <item name="android:actionModeCloseDrawable">@drawable/arrow_back</item>
    </style>
```







