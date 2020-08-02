Android View的测量过程

DecorView的MeasureSpec由Window的大小和自身的LayoutParams决定

普通View的MeasureSpec由父View的MeasureSpec和View自身的LayoutParams决定

在ViewGroup中，getChildMeasureSpec具体实现了普通View的MeasureSpec。

如果View在LayoutParams指定了精确大小，无论父View的SpecMode是什么，View的大小都为这个精确值。

其他情况（也就是View的LayoutParams指定WRAP_CONTENT和MATCH_PARENT），子View的大小就是父View剩下的空间。（这里把父View的SpecMode为UNSPECIFIED排除掉了）

该剩下的空间大小为：

```java
// ViewGroup中的measureChildWithMargins方法
// 宽: 父View测量得到的大小减去自身的padding和子View的margin
specSize - (mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin);
// 高:
specSize - (mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin);
```

ViewGroup中的measureChildWithMargins方法中调用子View的measure方法，并把测量模式和测量值(即MeasureSpec)传递到了子View，这样一来子View在覆写onMeasure方法时就知道了测量模式和测量大小。

子View测量完毕后（onMeasure方法的最后）加上：

```java
setMeasureDimension(measuredWidth, measuredHeight);
```

> 只有父View的测量模式是UNSPECIFIED，并且子View指定match_parent或者wrap_content时，子View的大小才取到minWidth这个值

父View可以在测量自身时调用：

```java
child.getMeasuredHeight();
```

得到子View的测量宽高来进一步确定自身的大小