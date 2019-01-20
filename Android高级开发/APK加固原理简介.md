为APK作加固，可以反之被反编译破解的风险。

#### 进一步增加混淆

Android默认提供了混淆规则，用26个英文字母来替代方法名和类名，开启方式：

```groovy
buildTypes {
    release {
        minifyEnabled false
        shrinkResources false

        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard.flags'
    }
}
```

> 混淆也可以减少APK体积；不打包未使用的方法，避免65535问题

为了使用更多的字符来替换方法名或者类名，可以修改`Proguard.java`来扩展混淆字符集。



#### APK加壳

该方案的基本思路是将APK用加密算法加密后，放入到一个`空程序`的DEX文件中，该空程序被称为**脱壳**程序，负责解密APK，然后动态加载该APK。其中涉及到的技术有：DEX文件修改、加密算法、动态加载。

可参考：[Android中的Apk的加固(加壳)原理解析和实现](https://blog.csdn.net/jiangwei0910410003/article/details/48415225)

