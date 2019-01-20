---

title: 关于MVP模式的理解
date: 2017-10-23
tags: Android
categories: 

---

MVP，全称 Model-View-Presenter。

MVC、MVP、MVVM，为了解决拥有图像图形界面的程序开发复杂性而产生的模式，是GUI应用程序的出现导致了MVC的产生。

在开发GUI应用程序的时候，会把管理用户界面的层次称为View，应用程序的数据为Model，Model提供数据操作的接口，执行相应的业务逻辑。View和Model之间如何粘合在一起，通过“X”。

当你在应用中只使用Model-View时，到最后，你会发现“所有的事物都被连接到一起”。



##### MVP模式结构

MVP模式的中的P（Presenter）是联系View层和数据层的纽带。以Google官方的讲解MVP的例子，解构一下MVP模式。代码：[googlesamples](https://github.com/googlesamples)/**android-architecture**

主界面是`TaskActivity`，涉及到的几个关键类：

- TaskFragment，View层，负责显示内容
- TaskPresenter，P层，控制显示逻辑
- TasksRepository，M层，固话数据

先看一下Presenter层和View层是如何关联到一起的。

有三个接口类：BaseView、BasePresenter、TaskContract。关系如下：

```java
public interface TasksContract {

    interface View extends BaseView<Presenter> {}
  
  	interface Presenter extends BasePresenter {}
}

/**
* BaseView
*/
public interface BaseView<T> {
    void setPresenter(T presenter);
}

/**
* BasePresenter
*/
public interface BasePresenter {
    void start();
}
```

在TaskActivity中新建TaskFragment，新建TaskPresenter：

```java
// Create the presenter
mTasksPresenter = new TasksPresenter(
  Injection.provideTasksRepository(getApplicationContext()), tasksFragment);
```

在TaskFragment中通过`setPresenter()`方法得到mPresenter。

在TaskPresenter的构造函数中，得到mTaskView：

```java
public TasksPresenter(@NonNull TasksRepository tasksRepository, @NonNull TasksContract.View tasksView) {
  mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null");
  mTasksView = checkNotNull(tasksView, "tasksView cannot be null!");

  // 关联View 和 Presenter
  mTasksView.setPresenter(this);
}
```



