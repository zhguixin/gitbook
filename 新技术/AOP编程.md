AOP编程，即切面编程，是一种编程思想；区别于OOP这种面向对象的编程思想，OOP把问题划分到每一个模块中去管理；AOP则将这些模块共有的问题，抽取出来单独管理，比如：权限检查、日志输出等。

就像OOP的实现语言有Java一样，AOP的实现有AspectJ。

AspectJ是Java语言的一种扩展，使用AspectJ有两种方法：

* 完全使用AspectJ语言，可以在AspectJ中调用Java类库；

  ```java
  // SecurityAspect.aj
  package ajia.security;
  
  public aspect SecurityAspect {
      private Authenticator authenticator = new Authenticator();
      pointcut secureAccess():execution(* ajia.messaging.MessageCommunicator.deliver(..));
      before():secureAccess(){
          System.out.println("Checking and authenticating user");
          authenticator.authenticate();
      }
  }
  ```

* 使用纯Java语言，然后使用`@AspectJ` 注解

> 在实际开发中，推荐使用注解的方式。aj文件需要特别的编译器才能识别



参考：

[Android中使用AspectJ](https://www.jianshu.com/p/e152b34b785b)

[深入理解Android之AOP](https://blog.csdn.net/innost/article/details/49387395)