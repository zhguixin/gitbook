利用StaticLayout，将一个已经转为转换为ASCII码的图片内容，输出到该StaticLayout上，然后将该Layout输出为Bitmap。

第一步：讲一个图片转换为ASCII码的文本内容，输出到本地文件中

具体可参考网络；

第二步：将文本内容输出到StaticLayout中，将**Bitmap作为Canvas的画布**，`Canvas canvas = new Canvas(bitmap);` ，然后调用：`layout.draw(canvas)` 输出到Bitmap中。

具体代码：

```java
public void drawAsBitmap(StringBuilder text, Context context) {
    // StaticLayout布局绘制文本时用到的画笔
    TextPaint paint = new TextPaint();
    paint.setColor(Color.BLACK);
    paint.setAntiAlias(true);
    paint.setTypeface(Typeface.MONOSPACE);
    paint.setTextSize(12);
    
    // 创建StaticLayout布局
    StaticLayout layout = new StaticLayout(text, paint, screenWidth,
                                          Layout.Alignment.ALION_CENTER, 1f, 0.0f, true);
    
    // 创建Bitmap，作为Canvas的画布
    Bitmap bitmap = Bitmap.create(layout.getWidth(), layout.getHeight(), 
                                  Bitmap.Config.ARGB_8888);
    
    Canvas canvas = new Canvas(bitmap);
    canvas.translate(10, 10);
    canvas.drawColor(Color.WHITE);
    layout.draw(canvas);
}
```

> 使用Canvas的drawText绘制文本是不会自动换行的，即使一个很长很长的字符串，drawText也只显示一行，超出部分被隐藏在屏幕之外。**StaticLayout**是android中处理文字换行的一个工具类，**StaticLayout**已经实现了文本绘制换行处理。上述代码设置了换行的位置是：`screenWidth`，即字符串超过屏幕宽度时自动换行

