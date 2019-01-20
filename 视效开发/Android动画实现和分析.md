Android动画实现和分析

Android平台支持三种动画机制：

- 逐帧动画，按帧播放，每一帧对应一张图片
- 补间动画，作用在View上，支持平移Translate、旋转rotate、缩放scale、通明度alpha，无法改变View自身的属性
- 属性动画，Android3.0（API11）之后引入的动画，功能强大，可以实现各种复杂的动画

##### 逐帧动画

在`res/drawable/`目录下定义个动画文件，名为`frame_drawable`，使用**animation-list**标签，如下所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- 设置是否只播放一次，默认为false -->
<animation-list
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true"
    >

	<!-- 指定动画图片资源；设置每一帧持续时间(ms) -->
    <item android:drawable="@drawable/a0" android:duration="100"/>
 ...
</animation-list>
```

使用的时候，借助于`AnimationDrawable`类：

```java
imageView = (ImageView) findViewById(R.id.image_view);
// 设置动画资源
imageView.setImageResource(R.drwaable.frame_drawable);
AnimationDrawable animationDrawable = (AnimationDrawable) imageView.getDrawable();
// 启动动画
animationDrawable.start();

// ...
// 停止动画
animationDrawable.stop();
```

在Java代码中，主要还是借助于`AnimationDrawable`，调用`addFrame(drawable, 100);` 设置动画资源，最后通过调用`imageView.setImageDrawable(animationDrawable);` 开始动画。

总结：逐帧动画使用简单方便，但是引入大量图片的时候，还是注意避免引起OOM。

##### 补间动画

补间动画有四种形式：alpha(淡入淡出)、translate(平移)、scale(缩放)、rotate(旋转)。经常用xml控制上述属性

举个栗子，在`res/anim/`目录下定义`fade_in.xml` ：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android" >
  
  <alpha
    android:duration="1000"
    android:fromAlpha="0.0"
    android:interpolator="@android:anim/accelerate_decelerate_interpolator"
    android:toAlpha="1.0" />
  
</set>
```

在java文件中给一个Button设置该动画：

```java
Animation animation = AnimationUtils.loadAnimation(mContext, R.anim.fade_in);
btn = (Button) findViewById(R.id.btn);
btn.startAnimation(animation);
```

其他一些动画效果，比如热点信息的轮播效果，借助于TextViewSwitcher；

设置Activity的切换动画：`overridePendingTransition((R.anim.enter_anim,R.anim.exit_anim);)`。记得要在`startActivity`或者`finish()`之后调用。

window的切换动画，比如应用在PopWindow、Dialog中，在`styles.xml`中定义主题样式：

```xml
<style name="popwindow_anim_style">
    <!-- 指定显示的动画xml -->
  <item name="android:windowEnterAnimation">@anim/popup_in</item>

    <!-- 指定消失的动画xml -->
  <item name="android:windowExitAnimation">@anim/popup_out</item>
</style>
```

使用的时候：

```java
mPopWindow.setAnimationStyle(R.style.popwindow_anim_style);
```



##### 属性动画

属性动画的作用原理，先改变值，然后 赋值 给对象的属性从而实现动画效果。

看一下，在在`res/animator/`目录下定义一个属性动画`increase_anim.xml` ：

```xml
<!-- ValueAnimator采用<animator>  标签 -->
<animator xmlns:android="http://schemas.android.com/apk/res/android"  
    android:valueFrom="0"
    android:valueTo="100"
    android:valueType="intType"
    android:duration="3000"
/>
```

在java代码中使用：

```java
// 载入XML动画
Animator animator = AnimatorInflater.loadAnimator(context, R.animator.increase_anim);  

// 设置动画对象
animator.setTarget(view);  

// 启动动画
animator.start();  
```

> 实际开发中，建议使用Java代码实现属性动画：因为很多时候属性的起始值是无法提前确定的（无法使用XML设置），这就需要在`Java`代码里动态获取。

