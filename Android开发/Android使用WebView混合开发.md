Android Native与WebView通信

##### 基本使用

之前与inmobi合作的加载视频的方式采用的就是JS注入。

Android中对WebView的基本设置：

```java
WebSettings settings = mWebView.getSettings();
    //可以执行JS
    settings.setJavaScriptEnabled(true);
    //使能缓存
    settings.setAppCacheEnabled(true);
    appInterface = new AppInterface(this);
    //添加JS接口
    mWebView.addJavascriptInterface(new JSInterface(), "jslauncher");
    //添加监听
    mWebView.setWebViewClient(new MyWebViewClient());
	// 目标网站
	mWebView.loadUrl(mUrl);
```

> tips，WebViewClient与WebChromeClient的区别：
>
> - WebViewClient主要帮助WebView处理各种通知、请求事件的
> - WebChromeClient主要辅助WebView处理Javascript的对话框、网站图标、网站title、加载进度等

#### JavaScript调用Java代码

设置WebView，调用`addJavascriptInterface` 方法：

`mWebView.addJavascriptInterface(new JSInterface(), "jslauncher");` 

我们通过向WebView注册一个名叫`JSInterface`的对象，然后在JS中可以访问到`jslauncher`这个对象：

```javascript
if (window.jslauncher != null) {
    window.jslauncher.openImage("./test.png")
}
```

如上，就可以调用这个对象的一些方法，最终可以调用到Java代码中，从而实现了JS与Java代码的交互。

JS通信接口，JSInterface定义如下：

```java
// js通信接口  
public class JSInterface {  

  private Context mContext;  

  public JSInterface(Context context) {  
    mContext = context;
  }  

  // 由JS脚本调用，函数入参由JS脚本传入
  @JavascriptInterface
  public void openImage(String img) {  
    System.out.println(img);  
    Intent intent = new Intent();  
    intent.putExtra("image", img);  
    intent.setClass(context, ShowWebImageActivity.class);  
    context.startActivity(intent);  
    System.out.println(img);  
  }  
}  
```

4.2及以后需要加上注释语句`@JavascriptInterface` ，这个接口可以避免注入恶意JS脚本通过反射执行一些危险方法，而出现的JS注入漏洞。

#### Java调用JavaScript代码

通过WebView调用`loadUrl` 方法，传入一个JavaScript代码的字符串，其中字符串以 **javascript:**  开头，如下代码：

```java
// 这段js函数的功能就是，遍历所有的img结点，并添加onclick函数;
// onclick函数的功能是在图片点击的时候调用本地java接口并传递url过去  
mWebView.loadUrl("javascript:(function(){" +  
                       "var objs = document.getElementsByTagName(\"img\"); " +   
                       "for(var i=0;i<objs.length;i++)  " +   
                       "{"  
                       + "    objs[i].onclick=function()  " +   
                       "    {  "   
                       + "        window.jslauncher.openImage(this.src);  " +   
                       "    }  " +   
                       "}" +   
                       "})()");  
}
```



#### 总结

JS脚本中通过调用`window.jslauncher.openImage(this.src);`调用JSInterface中定义的方法。

这种Android原生代码与JavaScript混合编程，就是市面上普遍采用的HybridApp开发机制，在进行Hybrid开发时，有几点需要注意：

1、Android在生产js脚本时，加载这段JS脚本的时机；(可以放在MyWebViewClient的`onPageFinished`方法中)

2、兼容4.2以下的版本需要考虑JS通过反射调用的注入漏洞；

3、WebView的内存控制；

#####使用场景：

通过JS注入设置网页的夜间模式：

```java
/*--------------------WebView 夜间模式-----------------*/
public static void setupWebView(WebView webView, String backgroudColor, String fontColor, String urlColor) {
  if (webView != null) {
    webView.setBackgroundColor(0);
    if (getCurrentSkinType(MyApplication.getIntstance()) == THEME_NIGHT) {
      // String.format,格式化jsStyle“字符串”
      String js = String.format(jsStyle, backgroudColor, fontColor, urlColor, backgroudColor);
      webView.loadUrl(js);
    }
  }
}

private static String jsStyle = "javascript:(function(){\n" +
  "\t\t   document.body.style.backgroundColor=\"%s\";\n" +
  "\t\t    document.body.style.color=\"%s\";\n" +
  "\t\t\tvar as = document.getElementsByTagName(\"a\");\n" +
  "\t\tfor(var i=0;i<as.length;i++){\n" +
  "\t\t\tas[i].style.color = \"%s\";\n" +
  "\t\t}\n" +
  "\t\tvar divs = document.getElementsByTagName(\"div\");\n" +
  "\t\tfor(var i=0;i<divs.length;i++){\n" +
  "\t\t\tdivs[i].style.backgroundColor = \"%s\";\n" +
  "\t\t}\n" +
  "\t\t})()";
```

