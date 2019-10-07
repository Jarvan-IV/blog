---
title: Java多线程之CountDownLatch、CyclicBarrier和Semaphore
date: 2019-9-21 23:55:00
author: xp
categories:
- Java
- 多线程
tags:
- Java
- 多线程
- 线程安全
---

# Java多线程之CountDownLatch、CyclicBarrier和Semaphore

> 在java 1.5中，提供了一些非常有用的辅助类来帮助我们进行并发编程，比如CountDownLatch，CyclicBarrier和Semaphore，今天我们就来学习一下这三个辅助类的用法。

## CountDownLatch

CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。

```java
public class CountDownLatchUtil {

    public static void main(String[] args) {
        final CountDownLatch countDown = new CountDownLatch(2);
        Thread t1 = new Thread(() -> {
            System.out.println("进入t1线程，等待其他线程初始化。。。。");
            try {
                countDown.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("t1线程执行完毕。。");
        });
        Thread t2 = new Thread(() -> {
            System.out.println("进入t2线程。。。。");
            countDown.countDown();
            System.out.println("t2线程执行完毕。。");
        });
        Thread t3 = new Thread(() -> {
            System.out.println("进入t3线程。。。。");
            countDown.countDown();
            System.out.println("t3线程执行完毕。。");
        });
        t1.start();
        t2.start();
        t3.start();
    }

}
```

运行结果如下：
![countdown](https://img-blog.csdnimg.cn/20181111102341167.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量（构造器中传入）。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

构造器中的计数值（count）实际上就是闭锁需要等待的线程数量。这个值只能被设置一次，而且CountDownLatch没有提供任何机制去重新设置这个计数值。

与CountDownLatch的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

### 使用场景

&ensp;&ensp;&ensp;&ensp;比如我们现在需要做一道菜，叫做**开水煮白菜**（自创的。。。哈哈），假设步骤如下：首先，将水烧开（需要3秒），然后将白菜倒入已烧开的水中（需要5秒）。这时，烧水的过程和洗菜的过程，完全可以同时进行。用代码描述如下：

```java
public class CountDownLatchDemo {
    //开水煮白菜
    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch countDown = new CountDownLatch(1);
        long time = System.currentTimeMillis();
        Thread t1 = new Thread(() -> {
            System.out.println("正在烧水。。。。。");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("水已烧开。。蔬菜可以下锅");
            countDown.countDown();
        });
        t1.start();
        System.out.println("正在清洗白菜。。。。");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("白菜清洗完毕。。。。");
        countDown.await();
        System.out.println("将白菜倒入沸水中" + " 共耗时：" + (System.currentTimeMillis() - time));
    }

}
```

运行结果如下：
![demo](https://img-blog.csdnimg.cn/20181111113258421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

跟以上案例比较接近的时候，比如我们在使用zookeeper的zkclient的时候，首先开启一个线程建立zookeeper的连接，然后就是在获取到连接之后，再对zookeeper的节点进行操作。这里的建立连接就好比上面场景的烧开水；对节点进行操作就好比洗白菜；当连接建立之后，就直接执行操作节点的指令，好比将白菜倒入沸水。

## CyclicBarrier

字面意思回环栅栏，**通过它可以实现让一组线程等待至某个状态之后再全部同时执行**。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。我们暂且把这个状态就叫做barrier，当调用await()方法之后，线程就处于barrier了。
CyclicBarrier提供2个构造器：

```java
public CyclicBarrier(int parties, Runnable barrierAction) {}
public CyclicBarrier(int parties) {}
```

参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容。

然后CyclicBarrier中最重要的方法就是await方法，它有2个重载版本：

```java
public int await() throws InterruptedException, BrokenBarrierException {}
public int await(long timeout, TimeUnit unit) throws InterruptedException,BrokenBarrierException,
               TimeoutException {}
```

 1. 第一个版本比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；
 2. 第二个版本是让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。
以运动员跑步为例，当裁判的枪声一响，所有的运动员就全部一起跑，这时，啦啦队开始加油！。

```java
public class CyclicBarrierDemo {

    static class Runner implements Runnable {

        private CyclicBarrier cyclicBarrier;
        private String name;

        public Runner(CyclicBarrier cyclicBarrier, String name) {
            this.cyclicBarrier = cyclicBarrier;
            this.name = name;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(1000 * (new Random().nextInt(5)));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("运动员：" + name + "is ready");
            try {
                cyclicBarrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.println("运动员：" + name + "go。。。。");
        }
    }

    /**
     * 当这些线程都达到barrier状态时会执行的内容
     */
    static class Cheering implements Runnable {

        @Override
        public void run() {
            System.out.println("啦啦队：" + "加油加油加油！！！！");
        }
    }

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3, new Cheering());
        Executor executor = Executors.newFixedThreadPool(3);
        Runner runnerA = new Runner(cyclicBarrier, "张三");
        Runner runnerB = new Runner(cyclicBarrier, "李四");
        Runner runnerC = new Runner(cyclicBarrier, "王麻子");
        executor.execute(runnerA);
        executor.execute(runnerB);
        executor.execute(runnerC);
    }

}
```

运行结果如下：
![啦啦队](https://img-blog.csdnimg.cn/2018111112011855.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

## Semaphore

Semaphore翻译成字面意思为 信号量，Semaphore可以控同时访问的线程个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可。
它提供了2个构造器：

```java
public Semaphore(int permits) {}
public Semaphore(int permits, boolean fair) {}
```

参数permits表示许可数目，即同时可以允许多少线程进行访问，参数fair表示是否是公平的，等待时间越久的越先获取许可
下面说一下Semaphore类中比较重要的几个方法，首先是acquire()、release()方法：

```java
//获取一个许可
public void acquire() throws InterruptedException {}
//获取permits个许可
public void acquire(int permits) throws InterruptedException {}
//释放一个许可
public void release() {}
//释放permits个许可
public void release(int permits) {}
```

以上4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

```java
//尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire() { };
//尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };
 //尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits) { };
//尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { };
```

另外还可以通过availablePermits()方法得到可用的许可数目。

假若一个工厂有5台机器，但是有8个工人，一台机器同时只能被一个工人使用，只有使用完了，其他工人才能继续使用。那么我们就可以通过Semaphore来实现：

```java
public class SemaphoreDemo {

    public static void main(String[] args) {
        int N = 8;            //工人数
        Semaphore semaphore = new Semaphore(5); //机器数目
        for(int i=0;i<N;i++)
            new Worker(i,semaphore).start();
    }

    static class Worker extends Thread{
        private int num;
        private Semaphore semaphore;
        public Worker(int num,Semaphore semaphore){
            this.num = num;
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            try {
                semaphore.acquire();
                System.out.println("工人："+this.num+" 占用一个机器在生产...");
                Thread.sleep(2000);
                System.out.println("工人："+this.num+"生产完成，释放出机器");
                semaphore.release();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

运行结果如下：
![工人](https://img-blog.csdnimg.cn/20181111123059876.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)
