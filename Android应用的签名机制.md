Android应用的签名机制

Android系统为保护应用不被篡改，要求对安装的应用必须进行签名。签名证书和私钥由开发者保存。以后安装的程序会进行校验，如果系统中已经存在了一个相同的包名和签名的应用，将会用新安装的应用替换旧的；如果包名相同但是签名不同，则会安装失败。



对生成的APK进行解压缩，在META-INF目录下会有三个文件：CERT.RSA、CERT.SF和MANIFEST.MF。

- **MANIFEST.MF**中保存了所有其他文件的SHA1摘要并base64编码后的值。
- **CERT.SF**文件 是对MANIFEST.MF文件中的每项中的每行加上“rn”后，再次SHA1摘要并base64编码后的值（这是为了防止通过篡改文件和其在MANIFEST.MF中对应的SHA1摘要值来篡改APK，要对MANIFEST的内容再进行一次数字摘要）。
- **CERT.RSA** 文件：包含了签名证书的公钥信息和发布机构信息。

对安装包的校验过程在源码的`frameworks/base/core/java/android/content/pm/PackageParser.java` 类中可以看到。



开发过程中调试用的Debug版本的APK，所用的签名的默认密钥存在debug.keystore文件中。在linux和Mac上debug.keystore文件位置是在**~/.android**路径下，在windows目录下文件位置是**C:\user\用户名.android**路径下。



```
apksigner sign --ks my-release-key.jks my-app.apk
```

