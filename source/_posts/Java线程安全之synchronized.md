---
title: Java线程安全之synchronized
date: 2019-9-22 22:10:00
author: xp
categories:
- Java
- 多线程
tags:
- Java
- 多线程
- 线程安全
---

# Java线程安全之synchronized

## 简介

synchronized是Java内建得同步机制，所以也有人称其为Intrinsic Locking，它提供了互斥的语义和可见性，当一个线程已经获取到当前锁时，其他试图获取的线程只能等待或者阻塞在那儿。
在Java5以前，synchronized是仅有的同步手段，在代码中，synchronized可以用来修饰方法，也可以使用在特定的代码块上，本质上synchronized方法等同于把方法全部语句synchronized块包起来。
synchronized是Java中的关键字，是一种同步锁，Monitor是 Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 Class的锁。每一个对象都有，也仅有一个 Monitor。它修饰的对象（Monitor）有以下几种：

 -修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；
 -修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象；
 -修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象；
 -修改一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象。

## 作用对象

### 修饰一个代码块

一个线程访问一个对象中的synchronized(this)同步代码块时，其他试图访问该对象的线程将被阻塞。我们看下面一个 Demo1：

```java
public class SyncMethod implements Runnable {

    private static int count;

    @Override
    public void run() {
        synchronized (this) {
            for (int i = 0; i < 5; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ":" + (count++));
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

编写一个运行测试类Client：

```java
public class CLient {
    public static void main(String[] args) {
        SyncMethod syncMethod = new SyncMethod();
        Thread thread1 = new Thread(syncMethod, "SyncMethod1");
        Thread thread2 = new Thread(syncMethod, "SyncMethod2");
        thread1.start();
        thread2.start();
    }
}
```

运行结果如下：
&ensp;&ensp;&ensp;&ensp;SyncMethod1:0
&ensp;&ensp;&ensp;&ensp;SyncMethod1:1
&ensp;&ensp;&ensp;&ensp;SyncMethod1:2
&ensp;&ensp;&ensp;&ensp;SyncMethod1:3
&ensp;&ensp;&ensp;&ensp;SyncMethod1:4
&ensp;&ensp;&ensp;&ensp;SyncMethod2:5
&ensp;&ensp;&ensp;&ensp;SyncMethod2:6
&ensp;&ensp;&ensp;&ensp;SyncMethod2:7
&ensp;&ensp;&ensp;&ensp;SyncMethod2:8
&ensp;&ensp;&ensp;&ensp;SyncMethod2:9

当两个并发线程(thread1和thread2)访问同一个对象(SyncMethod )中的synchronized代码块时，在同一时刻只能有一个线程得到锁并执行，另一个线程受阻塞，必须等待当前线程执行完这个代码块以后才能执行该代码块。thread1和thread2是互斥的，因为在执行synchronized代码块时，**Monitor（监视器对象）是同一个**，只有执行完该代码块才能释放该对象锁，下一个线程才能得到锁并且执行代码。
接下来我们把Client类的代码稍加修改：

```java
public class CLient {
    public static void main(String[] args) {
        SyncMethod syncMethod1 = new SyncMethod();
        SyncMethod syncMethod2 = new SyncMethod();
        Thread thread1 = new Thread(syncMethod1, "SyncMethod1");
        Thread thread2 = new Thread(syncMethod2, "SyncMethod2");
        thread1.start();
        thread2.start();
    }
}
```

运行结果如下：
&ensp;&ensp;&ensp;&ensp;SyncMethod1:0
&ensp;&ensp;&ensp;&ensp;SyncMethod2:0
&ensp;&ensp;&ensp;&ensp;SyncMethod2:1
&ensp;&ensp;&ensp;&ensp;SyncMethod1:1
&ensp;&ensp;&ensp;&ensp;SyncMethod2:2
&ensp;&ensp;&ensp;&ensp;SyncMethod1:3
&ensp;&ensp;&ensp;&ensp;SyncMethod1:5
&ensp;&ensp;&ensp;&ensp;SyncMethod2:4
&ensp;&ensp;&ensp;&ensp;SyncMethod1:6
&ensp;&ensp;&ensp;&ensp;SyncMethod2:7
出现以上结果的原因就是，创建了两个SyncMethod的对象syncMethod1 和syncMethod2 ；我们知道synchronized的监视器是对象，这时会有两把锁分别锁定syncMethod1 对象和syncMethod2 对象，而这两把锁是互不干扰的，不形成互斥，简而言之，**synchronized的监视器对象不是同一个**，所以两个线程可以同时执行。

### 修饰一个方法

synchronized修饰一个方法很简单，就是在方法返回值的前面加synchronized，synchronized修饰方法和修饰一个代码块类似，只是作用范围不一样，修饰代码块是大括号括起来的范围，而修饰方法范围是整个方法。如将【Demo1】中的run方法改成如下的方式，实现的效果一样。

```java
public class SyncMethod implements Runnable {

