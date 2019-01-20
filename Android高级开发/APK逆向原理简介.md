### APK反编译

常用的工具有：

* apktool，使用该工具可以直接反编译APK和重新打包，`apktool d toutiao.apk`,反编译得到smali文件
* dex2jar，通过dex文件得到jar文件
* jd－gui，可以直接将APK拖到这个工具中，查看相关方法、代码，然后去smali文件中搜索相关方法

> 有时候直接解压缩APK文件，可以看到不止一个classes.dex文件，这是启用了MultiDex(gradle中`multiDexEnabled true`)解决方法数超过65536的bug

#### 原理知识
Java虚拟机运行的是Java字节码，基于栈架构；Dalvik虚拟机运行的是Dalvik字节码，基于寄存器架构。

1. Java字节码保存在class文件中，由JVM虚拟机来解析执行；
2. Dalvik字节码由Java字节码转换而来，并打包到一个DEX(Dalvik Executable)可执行文件中，Dalvik虚拟机通过解析DEX文件来执行。

在Android SDK中提供了**dx**工具来讲Java字节码转换为Dalvik字节码，dx工具在打包Java字节码的时候，会消除Java字节码中的冗余信息，对常量池压缩，使得相同的字符串、常量在DEX文件中只会出现一次，从而减小文件体积。

我们编写一个`Hello.java`文件，执行：

```
javac Hello.java
```
得到`Hello.class`字节码，接着执行：

```
dx --dex --output=Hello.dex Hello.class
```
> 这里我们可以把这个dex文件push到手机上，假设push到了`/sdcard/vm/`目录下，然后运行：
>
> ```bash
> adb shell dalvikvm -classpath /sdcard/vm/Hello.dex Hello
> ```
>
> 可以直接查看运行结果。dalvikvm的作用就是创建一个虚拟机并执行参数中指定的Java类，基本使用格式：
>
> ```bash
> dalvikvm -classpath 类路径 类名 
> ```

得到`Hello.dex`文件。得到这两个文件后，我们依次反编译class、dex文件：

```
javap -c -classpath . Hello
dexdump.exe -d Hello.dex // dexdump.exe位于SDK的build-tools目录下
```
对比发现，反编译得到的dex字节码指令要小于class文件的字节码指令。
> JVM虚拟机中，每个线程执行时都有一个PC计数器和一个Java栈。
> PC计数器以字节为单位来记录当前运行位置距离方法开头的偏移量，JVM通过它的值来取指令执行；f
> Java栈以帧为单位，每一个帧称为一个栈帧，每调用一个方法就分配一个新的栈帧压入Java栈。Java栈帧中包括：局部变量区、操作数栈等信息。

#### Dalvik虚拟机介绍
Dalvik同样为每个线程维护一个PC计数器和调用栈，不过调用栈中是一份寄存器列表。
Dalvik有自己的一套指令集，通过反编译工具**apktool**直接反编译APK得到一些`smali`文件，这些文件就是包含Dalvik指令的类似汇编的代码。通过阅读`smali`文件我们可以学习一些Dalvik指令。

在这里我们不借助**apktool**工具，我们自己写一个java文件得到dex文件(见上一节），再通过dex文件得到smali文件，这样可以与java文件作对比，更直观一点。

> apktool实现也是基于**baksmali.jar**、**smali.jar** ，下载地址：https://bitbucket.org/JesusFreke/smali/downloads/

这里借助**baksmali.jar**工具反汇编Hello.dex，执行命令：

```
java -jar baksmali.jar d Hello.dex
```
在当前目录下生成`out/Hello.smali`，得到`Hello.smali`文件。

> 可通过命令：java -jar baksmali.jar -h，查看更多说明

可参考：[smali语法](https://link.jianshu.com/?t=http://blog.isming.me/2015/01/14/android-decompile-smali/)

> Android Studio中有一个插件：java2smali，可以在AS中搜索到，其功能是可以直接把java代码编译为smali，厉害了！

#### APK打包过程
反编译APK之前，也是很有必要要了解一下APK的打包过程的。官方提供的一个APK打包流程图。
整个过程可以分为七个步骤：

1. 通过AAPT工具进行资源文件（包括AndroidManifest.xml、布局文件、各种xml资源等）的打包，生成R.java文件。

   > 大部分文本格式的xml资源会被编译成二进制格式，除了`assets`和`res/raw`

2. 通过AIDL工具处理AIDL文件，生成相应的Java文件。

3. 通过Javac工具编译项目源码，生成Class文件。

   > 项目中使用了，还会借助于NDK编译C/C++代码

4. 通过DX工具将所有的Class文件转换成DEX文件，该过程主要完成Java字节码转换成Dalvik字节码，压缩常量池以及清除冗余信息等工作。

5. 通过ApkBuilder工具将资源文件、DEX文件打包生成APK文件。

   > apkbuilder在3.0之前是一个脚本文件，3.0去掉了该脚本文件，现在使用`%SDK_HOME%/tools/lib/sdklib.jar`来打包APK文件

6. 利用KeyStore对生成的APK文件进行签名。

7. 如果是正式版的APK，还会利用ZipAlign工具进行对齐处理，对齐的过程就是将APK文件中所有的资源文件举例文件的起始距离都偏移4字节的整数倍，这样通过内存映射访问APK文件 的速度会更快


