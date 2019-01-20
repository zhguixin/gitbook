### JobScheduler使用

#### 1、简介

Google在Android5.0加入该Service的相关API，意在解决稍后的某个时间点或者当满足某个特定的条件时执行一个任务(比如连接上了wifi、充电器、在某个CPU空闲时段批量处理任务)，这允许你的应用执行某些指定的任务时不需要考虑时机控制引起的电池消耗。

#### 2、创建JobService服务

和普通的Service一样，自定义的Service需要继承JobService这个类，必须要实现onStartJob和onStopJob两个方法。

```java
    public class MyJobSchedulerService extends JobService{

      /*
      	该方法用来执行用户指定的工作，大量的任务一定要放在子线程。这里有个返回值，
      	返回值是false,系统假设这个方法返回时任务已经执行完毕；
      	返回值是true,那么系统假定这个任务正要被执行。
      	如果耗时的任务放在了子线程里，这里必须要返回true，来告知系统这个任务还在执行。由用户在子线程中，执行完相应的任务后，主动调用jobFinished(JobParameters params, boolean needsRescheduled)来通知系统，任务已经执行完毕。
      */
        @Override
        public boolean onStartJob(JobParameters params) {
            return false;
        }

        @Override
        public boolean onStopJob(JobParameters params) {
            return false;
        }
    }
```

使用JobService需要在AndroidManifest中添加权限：

```java
<service android:name=".JobSchedulerService"
    android:permission="android.permission.BIND_JOB_SERVICE" />
```

#### 3、创建JobScheduler

接下来一步，就是要创建JobScheduler对象与JobService进行交互。

我们在Activity中可以通过`getSystemService( Context.JOB_SCHEDULER_SERVICE )`初始化了一个叫做mJobScheduler的JobScheduler对象。

```java
mJobScheduler = (JobScheduler) 
    getSystemService( Context.JOB_SCHEDULER_SERVICE );
```

然后通过`JobInfo.Builder`来构建一个JobInfo对象，然后传递给Service。

```java
//JobInfo.Builder有两个参数，第一个是标示Id，表示要运行的任务的标识符（类似于NotificationManger的标示Id）；第二个参数就是Service组件的类名
JobInfo.Builder builder = new JobInfo.Builder( 1,
        new ComponentName( getPackageName(), 
            MyJobSchedulerService.class.getName() ) );
//通过builder我们可以来确定任务的各种执行方式，此处设置任务可以每隔三秒运行一次
builder.setPeriodic( 3000 );

//运行任务
mJobScheduler.schedule(builder.build());


/*
   应用想停止某个任务，你可以调用JobScheduler对象的cancel(int jobId)来实现；
   想取消所有的任务，你可以调用JobScheduler对象的cancelAll()来实现。
*/
```



