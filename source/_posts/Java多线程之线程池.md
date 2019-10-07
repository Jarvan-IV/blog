---
title: Java多线程之线程池
date: 2019-9-22 23:20:00
author: xp
categories:
- Java
- 多线程
tags:
- Java
- 多线程
- 线程安全
---


# Java多线程之线程池

## 是什么

在多线程开发中，如果直接这样写：

```java
new Thread(new Runnable() {
@Override
    public void run() {
        // TODO Auto-generated method stub
        }
    }
).start();
```

如果并发的请求数量非常多，但每个线程执行的时间很短，这样就会频繁的创建和销毁线程，如此一来会大大降低系统的效率。可能出现服务器在为每个请求创建新线程和销毁线程上花费的时间和消耗的系统资源要比处理实际的用户请求的时间和资源更多。同时线程缺乏统一管理，可能无限制新建线程，相互之间竞争，及可能占用过多系统资源导致死机或oom。

线程池的作用就是线程完一个任务，并不被销毁，而是直接回收。线程池为线程生命周期的开销和资源不足问题提供了解决方案。通过对多个任务重用线程，线程创建的开销被分摊到了多个任务上。有如下好处：

 -重用存在的线程，减少对象创建、消亡的开销，性能佳。
 -可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
 -提供定时执行、定期执行、单线程、并发数控制等功能。

## 创建方式

Java通过Executors提供四种线程池，分别为：
 -newFixedThreadPool ：该方法返回一个固定数量的线程池，该方法的线程数始终不变，当有一个任务提交时，若线程池中空闲，则立即执行，若没有，则会被暂缓在一个任务队列中等待有空闲的线程去执行。
 -newSingleThreadExecutor ：该方法创建一个线程的线程池，若空闲则执行，若没有空闲线程则暂缓在任务队列中。
 -newCachedThreadPool：该方法返回一个可根据实际情况调整线程个数的线程池，不限制最大线程数量。若有一个任务并且无空闲线程，就会创建一个线程，若有空闲的线程则直接执行任务，若无任务则不创建线程。并且每一个空闲线程会在60s后自动回收。
 -newScheduledThreadPool ：该方法返回一个ScheduledExecutorService 对象，创建一个定长线程池，支持定时及周期性任务执行。

### newFixedThreadPool

构造器如下：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

### newSingleThreadExecutor

构造器如下：

```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

### newCachedThreadPool

构造器如下：

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

### newScheduledThreadPool

构造器如下：

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

下面用newScheduledThreadPool实现一个简单的定时任务功能：

```java
public class ScheduleThreadPoolDemo {

    static class Task implements Runnable {

        @Override
        public void run() {
            System.out.println("定时run");
        }
    }

    public static void main(String[] args) {
        Task task = new Task();
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(1);
        //2秒后开始，没隔5秒执行task
        scheduledExecutorService.scheduleWithFixedDelay(task, 2, 5, TimeUnit.SECONDS);
    }
}
```

其中scheduleWithFixedDelay方法参数如下：

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,long delay,TimeUnit unit);

}
```

 -command：要执行的任务
 -initialDelay：延迟首次执行时间
 -delay：延迟间隔时间
 -unit：时间单位

上述代码表示：

### 自定义线程池（推荐*****）

若Executors工厂类无法满足我们的需求，可以自己去创建自定义的线程池，其实Executors工厂类里面的创建线程方法其内部均是用了ThreadPoolExecutor这个类，这个类可以自定义线程，构造方法如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {}
```

 -corePoolSize：核心池的大小，如果调用了prestartAllCoreThreads()或者prestartCoreThread()方法，会直接预先创建corePoolSize的线程（初始化corePoolSize个线程），否则当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；这样做的好处是，如果任务量很小，那么甚至就不需要缓存任务，corePoolSize的线程就可以应对；
 -maximumPoolSize：线程池最大线程数，表示在线程池中最多能创建多少个线程，如果运行中的线程超过了这个数字，那么相当于线程池已满，新来的任务会使用RejectedExecutionHandler 进行处理；
 -keepAliveTime：表示线程没有任务执行时（空闲线程）最多保持多久时间会终止，然后线程池的数目维持在corePoolSize 大小；
 -unit：持续时间的单位。
 -workQueue：一个阻塞队列，用来存储等待执行的任务，如果当前对线程的需求超过了corePoolSize大小，才会放在这里；
 -threadFactory：线程工厂，主要用来创建线程，比如可以指定线程的名字；
 -handler：拒绝策略，如果线程池已满，新的任务的处理方式

自定义线程池构造方法的队列是什么类型是很关键的：

 -在使用有界队列（ArrayBlockingQueue）时，若有新的任务需要执行，如果线程池实际线程数小于corePoolSize，则优先创建线程，若大于corePoolSize，则会将任务加入队列等待空闲线程来执行，若队列已满，则在总线程数不大于maximumPoolSize的前提下，创建新的线程，若线程数大于maximumPoolSize，则执行拒绝策略。

```java
public class ThreadPoolExecutorDemo {

    static class MyTask implements Runnable {

        private String name;

        public MyTask(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程：" + Thread.currentThread().getName() + "正在执行" + name);
        }
    }

