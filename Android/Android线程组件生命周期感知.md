Android线程组件生命周期感知

2017/12/21

Android中开启的线程生命周期直到`run`函数体执行完成，或者`run`函数有异常抛出。如果在Activity中开启了后台线程，执行完成更新UI界面，但此时手机配置发生改变导致原来的Activity退出了（比如旋转了手机屏幕），但是线程还在后台执行。执行完成后更新UI界面时，Activity已经退出了，工作线程的执行结果就被丢弃了。所以很有必要对线程进行保留，对组件的生命周期进行感知。

使用场景：从本地图库添加图片到【本地添加】列表中，用户一次性添加30张图片。

我们可以把一个线程保存在一个没有UI界面的Fragment中，在onCreate方法中调用`setRetainInstance(true)`。即使配置发生变化导致Activity被销毁时，在Activity中被表示保持的Fragment也不会被销毁。我们可以在该Fragment中保存大量数据。

在Activity中创建持有工作线程的Fragment：

```java
public class ThreadRetainWithFragmentActivity extends Activity {
    private ThreadFragment mThreadFragment;
    private TextView mTextView;

    public void onCreate(Bundle savedInstanceState) {
        setContentView(R.layout.activity_retain_thread);
        mTextView = (TextView) findViewById(R.id.text_retain);
        FragmentManager manager = getFragmentManager();
        mThreadFragment = (ThreadFragment) 
          manager.findFragmentByTag("threadfragment");
        if (mThreadFragment == null) {
            FragmentTransaction transaction = manager.beginTransaction();
            mThreadFragment = new ThreadFragment();
            transaction.add(mThreadFragment, "threadfragment");
            transaction.commit();
        }
    }

    // 需要的地方开启线程，真正的执行在Fragment中
    public void onStartThread(View v) {
        mThreadFragment.execute();
    }

    public void setText(final String text) {
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                mTextView.setText(text);
            }
        });
    }
}
```

线程放在Fragment中，ThreadFragment持有Activity的引用，以便更新UI界面：

```java
public class ThreadFragment extends Fragment {
    private ThreadRetainWithFragmentActivity mActivity;
    private MyThread t;

    private class MyThread extends Thread {
        @Override
        public void run() {
            final String text = getTextFromNetwork();
            mActivity.setText(text);
        }

        // Long operation
        private String getTextFromNetwork() { // Simulate network operation 
          SystemClock.sleep(5000);
          return "Text from network";
        }
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setRetainInstance(true);
    }

    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        mActivity = (ThreadRetainWithFragmentActivity) activity;
    }

    @Override
    public void onDetach() {
        super.onDetach();
        mActivity = null;
    }

    public void execute() {
        t = new MyThread();
        t.start();
    }
}
```

