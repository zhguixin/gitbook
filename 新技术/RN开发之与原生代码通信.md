RN开发之与原生代码通信

#### Native传递信息到React端

在React端进行编码时，会通过props将父组件的信息传递到子组件。

在native端（考虑Android平台）启动的ReactActivity时有一个getLaunchOptions()方法，该方法返回值，就会传递到React端，作为整个父组件的属性。

如果不想要整个页面都通过React Native，则可以通过Native端的自定义View——ReactRootView来限制使用React开发的区域。在ReactRootView中同样提供了，向React父组件传递属性的方法：

```java
Bundle updateProps = mReactRootView.getAppProperties();
updateProps.putStringArrayList("image","http://test/1.jpg");
mReactRootView.setAppProperties(updateProps);
```

在React端就可以获得该属性：

```react
this.props.image
```

#### React端传递信息到Native端



以上简单信息的互相传输有时并不能满足一些开发需求，比如在React端复用在Native端已经开发好的一些模块，就需要在React端去调用这些已用的方法；又比如在Native端移除掉ReactRootView时，给React端一个通知去做一些处理等等。这样又有两种情况：React端调用Native端的方法；Native端调用React端的方法。

#### React端调用Native端的方法

以官方给出的一个示例研究：通过js代码调用Android平台的Toast模块。