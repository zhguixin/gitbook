Gradle组件打包

新建一个工程**MainApp**作为主工程，新建好之后，再次新建一个**Module**，起名为：**subapp**。此时的工程目录结构为：

> MainApp
>
> --app
>
>   --src
>
>   --build.gradle
>
>   --proguard-rules.pro
>
> --subapp
>
>   --src
>
>   --build.gradle
>
>   --proguard-rules.pro
>
> --build.gradle
>
> --settings.gradle

控制gradle编译打包的过程，希望把subapp编译出来的APK打包进app的assets目录下。

需要修改的几个文件：`app/build.gradle`、`subapp/build.gradle`，新建`app/copy_assets.gradle`。

各个文件具体修改如下：

```groovy
// app/build.gradle
android {
    // 新增
    sourceSets {
        // 指名assets目录
        main {
            debug.assets.srcDirs = ['../assetsdebug']
            release.assets.srcDirs = ['../assetsrelease']
        }
    }
    
}
```

新建`app/copy_assets.gradle`文件

```groovy
afterEvaluate {
    println("after evaluate")
    for (variant in android.applicationVariants) {
        def container = variant.getVariantData().getTaskContainer()
        String mergeTaskName = container.getMergeAssetsTask().name
        println(" mergeTaskName: "  +mergeTaskName)
        def mergeTask = tasks.getByName(mergeTaskName)
        mergeTask.dependsOn ":subapp:copyApkToAssets"
    }
}
```

修改`sub/build.gradle`文件

```groovy
// 新建一个Task
task copyApkToAssets << {
    try {
        println("copyApkTask")
        android.applicationVariants.all { variant ->
            variant.outputs.each { output ->
                def outputFile = output.outputFile
                def flavorName = variant.flavorName
                def buildType = variant.buildType.name
                if (outputFile != null) {
                    def path = "${rootDir.absolutePath}/${flavorName}/assets${buildType}/files"
                    delete fileTree(path) {
                        include '**/*.apk'
                    }
                    copy {
                        from outputFile
                        into rootProject.file(path)
                        rename {
                            fileName ->
                                fileName.replace("-${flavorName}-${buildType}", "")
                        }
                    }
                }
            }
        }
    }
    catch (Exception e) {
        System.err.print("copyApk Exception:" + e)
    }
}
```

打包进app生成的APK的assets目录后，直接安装app工程生成的APK文件，动态加载subapp。

主要相关代码：

```java
private static final String ASSERT_APK_FILE = "SubApp.apk";

// 把SubApp.apk拷贝到data/user/0/<package_name>/files/目录下
String assertName = "files" + File.separator + ASSERT_APK_FILE;
copyAssetsFileToAppFiles(assetName, ASSERT_APK_FILE, mContext);
// 借助于：InputStream is = mContext.getAssets().open(assetFileName);

// 拷贝成功后，直接通过path目录访问SubApp.apk
String path = mContext.getFilesDir().getAbsolutePath() + File.separator + ASSERT_APK_FILE;

// 获取SubApk.apk的PackageInfo信息
PackageInfo packageInfo = mContext.getPackageManager().getPackageArchiveInfo(path, PackageManager.GET_ACTIVITIES | PackageManager.GET_SERVICES | PackageManager.GET_PROVIDERS | PackageManager.GET_RECEIVERS | PackageManager.GET_META_DATA);

// 加载插件SubApp.apk中的资源
Resources resources = getResource(packageInfo.packageName, path);

File dexOpt = mContext.getDir("dexOpt", Context.MODE_PRIVATE);
ClassLoader classLoaders = getClassLoader(packageInfo.packageName, path, dexOpt.getAbsolutePath());

// 相关函数实现
private Resources getResource(String pkg, String path) {
    if (mResources.containsKey(pkg)) {
        return mResources.get(pkg);
    }
    try {
        AssetManager assetManager = AssetManager.class.newInstance();
        Class cls = AssetManager.class;
        Method method = cls.getMethod("addAssetPath", String.class);
        method.invoke(assetManager, path);
        Resources resources = new Resources(assetManager, mContext.getResources().getDisplayMetrics(),
                                            mContext.getResources().getConfiguration());
        mResources.put(pkg, resources);
        return resources;
    } catch (Exception e) {
        return null;
    }

}

private ClassLoader getClassLoader(String pkg, String path, String dexoptPath) {
    if (mClassLoaders.containsKey(pkg)) {
        return mClassLoaders.get(pkg);
    }
    ClassLoader classLoader = new DexClassLoader(path, dexoptPath, null, ClassLoader.getSystemClassLoader());
    mClassLoaders.put(pkg, classLoader);
    return classLoader;
}

//新建类加载器后可动态加载，SubApp.apk中相关类
Class<?> pluginClass = Class.forName("com.test.subapp.MainManager", true, classLoaders);
Object plugin = pluginClass.newInstance();
```