    static class JyThreadFactory implements ThreadFactory {

        private AtomicInteger count = new AtomicInteger();

        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("ThreadPoolExecutorDemo thread：" + count.getAndAdd(1));
            return t;
        }
    }

    public static void main(String[] args) {
        MyTask myTask1 = new MyTask("任务1");
        MyTask myTask2 = new MyTask("任务2");
        MyTask myTask3 = new MyTask("任务3");
        MyTask myTask4 = new MyTask("任务4");
        MyTask myTask5 = new MyTask("任务5");
        MyTask myTask6 = new MyTask("任务6");
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                1,//corePoolSize
                2,//maximumPoolSize
                3,//keepAliveTime
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(3),//队列容量为 3
                new JyThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
        //执行第一个任务，由于 corePoolSize为1，所以，线程池里面的线程会直接执行myTask1
        threadPoolExecutor.execute(myTask1);
        //执行第二个任务时，由于corePoolSize容量为1，有一个线程正在执行myTask1
        //所以此时的myTask2任务会直接进入队列，此时队列大小为1，
        // 由于队列的容量为3，队列未满，所以此时不会创建线程，而是继续在队列中等待上个任务处理完毕
        threadPoolExecutor.execute(myTask2);
        //此时myTask1仍未处理完毕，所以将任务myTask3放入队列，此时队列大小为2，
        // 没有超过队列的限制容量3，队列未满，所以此时不会创建线程，而是继续在队列中等待上个任务处理完毕
        threadPoolExecutor.execute(myTask3);
        //此时myTask1仍未处理完毕，所以将任务myTask4放入队列，此时队列大小为3，队列刚满
        // 但是没有超过队列的限制容量3，所以此时不会创建线程，而是继续在队列中等待上个任务处理完毕
        threadPoolExecutor.execute(myTask4);
        //此时myTask1仍未处理完毕，所以将任务myTask5放入队列，此时队列大小为4
        //超过了队列的限制容量3，此时创建了一个新的线程，由运行结果看，myTask1和myTask5是同时执行的，
        // 此时，线程池里面总共有2个线程，没有超过maximumPoolSize。
        // 由于创建了一个线程并且从任务队列中取出了一个任务，所以此时队列大小减为3。
        threadPoolExecutor.execute(myTask5);
        //再加入一个新的任务myTask6，此时队列的大小为4，超过了容量限制3，并且线程池里面总的线程数已经达到了maximumPoolSize。
        //所以此时执行任务myTask6会被拒绝策略拒绝。
        threadPoolExecutor.execute(myTask6);
        threadPoolExecutor.shutdown();
    }
}
```

运行结果如图：
![有界队列](https://img-blog.csdnimg.cn/20181111235914957.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

> 运行结果中的打印顺序很重要！！

 -无界的任务队列时（LinkedBlockingQueue），与有界队列相比，除非系统资源耗尽，否则无界的任务队列，不存在任务入队失败的情况。当有新任务到来，系统的线程数小于corePoolSize时，则新建线程执行任务。当达到corePoolSize后，就不会继续增加，若后续仍有新的任务加入，而又没有空闲的线程资源，则任务直接进入队列等候，**即maximumPoolSize不起作用**。若任务创建和处理的速度差异很大，无界队列会保持快速增长，直到耗尽系统资源。

```java
public class ThreadPoolExecutorLinkedDemo implements Runnable{
    private int taskId;

    public ThreadPoolExecutorLinkedDemo(int taskId) {
        this.taskId = taskId;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("任务："+ taskId);
    }

    public static void main(String[] args) {
        BlockingQueue blockingQueue = new LinkedBlockingQueue<>();
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                2,//corePoolSize
                10,//maximumPoolSize（linked时无用）
                3,//keepAliveTime
                TimeUnit.SECONDS,
                blockingQueue,//队列容量为 3
               new ThreadPoolExecutor.AbortPolicy());
        for (int i = 0;i< 8;i++) {
            threadPoolExecutor.execute(new ThreadPoolExecutorLinkedDemo(i));
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("队列容量："+blockingQueue.size());
    }
}
```

结果图：
![运行中打印先后顺序很重要](https://img-blog.csdnimg.cn/2018111200175986.jpg)

> ps：可以通过运行程序，看打印结果得知，线程池中的线程总数是corePoolSize。

#### 拒绝策略

有以下几种：

 1. 在默认的 ThreadPoolExecutor.AbortPolicy 中，处理程序遭到拒绝将抛出运行时RejectedExecutionException。其他任务继续执行。
 2. 在 ThreadPoolExecutor.CallerRunsPolicy 中，线程调用运行该任务的execute 本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。只要线程池未关闭，该策略直接在调用者线程中，运行当前丢弃的任务。
 3. 在 ThreadPoolExecutor.DiscardPolicy中 ，丢弃无法处理的任务，不给予任务处理。
 4. 在 ThreadPoolExecutor.DiscardOldestPolicy 中，如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）。即丢弃最老的一个请求，尝试再次提交当前任务。
 5. 自定义拒绝策略，实现RejectedExecutionHandler接口。
