view的滑动原理

在一个自定义View中，绘制一个圆。如果我想要让圆进行向上移动，给出两种方案。

方案一：

```java
mCustomView.scrollBy(0,10);//每次向上移动10个像素
```

方案二，在自定义的`CustomView`的onDraw方法中：

```java
  public void setDy(int dy) {
    mDy += 10;
    invalidate();
  }

@Override
protected void onDraw(Canvas canvas) {
    canvas.save();
    canvas.translate(0,-mDy);
    canvas.drawCircle(50,50,40.0f,mPaint);
    canvas.restore();
}
```

在外部调用：

```java
mCustomView.setDy(0,10);
```

两种方案都可以移动，其实scrollBy或者是scrollTo的实现原理就是设置canvas的translate：

```java
canvas.translate(-scrollX, -scrollY);
```

