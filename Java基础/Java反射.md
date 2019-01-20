#### 简介

程序运行时，动态的将类加载到JVM中。比如，Spring根据配置文件动态加载类。

反射相关的类一般都在java.lang.relfect包里。

#### 获得Class对象的方式

1.1 Object 的 getClass();方法

1.2 任何数据类型（包括基本数据类型）都有一个**静态** 的class属性

1.3 通过Class类的静态方法：`forName(String className)`(常用)

> 在运行期间，一个类，只有一个Class对象产生

#### 创建实例对象

1、使用Class对象的newInstance方法

```java
Class<?> clazz = Class.forName("com.zgx.ReflectTest");
Object obj = clazz.newInstance();
```

2、获得构造函数Constructor ，调用Constructor  的newInstance方法（可传入参数）

```java
Class<?> clazz = Class.forName("com.zgx.ReflectTest");
// 获取ReflectTest类带一个String参数的构造器,并调用她的构造方法
Object obj = clazz.getConstructor(String.class).newInstance("param");
```

#### 获取方法

1、获取所有的public方法

```java
public Method[] getMethods() throws SecurityException
```

2、获取所有的方法，包括public、private、protected

```java
public Method[] getDeclaredMethods() throws SecurityException
```

3、返回一个特定的方法

```java
// 第一个参数为方法名称，后面的参数为方法的参数对应Class的对象
public Method getMethod(String name, Class<?>... parameterTypes)
```

#### 方法调用

获得到`Method` 实例后，就可以通过**invoke**触发方法调用。

```java
// 第一个参数是反射得到的类实例对象，第二个参数是要传入方法的参数
public Object invoke(Object obj, Object... args) 
```

比如：

```java
try {
    // 反射得到Class对象
    Class<?> clazz = Class.forName("com.zgx.ReflectTest");
    // 反射得到类实例对象
    Object obj = clazz.getConstructor(String.class).newInstance("param");
    // 反射得到方法实例
    Method method = clazz.getMethod("print", String.class);
    // 触发方法调用
    method.invoke(obj, "hello");
} catch (Exception e) {
    e.printStackTrace();
}
```

#### 成员变量获取

getFiled: 访问公有的成员变量

getDeclaredField：所有已声明的成员变量。但不能得到其父类的成员变量

```java
Field field = clazz.getDeclaredField("mName");
field.setAccessible(true);// 解除私有属性设置
field.set(obj, "zgx"); // 为属性赋值，相当于调用语句：mName = "zgx"
```

getFileds和getDeclaredFields获取所有的成员变量。