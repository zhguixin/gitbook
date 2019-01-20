### Java注解

#### 注解的分类

- 源码注解（SOURCE），注解只在源码中存在，编译之后就没有了；
- 编译时注解（CLASS），@Override、@Deprecated、@SupperesWarnings，在class文件中也存在；
- 运行时注解（RUNTIME），运行时也起作用，会影响运行逻辑。

#### 自定义注解

```java
@Target({ElementType.METHORD,ElementType.Type})//作用域，方法和类
@Retention(RetentionPoilcy.RUNTIME)//运行时注解
@Inherited//标识性注解，允许子类继承
@Documented//生成JavaDoc时包含此注解
public @interface Description{
  String desc();//注解成员无参、无异常式声明
  int age() default 25;
}
//只有一个方法时，名称必须为String value(),使用时简洁
//成员类型受限，支持基本数据类型、String
//没有成员的注解为标识注解，只起标识作用

@Description(desc="hello",age=18)
public void print(){}
```

#### 解析注解

```java
    try {
       Class cl = Class.forName("");
       //类上面的注解是否存在
       boolean isExist =   cl.isAnnotationPresent(Description.class);
       if(isExist){
         //解析类上面的注解
          Description d = cl.getAnnotation(Description.class);
          System.out.println(d.value());
        }
    } catch (ClassNotFoundException e) {
            e.printStackTrace();
    }
```

在Retrofit中，我们通过注解的方式定义网络访问接口：

```java
/**
 *	获取最美壁纸中图片信息
*/
@GET("http://lab.zuimeia.com/wallpaper/category/1/")
Observable<HttpResultV1<ImageData>> getImage(@Query("page_size") int size);
```

简单看一Retrofit源码中，对注解的处理过程，ServiceMethod.Builder 类的构造方法中：

```java
public Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
  this.methodAnnotations = method.getAnnotations();//获取方法上的注解
  this.parameterTypes = method.getGenericParameterTypes();//获取方法参数列表
  this.parameterAnnotationsArray = method.getParameterAnnotations();//由于一个参数会有多个注解，此处获取方法参数注解的二维数组
}
```

然后在build方法中，对注解进行解析：

```java
public ServiceMethod build() {
  	// 处理方法上的注解
    for (Annotation annotation : methodAnnotations) {
    	parseMethodAnnotation(annotation);
  	}
  	// 处理参数注解
    int parameterCount = parameterAnnotationsArray.length;
  	parameterHandlers = new ParameterHandler<?>[parameterCount];
  	for (int p = 0; p < parameterCount; p++) {
    	Type parameterType = parameterTypes[p];
    	if (Utils.hasUnresolvableType(parameterType)) {
      		throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s", parameterType);
    	}

    	Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
    	if (parameterAnnotations == null) {
      		throw parameterError(p, "No Retrofit annotation found.");
    	}

    	parameterHandlers[p] = parseParameter(p, parameterType, 	parameterAnnotations);
  	}
}
```



通过反射解析注解是比较影响效率的，在ButterKnife中，使用到了APT（Android Processing Tool ）编译时解析技术。大概的过程就是，将注解的生命周期指定为CLASS，注解类继承`AbstractProcessor`，编译的时候会扫描所有要处理的注解类，调用`AbstractProcessor` 的`process`方法，对注解进行处理。处理完后，借助于**javapoet**生成java文件，一同生成class文件。



参考文章：[注解基础](https://juejin.im/post/5a1517a6f265da4312808f1b)、[ButterKnife源码分析](http://www.jianshu.com/p/0f3f4f7ca505)、[Android 如何编写基于编译时注解的项目](https://blog.csdn.net/lmj623565791/article/details/51931859)

