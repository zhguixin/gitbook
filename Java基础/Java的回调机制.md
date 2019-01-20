---
title: Java的回调机制 
date: 2016-7-29
tags: Java
categories: 
---
Java的回调机制可以使得在执行一个任务完毕的时候，能执行一个callback函数。
现在模拟实现如下一个场景：定义三个类。分别是主函数Main、业务处理类BusinessPro以及回调接口BusinessCallback。使得在业务处理类中，处理完业务后，执行一个callback函数。

Main.java
```java
package zh

public class Main{
	public static void main(String[] args){
		new BusinessProc().compute(100,new BusinessCallback(){
			@Override
			public void onBusinessProcEnd(){
				System.out.println("this is end");
			}
		});
	}
}
```
主函数中新建了一个BusinessPro的实例，入参为一整型和一个匿名的内部接口类并实现了其方法`onBusinessProcEnd()`。

BusinessProc.java
```java
package zh

public class BusinessProc{
	public void compute(int n, BusinessCallback callback){
		int sum = 0;
		for(int i=0;i<n;i++)
			sum += i;
		System.out.println(sum);
		
		callback.onBusinessProcEnd();
	}
}
```
业务处理类中的`compute()`方法则实现具体的业务，该实例简单演示1到n的和。实现业务处理完毕后执行回调函数，`compute()`方法包含两个入参：一个整型和一个接口类。

BusinessCallback.java
```java
public interface BusinessCallback{
	public void onBusinessProcEnd();
}
```
回调接口定义了一些回调方法，以供使用。