    private static int count;

    @Override
    public synchronized void run() {
        for (int i = 0; i < 5; i++) {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + (count++));
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

实现的效果和Demo1是一样的。
在用synchronized修饰方法时要注意以下几点：
 -synchronized关键字不能继承。
 -在定义接口方法时不能使用synchronized关键字。
 -构造方法不能使用synchronized关键字，但可以使用synchronized代码块来进行同步。

### 修饰一个静态的方法

我们知道静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象，代码如下：

```java
public class SyncMethod implements Runnable {

    private static int count;

    public synchronized static void method() {
        for (int i = 0; i < 5; i++) {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + (count++));
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public synchronized void run() {
        method();
    }
}
```

&ensp;&ensp;编写运行测试类 Client：

```java
public class CLient {
    public static void main(String[] args) {
        SyncMethod syncMethod1 = new SyncMethod();
        SyncMethod syncMethod2 = new SyncMethod();
        Thread thread1 = new Thread(syncMethod1, "SyncMethod1");
        Thread thread2 = new Thread(syncMethod2, "SyncMethod2");
        thread1.start();
        thread2.start();
    }
}
```

结果如下：
&ensp;&ensp;&ensp;&ensp;SyncMethod1:0
&ensp;&ensp;&ensp;&ensp;SyncMethod1:1
&ensp;&ensp;&ensp;&ensp;SyncMethod1:2
&ensp;&ensp;&ensp;&ensp;SyncMethod1:3
&ensp;&ensp;&ensp;&ensp;SyncMethod1:4
&ensp;&ensp;&ensp;&ensp;SyncMethod2:5
&ensp;&ensp;&ensp;&ensp;SyncMethod2:6
&ensp;&ensp;&ensp;&ensp;SyncMethod2:7
&ensp;&ensp;&ensp;&ensp;SyncMethod2:8
&ensp;&ensp;&ensp;&ensp;SyncMethod2:9

syncMethod1 和syncMethod1 是SyncMethod的两个对象，但在thread1和thread2并发执行时却保持了线程同步。这是因为method方法中synchronized 修饰的静态方法，静态方法是属于类的，所以锁的Monitor（监视器）是Class字节码，所以这两个线程用了同一把锁。这与Demo1是不同的。

### 修饰一个普通的方法

这和修饰一个静态方法的Monitor级别是一样的，运行结果也是一样的，代码如下：

```java
public class SyncMethod implements Runnable {

    private static int count;

    @Override
    public synchronized void run() {
        synchronized (SyncMethod.class) {
            for (int i = 0; i < 5; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ":" + (count++));
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 总结

现代的（Oracle）JDK中，JVM对此进行了大刀阔斧的改进，提供了三种不同的Monitor实现，也就是常说的三种不同的锁：偏斜锁（Biased Locking）、轻量级锁和重量级锁，大大的改进了其性能。
所谓锁的升级、降级，就是JVM优化synchronized 运行的机制，当JVM检测到不同的竞争状况时，会自动切换到合适的锁实现，这种切换就是锁的升级、降级。如果线程执行发生异常，此时JVM会让线程自动释放锁。
从性能的角度，synchronized早期的实现比较低效，对比ReentrantLock，大多数场景性能都相差较大。但是在Java  6 种对其进行了非常多的改进，在高竞争情况下，ReentrantLock仍然有一定优势。在大多数情况下，无需纠结于性能，还是考虑代码书写结构的便利性、可维护行，可读性等。
