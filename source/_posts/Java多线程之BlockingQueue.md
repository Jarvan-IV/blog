---
title: Java多线程之BlockingQueue
date: 2019-9-22 23:55:00
author: xp
categories:
- Java
- 多线程
tags:
- Java
- 多线程
- 线程安全
---

# Java多线程之BlockingQueue

 在JDK1.5新增的Concurrent包中，BlockingQueue很好的解决了多线程中，如何高效安全“传输”数据的问题。通过这些高效并且线程安全的队列类，为我们快速搭建高质量的多线程程序带来极大的便利。本文详细介绍了BlockingQueue家庭中的所有成员，包括他们各自的功能以及常见使用场景。
为什么说是阻塞（Blocking）的呢？是因为 BlockingQueue 支持当获取队列元素但是队列为空时，会阻塞等待队列中有元素再返回；也支持添加元素时，如果队列已满，那么等到队列可以放入新元素时再放入。

![BlockQueque](https://img-blog.csdnimg.cn/20181105214722458.jpg)

|~|抛异常 |特定值|阻塞|超时
|--|--|--|--|--|
| 插入 |add(o)  |offer(o)|put(o)|offer(o,timeout,timeUnit)|
| 移除|remove(o)|poll(o)|take(o)|poll(timeout,timeunit)|
| 检查|element(o)|peek(o)|~|~

四组不同的行为方式解释：

 -抛异常：如果试图的操作无法立即执行，抛一个异常
 -特定值：如果试图的操作无法立即执行，返回一个特定的值（一般是 true/false）
 -阻塞：如果试图的操作无法立即执行，该方法将会发生阻塞，直到能执行
 -超时：如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，但等待时间不会超过给定值。返回一个特定的值以告知该操作是否成功。
BlockingQueue的核心方法：
放入数据：
　　offer(anObject):表示如果可能的话,将anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,
　　　　则返回true,否则返回false.（本方法不阻塞当前执行方法的线程）
　　offer(E o, long timeout, TimeUnit unit),可以设定等待的时间，如果在指定的时间内，还不能往队列中
　　　　加入BlockingQueue，则返回失败。
　　put(anObject):把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断
　　　　直到BlockingQueue里面有空间再继续.
获取数据：
　　poll(time):取走BlockingQueue里排在首位的对象,若不能立即取出,则可以等time参数规定的时间,
　　　　取不到时返回null;
　　poll(long timeout, TimeUnit unit)：从BlockingQueue取出一个队首的对象，如果在指定时间内，
　　　　队列一旦有数据可取，则立即返回队列中的数据。否则知道时间超时还没有数据可取，返回失败。
　　take():取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到
　　　　BlockingQueue有新的数据被加入; 
　　drainTo():一次性从BlockingQueue获取所有可用的数据对象（还可以指定获取数据的个数）， 
　　　　通过该方法，可以提升获取数据效率；不需要多次分批加锁或释放锁。
&ensp;&ensp;无法向一个 BlockingQueue 中插入 null。如果你试图插入 null，BlockingQueue 会抛出一个 NullPointerException。可以访问到 BlockingQueue 中的所有元素，而不仅仅是开始和结束的元素。比如说你将一个对象放入队列之中以等待处理，但你的应用想要将其取消掉，那么你可以调用诸如remove(o)方法啦将队列中的特定对象进行移除。但是这么干相率并不高，因此尽量不要用这一类方法，除非迫不得已。

## ArrayBlockingQueue

基于数组的阻塞队列实现，在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象，这是一个常用的阻塞队列，除了一个定长数组外，ArrayBlockingQueue内部还保存着两个整形变量，分别标识着队列的头部和尾部在数组中的位置。

ArrayBlockingQueue在生产者放入数据和消费者获取数据，都是共用同一个锁对象，由此也意味着两者无法真正并行运行，这点尤其不同于LinkedBlockingQueue；按照实现原理来分析，ArrayBlockingQueue完全可以采用分离锁，从而实现生产者和消费者操作的完全并行运行。Doug Lea之所以没这样去做，也许是因为ArrayBlockingQueue的数据写入和获取操作已经足够轻巧，以至于引入独立的锁机制，除了给代码带来额外的复杂性外，其在性能上完全占不到任何便宜。 ArrayBlockingQueue和LinkedBlockingQueue间还有一个明显的不同之处在于，前者在插入或删除元素时不会产生或销毁任何额外的对象实例，而后者则会生成一个额外的Node对象。这在长时间内需要高效并发地处理大批量数据的系统中，其对于GC的影响还是存在一定的区别。而在创建ArrayBlockingQueue时，我们还可以控制对象的内部锁是否采用公平锁，默认采用非公平锁。
ArrayBlockingQueue内部使用可重入锁ReentrantLock + Condition来完成多线程环境的并发操作，内部主要数据如下：

 -items，一个定长数组，维护ArrayBlockingQueue的元素
 -takeIndex，int，为ArrayBlockingQueue对首位置
 -putIndex，int，ArrayBlockingQueue对尾位置
 -count，元素个数
 -lock，锁，ArrayBlockingQueue出列入列都必须获取该锁，两个步骤公用一个锁
 -notEmpty，出列条件
 -notFull，入列条件

## LinkedBlockingQueue

基于链表的阻塞队列，同ArrayListBlockingQueue类似，其内部也维持着一个数据缓冲队列（该队列由一个链表构成），当生产者往队列中放入一个数据时，队列会从生产者手中获取数据，并缓存在队列内部，而生产者立即返回；只有当队列缓冲区达到最大值缓存容量时（LinkedBlockingQueue可以通过构造函数指定该值），才会阻塞生产者队列，直到消费者从队列中消费掉一份数据，生产者线程会被唤醒，反之对于消费者这端的处理也基于同样的原理。而LinkedBlockingQueue之所以能够高效的处理并发数据，还因为其对于生产者端和消费者端分别采用了独立的锁来控制数据同步，这也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。
下面代码演示使用LinkedBlockingQueue来实现生产者消费者：

```java
public class Consumer implements Runnable {

    private BlockingQueue<String> queue;
    private static final int DEFAULT_RANGE_FOR_SLEEP = 1000;

    public Consumer(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        System.out.println("启动消费者线程！");
        Random r = new Random();
        boolean isRunning = true;
        try {
            while (isRunning) {
                System.out.println("正从队列获取数据...");
                String data = queue.poll(2, TimeUnit.SECONDS);
                if (null != data) {
                    System.out.println("拿到数据：" + data);
                    System.out.println("正在消费数据：" + data);
                    Thread.sleep(r.nextInt(DEFAULT_RANGE_FOR_SLEEP));
                } else {
                    // 超过2s还没数据，认为所有生产线程都已经退出，自动退出消费线程。
                    isRunning = false;
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        } finally {
            System.out.println("退出消费者线程！");
        }
    }
}
```

```java
public class Producer implements Runnable {

    private volatile boolean isRunning = true;
    private BlockingQueue queue;
    private static AtomicInteger count = new AtomicInteger();
    private static final int DEFAULT_RANGE_FOR_SLEEP = 1000;

    public Producer(BlockingQueue queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String data = null;
        Random r = new Random();
        System.out.println("启动生产者线程！");
        try {
            while (isRunning) {
                System.out.println("正在生产数据...");
                Thread.sleep(r.nextInt(DEFAULT_RANGE_FOR_SLEEP));

                data = "data:" + count.incrementAndGet();
                System.out.println("将数据：" + data + "放入队列...");
                if (!queue.offer(data, 2, TimeUnit.SECONDS)) {
                    System.out.println("放入数据失败：" + data);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        } finally {
            System.out.println("退出生产者线程！");
        }
    }

    public void stop() {
        isRunning = false;
    }
}
```

```java
public class Client {
    public static void main(String[] args) throws InterruptedException {
        // 声明一个容量为10的缓存队列
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);

        Producer producer1 = new Producer(queue);
        Producer producer2 = new Producer(queue);
        Producer producer3 = new Producer(queue);

        Consumer consumer = new Consumer(queue);

        // 借助Executors
        ExecutorService service = Executors.newCachedThreadPool();
        // 启动线程
        service.execute(producer1);
        service.execute(producer2);
        service.execute(producer3);
        service.execute(consumer);

       /* // 执行20s
        Thread.sleep(20 * 1000);
        producer1.stop();
        producer2.stop();
        producer3.stop();

        Thread.sleep(2000);
        // 退出Executor
        service.shutdown();*/
    }
}
```

## 延迟队列 DelayQueue

DelayQueue中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。DelayQueue是一个没有大小限制的队列，因此往队列中插入数据的操作（生产者）永远不会被阻塞，而只有获取数据的操作（消费者）才会被阻塞。
使用场景：DelayQueue使用场景较少，但都相当巧妙，常见的例子比如使用一个DelayQueue来管理一个超时未响应的连接队列。

下面以一个网吧上网，时间到自动下机为例：

```java
public class DelayQueueExample {

    public static void main(String[] args) throws InterruptedException {
        DelayQueue<User> queue = new DelayQueue<>();
        User element1 = new User("小明",8000);
        User element2 = new User("小红",2000);
        User element3 = new User("小芳",3000);
        queue.put(element1);
        queue.put(element2);
        queue.put(element3);
        User e = queue.take();
        System.out.println(e.userName + "上网" + e.delayTime + " ms即将下机");
        User e2 = queue.take();
        System.out.println(e.userName + "上网" + e2.delayTime + " ms即将下机");
        User e3 = queue.take();
        System.out.println(e.userName + "上网" + e3.delayTime + " ms即将下机");
    }
}

class User implements Delayed {

    long delayTime;
    long tamp;
    String userName;
    User(String name,long delay) {
        delayTime = delay;
        userName = name;
        tamp = delay + System.currentTimeMillis();
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return tamp - System.currentTimeMillis();
    }

    @Override
    public int compareTo(Delayed o) {
        return tamp - ((User) o).tamp > 0 ? 1 : -1;
    }
}
```

## PriorityBlockingQueue

 基于优先级的阻塞队列（优先级的判断通过构造函数传入的Compator对象来决定），但需要注意的是PriorityBlockingQueue并不会阻塞数据生产者，而只会在没有可消费的数据时，阻塞数据的消费者。因此使用的时候要特别注意，生产者生产数据的速度绝对不能快于消费者消费数据的速度，否则时间一长，会最终耗尽所有的可用堆内存空间。在实现PriorityBlockingQueue时，内部控制线程同步的锁采用的是公平锁。

```java
public class PriorityBlockingQueueTest {

    private static PriorityBlockingQueue<User> queue = new PriorityBlockingQueue<>();

    public static void main(String[] args) {
        queue.add(new User(1, "wu"));
        queue.add(new User(5, "wu5"));
        queue.add(new User(23, "wu23"));
        queue.add(new User(55, "wu55"));
        queue.add(new User(9, "wu9"));
        queue.add(new User(3, "wu3"));
        for (User user : queue) {
            try {
                System.out.println(queue.take().name);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    //静态内部类
    static class User implements Comparable<User> {

        public User(int age, String name) {
            this.age = age;
            this.name = name;
        }

        int age;
        String name;

        @Override
        public int compareTo(User o) {
            return this.age > o.age ? -1 : 1;
        }
    }
}
```

## SynchronousQueue

SynchronousQueue 它的特别之处在于它内部没有容器，一个生产线程，当它生产产品（即put的时候），如果当前没有人想要消费产品(即当前没有线程执行take)，此生产线程必须阻塞，等待一个消费线程调用take操作，take操作将会唤醒该生产线程，同时消费线程会获取生产线程的产品（即数据传递），这样的一个过程称为一次配对过程(当然也可以先take后put,原理是一样的)。

声明一个SynchronousQueue有两种不同的方式，它们之间有着不太一样的行为。公平模式和非公平模式的区别:
如果采用公平模式：SynchronousQueue会采用公平锁，并配合一个FIFO队列来阻塞多余的生产者和消费者，从而体系整体的公平策略；
但如果是非公平模式（SynchronousQueue默认）：SynchronousQueue采用非公平锁，同时配合一个LIFO队列来管理多余的生产者和消费者，而后一种模式，如果生产者和消费者的处理速度有差距，则很容易出现饥渴的情况，即可能有某些生产者或者是消费者的数据永远都得不到处理。

```java
public class SynchronousQueueExample {

    static class SynchronousQueueProducer implements Runnable {

        private BlockingQueue<String> blockingQueue;

        public SynchronousQueueProducer(BlockingQueue<String> queue) {
            this.blockingQueue = queue;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    String data = UUID.randomUUID().toString();
                    System.out.println("Put: " + data);
                    blockingQueue.put(data);
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    static class SynchronousQueueConsumer implements Runnable {

        private BlockingQueue<String> blockingQueue;

        public SynchronousQueueConsumer(BlockingQueue<String> queue) {
            this.blockingQueue = queue;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    String data = blockingQueue.take();
                    System.out.println(Thread.currentThread().getName()
                            + " take(): " + data);
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    public static void main(String[] args) {
        final BlockingQueue<String> synchronousQueue = new SynchronousQueue<>();

        SynchronousQueueProducer queueProducer = new SynchronousQueueProducer(
                synchronousQueue);
        new Thread(queueProducer).start();

        SynchronousQueueProducer queueProducer1 = new SynchronousQueueProducer(
                synchronousQueue);
        new Thread(queueProducer1).start();

        SynchronousQueueConsumer queueConsumer1 = new SynchronousQueueConsumer(
                synchronousQueue);
        new Thread(queueConsumer1).start();

        SynchronousQueueConsumer queueConsumer2 = new SynchronousQueueConsumer(
                synchronousQueue);
        new Thread(queueConsumer2).start();

    }
}
```

运行结果：
&ensp;&ensp;Put: 4e698bbf-f34f-425d-bab8-fc0fbd5eacf8
&ensp;&ensp;Put: c1d04b80-e8ce-43db-bca4-56b6e8153b67
&ensp;&ensp;Thread-2 take(): c1d04b80-e8ce-43db-bca4-56b6e8153b67
&ensp;&ensp;Thread-3 take(): 4e698bbf-f34f-425d-bab8-fc0fbd5eacf8
&ensp;&ensp;Put: 60d1ff9a-7339-4f3f-b669-c2cc3a93105d
&ensp;&ensp;Put: 7b0595b7-54fc-4616-9787-2a22877d24a1
&ensp;&ensp;Thread-3 take(): 7b0595b7-54fc-4616-9787-2a22877d24a1
&ensp;&ensp;Thread-2 take(): 60d1ff9a-7339-4f3f-b669-c2cc3a93105d

从运行结果来看，插入数据的线程和获取数据的线程，交替执行。
