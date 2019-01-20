dytt_note_3

```xml
    <!-- 共用层样式 -->
    <style name="base_layout">
        <item name="android:orientation">horizontal</item>
        <item name="android:layout_width">match_parent</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:paddingTop">16dp</item>
        <item name="android:paddingBottom">16dp</item>
        <item name="android:paddingLeft">12dp</item>
        <item name="android:paddingRight">12dp</item>
        <item name="android:gravity">center_vertical</item>
        <item name="android:focusable">true</item>
        <item name="android:clickable">true</item>
    </style>

    <!-- 最外层样式 -->
    <style name="wrap_layout">
        <item name="android:orientation">vertical</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:layout_width">match_parent</item>
        <item name="android:layout_marginLeft">8dp</item>
        <item name="android:layout_marginRight">8dp</item>
        <item name="android:layout_marginTop">8dp</item>
        <item name="android:padding">1px</item>
        <item name="android:background">@drawable/background_ripple</item>
    </style>

    <!-- textview样式 -->
    <style name="usertext">
        <item name="android:textSize">16sp</item>
        <item name="android:layout_width">wrap_content</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:layout_weight">1</item>
    </style>

    <!-- 文本右边箭头样式 -->
    <style name="img_arrow">
        <item name="android:layout_width">wrap_content</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:src">@drawable/setting_arrow</item>

    </style>

    <!-- view分割线样式 -->
    <style name="bg_line">
        <item name="android:layout_marginStart">4dp</item>
        <item name="android:layout_marginEnd">4dp</item>
        <item name="android:layout_width">match_parent</item>
        <item name="android:layout_height">1px</item>
        <item name="android:background">@color/lightBlack</item>
    </style>

    <!-- 全圆角样式 -->
    <style name="single_layout" parent="base_layout">
        <item name="android:background">@drawable/background_ripple</item>
    </style>
```



```xml
    <View style="@style/bg_line"/>
    <LinearLayout style="@style/single_layout">
            <TextView style="@style/usertext" android:text="辅助功能"/>
            <ImageView style="@style/img_arrow"/>
    </LinearLayout>

    <View style="@style/bg_line"/>
    <LinearLayout style="@style/single_layout">
        <TextView style="@style/usertext" android:text="清除缓存"/>
        <TextView
            android:text="0.56M"
            android:textSize="12sp"
            android:textStyle="italic"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />
    </LinearLayout>

    <View style="@style/bg_line"/>
    <LinearLayout style="@style/single_layout">
        <TextView style="@style/usertext" android:text="软件版本"/>
        <TextView
            android:text="V1.0"
            android:textSize="12sp"
            android:textStyle="italic"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />
    </LinearLayout>

    <View style="@style/bg_line"/>
    <LinearLayout style="@style/single_layout">
        <TextView style="@style/usertext" android:text="夜间模式"/>
        <android.support.v7.widget.SwitchCompat
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </LinearLayout>

    <View style="@style/bg_line"/>
    <LinearLayout style="@style/single_layout">
        <TextView style="@style/usertext" android:text="关于作者"/>
        <ImageView style="@style/img_arrow"/>
    </LinearLayout>
```



backgroun_ripple.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="#40000000">
    <item android:drawable="@drawable/bg_selector"/>
</ripple>
```

bg_selector.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item>
        <shape android:shape="rectangle">
            <corners android:radius="8dp" />
            <solid android:color="#ffffff" />
            <padding android:left="8dp"
                android:top="0dp"
                android:right="8dp"
                android:bottom="0dp" />
        </shape>
    </item>
</selector>
```

AboutFragment.java

```java
        mHeadImg = view.findViewById(R.id.head_img);
        mHeadImg.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                shoDialog();
            }
        });

    private void shoDialog(){
        if (!mDialog.isAdded()) {
            mDialog.show(getFragmentManager(), "ImageShowDialog");
        }
    }
```

ImageDialogFragment.java

```java
    @Override
    public void onStart() {
        super.onStart();
        Window window = getDialog().getWindow();
//        window.setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN,
//                WindowManager.LayoutParams.FLAG_FULLSCREEN);
//        window.addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        WindowManager.LayoutParams lp = window.getAttributes();
        lp.gravity = Gravity.TOP;
        lp.height = WindowManager.LayoutParams.WRAP_CONTENT;
        lp.width = WindowManager.LayoutParams.MATCH_PARENT;
        window.setAttributes(lp);
      	window.setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
    }   

	@Override
    public void onPause() {
        super.onPause();
        if (getDialog() != null) {
            getDialog().dismiss();
        }
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        getDialog().requestWindowFeature(Window.FEATURE_NO_TITLE);
        return inflater.inflate(R.layout.fragment_image, container, false);
    }
```

