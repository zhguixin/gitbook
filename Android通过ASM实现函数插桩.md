Android通过ASM实现函数插桩

自定义gradle plugin，该plugin实现plugin接口并继承Transform来实现：在class文件转换为dex文件之前做一些操作

```java
class LifecyclePlugin extends Transform implements Plugin<Project> {

    @Override
    void apply(Project project) {
        // 注册Transform
        def android = project.extensions.getByType(AppExtension)
        android.registerTransform(this)
    }

    @Override
    String getName() {
        return "LifecyclePlugin"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

    @Override
    void transform(@NonNull TransformInvocation transformInvocation) {
      // 借助ASM操作class字节码, 真正实现函数插桩
        ...
        ...
        ...
    }
}
```



