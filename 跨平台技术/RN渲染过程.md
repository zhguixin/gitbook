RN渲染过程

React侧，一个组件的生命周期为：componentWillMount --- render --- componentDidMount

其中`render`操作绘制DOM树

ReactNative库里提供了一些View：Text、Image

最终是在Native侧，通过UIManager创建：

```java
UIManager.createView(tag, this.viewConfig.uiViewClassName, nativeTopRootTag, updatePayload);
```

React组件更新时，调用到UIManager的updateView：

```java
UIManager.updateView(int tag, String className, ReadableMap props);
```



