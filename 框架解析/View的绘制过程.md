View的绘制过程

View的绘制入口在ViewRootImpl的`performTraversals` 方法中，调用链：

```java
performDraw() -->> draw(isFullRedrawNeeded) -->> drawSoftWare(Surface,...)
--> View.draw(Canvas)
-->> ViewGroup.dispatchDraw(Canvas)
-->> ViewGroup.drawChild(Canvas, childView, drawingTime)
-->> View.draw(Canvas canvas, ViewGroup parent, long drawingTime)
```

进入到View类中的draw方法中，按照：绘制背景-->绘制自己（调用OnDraw）-->绘制子View --> 绘制装饰物。

View中的`dispatchDraw` 为空方法，只有ViewGroup有此方法，在这个方法中，ViewGroup会进一步的将绘制任务交给自己的子View绘制，调用draw三个入参的重载方法，涉及到坐标系变换和硬件加速。

#### 认识Surface

Surface在ViewRootImpl中被初始化：

```java
Surface mSurface = new Surface();
```

在`performTraversals` 跨进程调用到WMS的`relayoutWindow` 方法，跨进程调用时，会将**mSurface** 传递进去。

在WMS中，再次调用`WindowSurfaceController` 的`getSurface` 方法：

```java
surfaceController.getSurface(outSurface);

// WindowSurfaceController.java
void getSurface(Surface outSurface) {
    outSurface.copyFrom(mSurfaceControl);
}

// Surface.java
public void copyFrom(SurfaceControl other) {
    long surfaceControlPtr = other.mNativeObject;
    long newNativeObject = nativeCreateFromSurfaceControl(surfaceControlPtr);
}
```

#### View绘制数据到SurfaceFlinger进程的进行混合

App进程与SurfaceFlinger进程通过匿名共享内存的方式实现进程间通信。