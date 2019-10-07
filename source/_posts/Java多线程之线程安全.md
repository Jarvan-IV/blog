---
title: Java多线程之线程安全
date: 2019-9-22 22:00:00
author: xp
categories:
- Java
- 多线程
tags:
- Java
- 多线程
- 线程安全
---

# Java多线程之线程安全

## 概念

当多个线程访问某一个类（对象或方法时），这个类始终都能表现出正确的行为，那么这个类（对象或方法）就是线程安全的。

## 反例

```java
public class Mythread extends Thread{

    private int count = 5;

    @Override
    public void run() {
        while (count > 0){
            count--;
            System.out.println(Thread.currentThread().getName() + "正在执行：" + count);
        }
    }

    public static void main(String[] args) {
        Mythread mythread = new Mythread();
        Thread t1 = new Thread(mythread,"t1");
        Thread t2 = new Thread(mythread,"t2");
        Thread t3 = new Thread(mythread,"t3");
        Thread t4 = new Thread(mythread,"t4");
        Thread t5 = new Thread(mythread,"t5");
        t1.start();
        t2.start();
        t3.start();
        t4.start();
        t5.start();
    }
}
```

结果如下：
&ensp;&ensp;t1正在执行：4
&ensp;&ensp;t1正在执行：2
&ensp;&ensp;t1正在执行：1
&ensp;&ensp;t1正在执行：0
&ensp;&ensp;t2正在执行：3

> 上述代码中，我们的期望结果是 打印count的值应为： 4 3 2 1 0，与预期结果不相符合，所以Mythread这个类线程不安全，简单来讲，就是有多个线程去操作count这个变量，但是count变量的值并不是我们期望的值，这就引发了线程安全的问题。

## 正例

```java
public class Mythread extends Thread{

    private int count = 5;

    @Override
    public synchronized void run() {
        while (count > 0){
            count--;
            System.out.println(Thread.currentThread().getName() + "正在执行：" + count);
        }
    }

    public static void main(String[] args) {
        Mythread mythread = new Mythread();
        Thread t1 = new Thread(mythread,"t1");
        Thread t2 = new Thread(mythread,"t2");
        Thread t3 = new Thread(mythread,"t3");
        Thread t4 = new Thread(mythread,"t4");
        Thread t5 = new Thread(mythread,"t5");
        t1.start();
        t2.start();
        t3.start();
        t4.start();
        t5.start();
    }
}
```

结果如下：
&ensp;&ensp;t2正在执行：4
&ensp;&ensp;t2正在执行：3
&ensp;&ensp;t2正在执行：2
&ensp;&ensp;t2正在执行：1
&ensp;&ensp;t2正在执行：0

> 可以在任意对象及方法上加锁，而加锁的代码称为“互斥区”或“临界区”，加了synchronized关键字后，&ensp;&ensp;当多个线程访问MyThead的run方法时，以排队的方式进行处理（这里的排队是按照CPU分配的先后顺序而定的），一个线程想要执行synchronized代码体内容，，必须得等上一个线程执行完，才能拿到锁，如果拿不到锁，这个线程就会不断的尝试获得这把锁，直到拿到为止，而且**多个线程同时去竞争这把锁**。

## 总结

 如果想要达到线程安全，大概有以下办法：

 -封装：通过封装，我们可以将对象内部状态隐藏，保护起来。
 -不可变

线程安全需要保证几个基本特征：
 -原子性：简单说就是相关操作不会中途被其他线程干扰，一般通过同步机制实现。
 -可见性：是一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态反映到主内存上。
 -有序性：是保证线程串行语义，避免指令重排等。

能够实现线程安全得方法很多，上面synchronized虽然能保证线程安全，但是会导致锁竞争问题，比如，当有100个线程在同时操作一个变量，当线程1执行完毕后，后面的线程2到线程100，一共99个线程，都会去**争夺这一把所，产生非常激烈的锁竞争问题，这可能会导致服务器的CPU使用率过高**。
