实现如下功能：锁屏上有个链接，点击之后打开一个可以在锁屏界面上显示的Activity（**AppDetailActivity**）。在新打开的**AppDetailActivity**中，点击任何按钮直接解锁，如果用户设置了安全锁屏则调起安全解锁界面。

实现**AppDetailActivity**，该Activity可在锁屏界面上显示：

```java
public class AppDetailActivity extends AppCompatActivity {

    // 监听灭屏广播，灭屏后finish掉自己
    private BroadcastReceiver receiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            AppDetailActivity.this.finish();
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 添加标记为使得该Activity可以在锁屏界面上显示
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                | WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON);
        setContentView(R.layout.activity_detail);

        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_SCREEN_OFF);
        registerReceiver(receiver, intentFilter);
    }

    public void btnOnClick(View view) {
        // 点击一个按钮，触发解锁操作；
        // 注意：Android O之后，需要显示调用一次requestDismissKeyguard()方法
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            KeyguardManager keyguardManager = (KeyguardManager) getSystemService(Context.KEYGUARD_SERVICE);
            keyguardManager.requestDismissKeyguard(this, null);
        }
        // 此处启动的是一个【一像素的Activity】，主要是为了触发解锁操作
        Intent intent = new Intent(this, OnePixelActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        try {
            unregisterReceiver(receiver);
            receiver = null;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

一像素Activity的实现如下：

```java
/**
 * AndroidManifest配置
 * <activity android:name=".OnePixelActivity"
            android:theme="@android:style/Theme.Translucent" />
 */
public class OnePixelActivity extends Activity {

    // 监听灭屏广播，灭屏后销毁自己
    private BroadcastReceiver receiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            OnePixelActivity.this.finish();
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getWindow().addFlags(WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);

        // 窗口位于左上角，宽、高各为1像素
        Window window = getWindow();
        window.setGravity(Gravity.LEFT | Gravity.TOP);
        WindowManager.LayoutParams params = window.getAttributes();
        params.x = 0;
        params.y = 0;
        params.height = 1;
        params.width = 1;
        window.setAttributes(params);

        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_SCREEN_OFF);
        intentFilter.addAction(Intent.ACTION_USER_PRESENT);
        registerReceiver(receiver, intentFilter);
    }

    @Override
    protected void onResume() {
        super.onResume();
        boolean isLocked = isKeyguardLocked(this);
        Log.d("zgx", "onResume: isLocked=" + isLocked);
        // 非锁屏状态下，直接finish掉自己，没必要显示
        if (!isLocked) {
            finish();
        }
    }

    @Override
    public void onBackPressed() {
        super.onBackPressed();
        finish();
    }

    @Override
    public void onDestroy(){
        try {
            unregisterReceiver(receiver);
            receiver = null;
        } catch (Exception e) {
            e.printStackTrace();
        }
        this.getWindow().clearFlags(WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);
        super.onDestroy();
    }

    private boolean isKeyguardLocked(Context context) {
        KeyguardManager keyguardManager = (KeyguardManager) context.getSystemService(Context.KEYGUARD_SERVICE);
        return keyguardManager.isKeyguardLocked();
    }
}
```

