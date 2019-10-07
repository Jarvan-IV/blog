---
title: Java线程安全之Lock
date: 2019-9-22 23:45:00
author: xp
categories:
- Java
- 多线程
tags:
- Java
- 多线程
- 线程安全
---

# Java线程安全之Lock

> 在上一篇文章种，简要的介绍了一下synchronized 的使用，在使用synchronized 的代码中，只有等待当前线程执行完，其他的线程才能获取到锁并执行，那么如果多个线程对同一个对象进行读操作，实际上不应该有冲突，但是 synchronized 会导致线程等待并且不可以中断等待（除非线程出现异常）。为了弥补这些问题，Lock横空出世！

Java 5 的 concurrent 包开始提供Lock接口，提供方法主要包括：

 -`void lock()`：获取锁，如果不能成功获取，则等待，与 synchronized 类似
 -`void lockInterruptibly()`：获取锁，如果不能成功获取，则等待，且能够响应中断 t.interrupt()。
 -`boolean tryLock()`：获取锁，如果成功，返回 true，如果不成功，返回 false，不等待
 -`boolean tryLock(long time, TimeUnit unit);`：在一定的时间内，获取锁，如果成功，返回 true，如果不成功，返回 false，不等待
 -`void unlock();`：释放锁
 -`Condition newCondition();`：获取等待通知组件，该组件和当前的锁绑定，当前线程只有获得了锁，才能调用该组件的wait()方法，而调用后，当前线程将释放锁。
接下来主要介绍2类：ReentrantLock和ReadWriteLock

## ReentrantLock

当看到ReentrantLock，可能好奇为什么是再入（重入）？它是表示当一个线程试图获取一个它已经获取的锁时，这个获取的动作就自动成功，这是对锁获取粒度的一个概念，也就是锁的持有是以线程为单位而不是基于调用次数。Java锁实现强调再入行是为了和pthread的行为进行区分。

载入锁可以设置公平性，`Lock lock = new ReentrantLock(true);`这里所谓的公平性是指在竞争场景中，当公平性为真时，会倾向于将锁赋予等待时间最久的线程。公平性是减少线程"饥饿"（个别线程长期等待锁，但始终无法获取）情况发生的一个办法。如果使用synchronized，我们根本无法进行公平性的选择，其永远是不公平的，这也是主流操作系统调度的选择。通用场景中，公平性未必有想象中的那么重要，Java默认的调度策略很少会导致”饥饿“发生。与此同时，若要保证公平性则会引入额外开销，自然会导致吞吐量下降。所以，只有当你的程序确实有公平需要的时候，才有必要指定它。

接下来使用ReentrantLock实现一个简单的生产者消费者模型：

```java
public class ProduceandConsume {
    private static int count = 0;
    private static final int full = 10;
    //创建一个锁对象
    private Lock lock = new ReentrantLock();
    //创建两个条件变量，一个为缓冲区非满，一个为缓冲区非空
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    public static void main(String[] args) {
        ProduceandConsume test2 = new ProduceandConsume();
        new Thread(test2.new Producer()).start();
        new Thread(test2.new Consumer()).start();

        new Thread(test2.new Producer()).start();
        new Thread(test2.new Consumer()).start();

        new Thread(test2.new Producer()).start();
        new Thread(test2.new Consumer()).start();

        new Thread(test2.new Producer()).start();
        new Thread(test2.new Consumer()).start();
    }
    class Producer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(3000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                //获取锁
                lock.lock();
                try {
                    while (count == full) {
                        try {
                            notFull.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    count++;
                    System.out.println(Thread.currentThread().getName()
                            + "生产者生产，目前总共有" + count);
                    //唤醒消费者
                    notEmpty.signal();
                } finally {
                    //释放锁
                    lock.unlock();
                }
            }
        }
    }
    class Consumer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(3000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                lock.lock();
                try {
                    while (count == 0) {
                        try {
                            notEmpty.await();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    count--;
                    System.out.println(Thread.currentThread().getName()
                            + "消费者消费，目前总共有" + count);
                    notFull.signal();
                } finally {
                    lock.unlock();
                }
            }
        }
    }
}
```

