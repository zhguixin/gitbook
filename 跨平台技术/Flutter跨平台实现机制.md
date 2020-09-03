Flutter不同于React Native跨平台框架，Flutter自己实现了图形库的绘制。

参考了如下几篇文章：

[ Flutter原理与美团的实践](https://blog.csdn.net/MeituanTech/article/details/81567238)

[深入理解flutter的编译原理与优化](https://yq.aliyun.com/articles/604052?utm_content=m_1000004305)

[Flutter框架研究和与RN对比](http://szuwest.github.io/flutterkuang-jia-yan-jiu-he-yu-rndui-bi.html)

[Flutter 原理简解](https://juejin.im/entry/5afa9769518825428630a61c)

### Flutter渲染机制



### Flutter混合栈管理

混合栈指的是，页面栈里即含有Flutter页面又含有Native页面。每个页面由一个FlutterView维护，当交替出现Native页面和Flutter页面时，会出现内存过大的情况。

> FlutterView内部会初始化一个Engine，维护与Native同信的PlatformChannel，管理DartVM 处理垃圾回收，处理图形渲染

为了避免内存过大的情况，可以复用FlutterView，或者复用FlutterNativeView。

> FlutterNativeView是FlutterView的内部成员变量，所以复用FlutterNativeView是更轻量级一些的
>
> 关于FlutterView参考文章：https://www.ccarea.cn/archives/575

具体实现参考文章：https://www.jianshu.com/p/6c7c66111bc8

https://juejin.im/post/6844903763627474957

