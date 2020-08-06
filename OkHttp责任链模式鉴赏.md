OkHttp责任链模式鉴赏

拦截器链：RealInterceptorChain，维护了存放拦截器(Interceptor)的容器集合、当前执行到的拦截器索引index

RealInterceptorChain的proceed方法触发当前index指向的拦截器：

```java
// 获取当前拦截器
Interceptor interceptor = interceptors.get(index);
// 执行当前拦截器
Response response = interceptor.intercept(next);
// index 加一
index = index + 1;
```

在每个拦截器里面处理完自己的事情，再调用RealInterceptorChain的proceed方法，触发下一个拦截器：

```java
// do self
// ...
chain.proceed();

```

https://blog.csdn.net/qq_15274383/article/details/78485648