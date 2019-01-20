Android资源打包的时候，会将assets目录下、以及raw目录下的文件，直接打包到APK中。其他资源会通过aapt工具，生成资源ID，放到R.java文件中，然后对xml文件进行二进制压缩，放到resource.arsc文件中，这样做的目的：一个是减少资源空间、另外一个就是增加文件安全性（二进制文件不容易查看）。

在应用中，会通过AssetManager和Resource进行资源文件的查找。

其中，AssetManager根据文件名查找资源；Resource根据资源ID查找资源，如果资源ID对应的是一个文件，会进一步通过AssetManager来打开。

![资源编译打包](http://img.my.csdn.net/uploads/201304/01/1364831227_6705.jpg)

APK安装流程

点击APK安装，弹出APK确认安装界面（对应于PackageInstallerActivity ），用户点击确定后，启动APK的真正安装流程。

首先显示安装进度的界面（对应于InstallAppProgress），在此过程中：

* 会先将APK 拷贝到/data/app目录下（DefaultContainerService ），解压并扫描安装包；
* 在/data/data/pkg-name。创建以应用包名为目录名的文件夹；
* 然后对dex文件进行优化，并保存在dalvik-cache目录下 
* PackageManagerService（PMS），扫描AndroidManifest文件，注册四大组件信息到PMS中
* 发送安装完成广播

