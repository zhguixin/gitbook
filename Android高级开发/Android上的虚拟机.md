Android系统中在5.0之前使用的是Dalvik虚拟机，在Android5.0之后Android Runtime（ART）就彻底替代了原先的Dalvik，成为Android系统上新的虚拟机。

### Dalvik虚拟机介绍

####Dalvik启动

Dalvik虚拟机是在Zygote进程启动的时候创建的，创建了DVM（表示Dalvik虚拟机）后还会将Java运行时库加载进来，这样后面每个从Zygote进程fork出来的进程不仅会得到一个DVM的拷贝，还会与Zygote进程共享Java运行时库。

####Dalvik功能

DVM脱胎于JVM，包括：内存管理、垃圾收集、JIT编译、JNI和进程线程管理。

其中JIT(Just-In-Time)编译，是在程序运行的过程中进行编译的。DVM会将热点代码通过JIT编译为本地机器码，加快执行速度，其他的代码则按照对应指令进行解释执行。

####Dalvik指令集

DVM使用的指令是基于寄存器的，类文件格式使用的是Dex（Dalvik Executable）。相比于JVM，可以减少了指令操作进栈、出栈的内存消耗，并且通过`dx`打包class得到dex文件，使得字节码中的在DVM中的内存排布更加紧凑。

参考：

[Android上的Dalvik虚拟机](https://paul.pub/android-dalvik-vm/)

[罗升阳:Dalvik虚拟机简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/8852432)

[Android2.3的dalvik源码](https://www.androidos.net.cn/android/2.3.7_r1/xref/dalvik)

### ART虚拟机介绍

ART虚拟机是在Dalvik虚拟机基础上进行优化的，直观的感受是程序的运行速度要更快了。ART虚拟机通过AOT（Ahead-Of-Time）在程序APK运行之前将Dex字节码进行翻译，得到本地机器码，运行时直接执行本地机器码从而提高运行速度。

> 在Dalvik虚拟机上，APK中的Dex文件在安装时也会做，被优化成了odex文件，在运行时，会被JIT编译器编译成native代码

AOT编译的具体过程，当APK在安装的时候通过`dex2oat`的工具将dex文件编译成`oat`文件，并保存在`/data/dalvik-cache/`目录下，因为会兼容32位的CPU，会有两个文件：

- /data/dalvik-cache/arm/
- /data/dalvik-cache/arm64/

生成的`oat`文件，既包含了dex文件中原先的内容，也包含了已经编译好的native代码，`oat`文件是遵循ELF格式的，ELF是Linux系统上的可执行文件。

dex文件可以通过`dexdump`工具进行分析。oat文件也有对应的dump工具，这个工具就叫做`oatdump`，可以借助这个文件来进一步分析oat文件。



参考：

[Android上的ART虚拟机](https://paul.pub/android-art-vm/#id-art-vs-dalvik)

[罗升阳：Android运行时ART简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/39256813)

[Android5.1的art源码](https://www.androidos.net.cn/android/5.1.0_r3/xref/art)