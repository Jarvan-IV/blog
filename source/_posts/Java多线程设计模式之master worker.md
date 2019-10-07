---
title: Java多线程设计模式之master worker
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


 # Java多线程设计模式之master worker

## 什么是master worker

 Master-Worker 模式是常用的**并行计算模式**。它的核心思想是系统由两类进程协作工作：master进程和worker进程。**master负责接受和分配任务**，worker负责处理子任务。当各个worker子进程处理完成后，会将结果返回给master，由master做归纳和总结。其好处是能将一个大任务分解成若干个小任务，并行执行，从而提高系统的吞吐量。

## 实现

![代码运行图](https://img-blog.csdnimg.cn/2018121622490453.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)
 任务Task.java

```java
public class Task {
    private int id;
    private String name;

    public Task(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}
```

Worker：

```java
public class Worker implements Runnable {

    private ConcurrentLinkedQueue<Task> workQueue;

    private Map<String, Object> resultMap;

    private CountDownLatch countDownLatch;

    public Worker(ConcurrentLinkedQueue<Task> workQueue,
            Map<String, Object> resultMap, CountDownLatch countDownLatch) {
        this.workQueue = workQueue;
        this.resultMap = resultMap;
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        while (true) {
            Task task = this.workQueue.poll();
            if (task == null) {
                break;
            }
            //真正的业务处理
            Object output = handle(task);
            this.resultMap.put("子任务" + task.getId(), output);
        }
        this.countDownLatch.countDown();
    }

    //模拟处理任务
    private Object handle(Task task) {
        Object output;
        try {
            //表示处理task任务的耗时。。。
            Thread.sleep(300);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        output = task.getId();
        return output;
    }
}
```

master：

```java
public class Master {

    //1.承装任务的集合
    private ConcurrentLinkedQueue<Task> workQueue = new ConcurrentLinkedQueue<>();
    //2.使用HashMap承装所有的worker对象
    private Map<String, Worker> workers = new HashMap<>();
    //3.使用一个容器承装每一个worker并行执行任务的结果集
    private Map<String, Object> resultMap = new ConcurrentHashMap<>();
    //如果线程池中的线程60s未被使用，将会从线程池中移除
    private ExecutorService executorService = Executors.newCachedThreadPool();

    //4.构造方法
    public Master(int workerCount, final CountDownLatch countDownLatch) {
        for (int i = 0; i < workerCount; i++) {
            //每一个worker对象都需要有Master的引用workQueue用于任务的领取
            //resultMap用于任务的提交
            Worker worker = new Worker(this.workQueue,this.resultMap,countDownLatch);
            //key表示每一个worker的名字,value表示线程执行对象
            workers.put("子任务" + Integer.toString(i), worker);
        }
    }

    //5.提交方法（将任务提交到任务队列）
    public void submit(Task task) {
        this.workQueue.add(task);
    }

    //6.需要有一个执行的方法（启动应用程序，让所有的worker工作）
    public void execute() {
        for (Map.Entry<String, Worker> entry : workers.entrySet()) {
            executorService.execute(entry.getValue());
        }
    }

    //7.返回结果集（汇总逻辑）
    public int getResult() {
        int ret = 0;
        for (Map.Entry<String, Object> entry : resultMap.entrySet()) {
            ret += (int) entry.getValue();
        }
        return ret;
    }

    public ExecutorService getExecutorService() {
        return executorService;
    }
}
```

测试类client：

```java
public class Client {

    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        //保证10个任务线程执行完毕
        final CountDownLatch countDownLatch = new CountDownLatch(10);
        Master master = new Master(10, countDownLatch);
        for (int i = 1; i <= 100; i++) {
            master.submit(new Task(i, "任务" + i));
        }
        master.execute();
        //阻塞等待任务线程执行完毕
        countDownLatch.await();
        System.out.println(
                "最终结果：" + master.getResult() + " . 执行耗时 : " + (System.currentTimeMillis()
                        - start));
    }
}
```

运行结果如下:

最终结果：5050 . 执行耗时 : **3003**

## 总结

Master-Worker模式是一种将串行任务并行化的方案，被分解的子任务在系统中可以被并行处理（子任务是同级别关系，这些子任务独立，谁处理前，谁处理后都没关系）。