属性动画有两个重要的类：`ValueAnimator`和`ObjectAnimator` 。其中ObjectAnimator是ValueAnimator的子类，使用这个类可以对任意对象的任意属性进行动画操作。

属性动画的运行机制是通过不断改变对象的属性值实现动画，开始值和结束值之间的动画过渡通过ValueAnimator来计算，所以我们只需提供开始值、结束值以及运行时间即可。除此之外，ValueAnimator还负责管理动画的播放次数、播放模式、以及设置动画监听器。

ValueAnimator 到底是怎样实现从初始值平滑过渡到结束值的呢？这个就是由TypeEvaluator 和TimeInterpolator 共同决定的。

- 估值器（TypeEvaluator），时间间隔内具体的变化数值，**控制动画的运动轨迹**
- 插值器（Interpolator），决定了值在时间间隔内的变化速率（匀速、加速。。），**控制动画的快慢节奏**


**ValueAnimator**

```java
ValueAnimator animator = ValueAnimator.ofInt(0,3);
animator.setDuration(5000);  
animator.start();
```

自定义估值器的使用：

```java
// 创建初始动画时的对象  & 结束动画时的对象
myObject object1 = new myObject();  
myObject object2 = new myObject();  

ValueAnimator anim = ValueAnimator.ofObject(new myObjectEvaluator(), object1, object2);  
// 创建动画对象 & 设置参数
// 参数说明
// 参数1：自定义的估值器对象（TypeEvaluator 类型参数）
// 参数2：初始动画的对象
// 参数3：结束动画的对象
anim.setDuration(5000);  
anim.start();

// myObjectEvaluator 继承TypeEvaluator，并实现该类的唯一方法 evaluate
public class myObjectEvaluator implements TypeEvaluator {

    // 参数1：当前动画完成的百分比
    // 参数2/3：动画的开始值/结束值
    @Override
    public Object evaluate(float fraction, Object startValue, Object endValue) {
        Point startPoint = (Point) startValue;
        Point endPoint = (Point) endValue;
        float x = startPoint.getX() + fraction * (endPoint.getX() - startPoint.getX());

        float y = (float) (Math.sin(x * Math.PI / 180) * 100) + endPoint.getY() / 2;
        Point point = new Point(x, y);
        return point;
    }
}
```

通过自定义估值器，我们可以自己决定在object1 和 object2 之间object值得变化情况。

**ObjectAnimator**

ObjectAnimator继承自ValueAnimator，Objective可以选择属性进行操作。

```java
ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(mButton, "alpha", 0f,3f);
objectAnimator.setDuration(500);
objectAnimator.start();  
```

如果需要采用`ObjectAnimator` 类实现动画效果，那么需要操作的对象就必须有该属性的`set()` 和 `get()`方法。

如果想要实现组合动画的功能，可以借助于`AnimatorSet`类：

```java
AnimatorSet.play(Animator anim)   // 播放当前动画
AnimatorSet.after(long delay)   // 将现有动画延迟x毫秒后执行
AnimatorSet.with(Animator anim)   // 将现有动画和传入的动画同时执行
AnimatorSet.after(Animator anim)   // 将现有动画插入到传入的动画之后执行
AnimatorSet.before(Animator anim)   // 将现有动画插入到传入的动画之前执行
```



#### 总结

补间动画的平移操作并没有实现View的真正移动，点击原来的位置还能响应；

补间动画用xml书写，复用效率高，Activity间的切换动画、窗口弹出或隐藏动画常用补间动画实现；

属性动画切记在退出时，及时停止，防止内存泄漏；




详细参考：[Android动画总结](http://www.jianshu.com/p/420629118c10) 、[Android补间动画使用总结](http://www.jianshu.com/p/733532041f46)、[Android属性动画详细攻略](http://www.jianshu.com/p/2412d00a0ce4)