JNI（Java Native Interface，Java本地接口），通过JNI技术打通了Java层与Native层(C/C++)，使得Java和Native可以相互调用。

Android中在Zygote进程创建Android虚拟机后，会完成对虚拟机JNI方法的注册。

**jni存在的常见目录：**

- `/framework/base/core/jni/`
- `/framework/base/services/core/jni/`

如果是自定义的JNI方法，需要借助于``System.loadLibrary()`方法，例如：MediaPlayer.java中调用`System.loadLibrary("media_jni")`把libmedia_jni.so动态库加载到内存。



### NDK编程

在Android开发中，提供了NDK了实现JNI编程。

借助于`javah` 命令生成头文件，在Android工程目录下的`./src/main/java` 目录下，运行：

```bash
javah -classpath . -jni site.zhguixin.test.jni.JniTest
```

其中`site.zhguixin.test.jni.JniTest` 是含有本地方法的JniTest类的完整包名。

```java
package site.zhguixin.test.jni;

public class JniTest {
    private native String print();
}
```