相对于synchronized，ReentrantLock有以下好处：

 -等待可中断，持有锁的线程长期不释放的时候，正在等待的线程可以选择放弃等待，这相当于Synchronized来说可以避免出现死锁的情况。
 -公平锁，多个线程等待同一个锁时，必须按照申请锁的时间顺序获得锁，Synchronized锁非公平锁，ReentrantLock默认的构造函数是创建的非公平锁，可以通过参数true设为公平锁，但公平锁表现的性能不是很好。
 -锁绑定多个条件，一个ReentrantLock对象可以同时绑定对个对象。
 -可以响应中断请求。
 -带有超时的获取锁尝试。

## ReadWriteLock

&ensp;&ensp;读写锁分为读锁和写锁，多个读锁之间是不需要互斥的(读操作不会改变数据，如果上了锁，反而会影响效率)，写锁和写锁之间需要互斥，也就是说，如果只是读数据，就可以多个线程同时读，但是如果你要写数据，就必须互斥，使得同一时刻只有一个线程在操作。示例：

```java
class ReadWrite {

    /* 共享数据，只能一个线程写数据，可以多个线程读数据 */
    private Object data = null;
    /* 创建一个读写锁 */
    ReadWriteLock rwlock = new ReentrantReadWriteLock();

    /**
     * 读数据，可以多个线程同时读， 所以上读锁即可
     */
    public void get() {
        /* 上读锁 */
        rwlock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 准备读数据!");
            /* 休眠 */
            Thread.sleep((long) (Math.random() * 1000));
            System.out.println(Thread.currentThread().getName() + "读出的数据为 :" + data);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            rwlock.readLock().unlock();
        }

    }

    /**
     * 写数据，多个线程不能同时 写 所以必须上写锁
     */
    public void put(Object data) {

        /* 上写锁 */
        rwlock.writeLock().lock();

        try {
            System.out.println(Thread.currentThread().getName() + " 准备写数据!");
            /* 休眠 */
            Thread.sleep((long) (Math.random() * 1000));
            this.data = data;
            System.out.println(Thread.currentThread().getName() + " 写入的数据: " + data);

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rwlock.writeLock().unlock();
        }
    }
}

/**
 * 测试类
 *
 * @author Liao
 */
public class ReadWriteLockTest {

    public static void main(String[] args) {

        /* 创建ReadWrite对象 */
        final ReadWrite readWrite = new ReadWrite();

        /* 创建并启动3个读线程 */
        for (int i = 0; i < 3; i++) {
            new Thread(new Runnable() {

                @Override
                public void run() {
                    readWrite.get();

                }
            }).start();

            /*创建3个写线程*/
            new Thread(new Runnable() {

                @Override
                public void run() {
                    /*随机写入一个数*/
                    readWrite.put(new Random().nextInt(8));

                }
            }).start();
        }

    }
}
```

运行结果为：
&ensp;&ensp;Thread-0 准备读数据!
&ensp;&ensp;Thread-2 准备读数据!
&ensp;&ensp;Thread-4 准备读数据!
&ensp;&ensp;Thread-4读出的数据为 :null
&ensp;&ensp;Thread-2读出的数据为 :null
&ensp;&ensp;Thread-0读出的数据为 :null
&ensp;&ensp;Thread-1 准备写数据!
&ensp;&ensp;Thread-1 写入的数据: 2
&ensp;&ensp;Thread-3 准备写数据!
&ensp;&ensp;Thread-3 写入的数据: 5
&ensp;&ensp;Thread-5 准备写数据!
&ensp;&ensp;Thread-5 写入的数据: 2

从第一行到第三行可知，上了读锁之后，有Thread-0  Thread-2 Thread-4 三个线程在同时读数据，三个线程并不是一个执行完才能执行另一个。而进行写操作时，必须等待一个写的线程执行完成之后，另一个线程才能写！ReadLock可以被多个线程持有并且在作用时排斥任何的WriteLock，而WriteLock则是完全的互斥。这一特性最为重要，因为对于高读取频率而相对较低写入的数据结构，使用此类锁同步机制则可以提高并发量。
