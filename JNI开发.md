JNI是Java语言提供的与C/C++交互的规范，Android提供了NDK这一套工具集来允许使用C/C++实现应用的部分功能。

添加NDK支持，在`local.properties`文件中新增：

```
cmake.dir=D\:\\AndroidSdk\\sdk\\cmake\\3.10.2.4988404
ndk.dir=D\:\\AndroidSdk\\sdk\\ndk-bundle
```

配置CMake，在项目根module的build.gradle中添加：

```groovy
android {
    defaultConfig { // defaultConfig加入
            externalNativeBuild {
                cmake { // 填写 CMake 的命令参数
                    cppFlags "-std=c++11"
                    abiFilters 'armeabi-v7a', 'arm64-v8a' // 指定编译so库的CPU架构
                }
            }
    }
    
   externalNativeBuild { // android加入
        cmake { // 指明了CMakeList.txt的路径
            path "src/main/cpp/CMakeLists.txt"
            version "3.10.2"
        }
    }
}
```

在`build.gradle`里面看到，有两个地方用到了`externalNativeBuild`，一个是在`defaultConfig`里面，是一个是在`defaultConfig`外面。

#### CMake的运转流程

> - 1、Gradle 调用外部构建脚本`CMakeLists.txt`
> - 2、CMake 按照构建脚本的命令将 C++ 源文件 `native-lib.cpp` 编译到共享的对象库中，并命名为 `libnative-lib.so` ，Gradle 随后会将其打包到APK中
> - 3、运行时，应用的`MainActivity` 会使用`System.loadLibrary()`加载原生库。应用就是可以使用库的原生函数`stringFromJNI()`。

#### CMakeLists.txt文件

`CMakeLists.txt`我们看到这里主要是分为四个部分

```cmake
cmake_minimum_required(VERSION 3.4.1)

add_library( # Sets the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )


find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```

四个部分解释如下：

> - cmake_minimum_required(VERSION 3.4.1)：指定CMake的最小版本
> - add_library：创建一个静态或者动态库，并提供其关联的源文件路径，开发者可以定义多个库，CMake会自动去构建它们。Gradle可以自动将它们打包进APK中。
>   - 第一个参数——native-lib：是库的名称
>   - 第二个参数——SHARED：是库的类别，是`动态的`还是`静态的`
>   - 第三个参数——src/main/cpp/native-lib.cpp：是库的源文件的路径
> - find_library：找到一个预编译的库，并作为一个变量保存起来。由于CMake在搜索库路径的时候会包含系统库，并且CMake会检查它自己之前编译的库的名字，所以开发者需要保证开发者自行添加的库的名字的独特性。
>   - 第一个参数——log-lib：设置路径变量的名称
>   - 第一个参数—— log：指定NDK库的名子，这样CMake就可以找到这个库
> - target_link_libraries：指定CMake链接到目标库。开发者可以链接多个库，比如开发者可以在此定义库的构建脚本，并且预编译第三方库或者系统库。
>   - 第一个参数——native-lib：指定的目标库
>   - 第一个参数——${log-lib}：将目标库链接到NDK中的日志库，

工程编写打包好so库，可以添加到目标工程中去。

> 编译得到的so库位置：`build\intermediates\cmake\release\obj\arm64-v8a\`

JNI注册时机：

- 在Zygote进程，启动完虚拟机后，调用`startReg(AndroidRuntime.cpp)`进行注册

- 调用System.loadLibrary()直接加载so库

  > System.loadLibrary("android_jni");
  >
  > 会从指定lib目录下查找，并加上lib前缀和.so后缀

JNI静态注册过程，在通过javah命令生成对应类全限定名的`*.h`文件，应用侧开发使用比较多；

JNI动态注册过程，在so库被加载后，会回调`JNI_onLoad()`方法，可以在这个回调方法里对JNI方法进行动态注册。

动态注册过程就是将Java方法与C/C++方法关联的过程，JNINativeMethod记录了这个关联过程

```java
typedef struct {
    const char* name;//Java方法的名字
    const char* signature;//Java方法的签名信息
    void*       fnPtr;//JNI中对应的方法指针
} JNINativeMethod;
```

> Java方法的签名信息，比如有个Java方法：`String getStr(int index，long[] arr)`
>
> 签名信息为：`(I[J)Ljava/lang/String`
>
> 参数在前用括号包裹返回值在后，Java引用类型表示为**L+包名**，**[**：表示数组

构造一个JNINativeMethod数组：

```java
static const JNINativeMethod gMethods[] = {
	...
 	{"start",            "()V",      (void *)android_media_MediaRecorder_start},//1
	...
};
```

在JNI_onLoad方法中实现

```cpp
//回调函数
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved){
  JNIEnv* env = NULL;
  //获取JNIEnv
  if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
    return -1;
  }
  assert(env != NULL);
  //注册函数 registerNatives ->registerNativeMethods ->env->RegisterNatives
  if(!registerNatives(env)){
    return -1;
  }
  //返回jni 的版本 
  return JNI_VERSION_1_6;
}
```

参考：JNI的两种注册过程https://www.jianshu.com/p/1d6ec5068d05



