#### 1、WorkManager

workmanager可以在应用退出的时候，依然保证后台任务的继续执行。workmanager会根据API的版本选择调用JobScheduler或者AlarmManager。

涉及到几个类：

- Worker：抽象类，实现`doWork` 方法，实现具体的任务

- WorkRequest，对Worker进行一次包装，添加任务执行时的一些细节，比如：网络环境，低电量等

  但是这里的WorkRequest是一个抽象类，我们会使用它的子类：OneTimeWorkRequest或者PeriodicWorkRequest

  ```java
  OneTimeWorkRequest uploadRequest =
      new OneTimeWorkRequest.Builder(UploadWorker.class)
      .setInputData(createInputData())
      .addTag(TAG_OUTPUT)
      .build();
  ```

- WorkManager，对任务进行排队、管理和调度。

  ```java
  mWorkManager = WorkManager.getInstance();
  mWorkManager.enqueue(uploadRequest);
  ```

- WorkStatus，包含特定的任务信息，WorkManger为每一个任务，提供的是LiveData包裹的WorkStatus对象

  ```java
  mWorkManager.getStatusesByTag(TAG_OUTPUT) : LiveData<List<WorkStatus>>
  ```

  如此一来，我们可以借助ViewModel，在Activity中观察数据的变化，最终反应到UI上去。

另外，WorkManager可以处理多个任务，这几个任务的调用策略，可以在交给WorkManager时指定。

```java
mWorkManager.beginWith(work1,work2)
    .then(work3)
    .then(work4)
    .enqueue();
```

更复杂的任务链式调用可以借助于：WorkContinuation。