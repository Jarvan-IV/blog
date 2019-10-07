---
title: JVM之内存模型
date: 2018-10-25 22:18:52
author: xp
categories:
- Java
- JVM
tags:
- Java
- JVM
- 内存模型
---

# JVM之内存模型

> Java虚拟机把管理的内存划分为若干不同的数据区域， 由类加载器(classloader) +，执行引擎(execution engine) +，运行时数据区域(runtime data area) 组成。
![Java运行时数据区](https://img-blog.csdnimg.cn/20181025210712619.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

## 程序计算器(PC，Program Counter Register)

&ensp;&ensp;是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在JVM规范中，每个线程都有它自己的程序计数器，并且任何时间一个线程都只有一个方法在执行，也就是所谓的当前方法。程序计数器会存储当前线程正在执行的Java方法的JVM指令地址；或者，如果是在执行本地方法，则是未指定值（undefined）。字节码解释器通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支，循环，跳转，异常处理，线程恢复等基础功能都需要依赖这个计数器来完成。此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域

## Java虚拟机栈(Java Virtual Machine Stack)

&ensp;&ensp;早期也叫Java栈。每个线程在创建时，都会创建一个虚拟机栈，其内部保存一个个的栈帧（StackFrame），用于存储**局部变量表**、操作数栈、帧数据区、动态链接、方法出口等信息。**在一个时间点，对应的只会有一个活动的栈帧，通常叫做当前帧**，方法所在的类叫做当前类。如果在该方法中调用了其他方法，对应的新的栈帧会被创建出来，成为新的当前帧，一直到它返回结果或者执行结束。JVM直接对Java栈的操作只有两个，就是对栈帧的压栈和出栈。

&ensp;&ensp; 局部变量表存放了编译期可知的各种基本数据类型（Boolean，byte，char，short，int，float，long。double）、对象引用和returnAddress（指向了一条字节码指令的地址）。

&ensp;&ensp; 操作数栈主要保存计算过程的中间结果，同时作为计算过程中变量的临时存储空间。

&ensp;&ensp;  除了局部变量表和操作数栈外，栈还需要一些数据来支持常量池的解析，这里帧数据区保存着访问常量池的指针，方便程序访问常量池，另外，当方法返回或者出现异常时，虚拟机必须有一个异常处理表，方便发送异常的时候找到异常的代码，因此异常处理表也是帧数据区的一部分。

在Java虚拟机规范中，对这个区域规定了两种异常情况：

- 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverFlowError异常。比如。不合适的递归方法可能导致此问题。
- 如果虚拟机栈可以动态扩展（当前大部分虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），如果扩展时无法申请到足够的内存， 就会抛出OutOfMemoryError。

## 本地方法栈（Native Method Stack）

&ensp;&ensp;它和Java虚拟机栈是非常相似的，也是线程私有的，虚拟机栈执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。在Oracle HotSpot JVM中，本地方法栈和Java虚拟机栈是在同一块区域，这完全取决与技术实现的决定，并未在规范中强制。也会抛出StackOverFlowError和OutOfMemoryError异常。

## Java堆（Heap）

&ensp;&ensp;它是Java内存管理的核心区域，用来放置Java对象的实例，几乎所有创建的Java对象实例都是被直接分配在堆上。堆被所有的线程共享，在虚拟机启动时，我们指定“Xmx”之类的参数就是用来指定最大堆空间等指标。Java堆是垃圾收集器管理的主要区域，由于现在收集器基本都采用**分代收集算法**，所以Java堆中还可以细分为：新生代（又分，Eden、From survivor，To survivor），老年代。从内存分配的角度来看，线程共享的Java堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB）。

&ensp;&ensp;根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上连续即可。如果堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError。
![Java堆](https://img-blog.csdnimg.cn/20181025224712982.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

## 方法区（Method Area）

&ensp;&ensp;这也是所有线程共享的一块内存区域，用于存储所谓的元数据（Meta Data），例如类结构信息，以及对应的运行时常量池，字段，方法代码等。由于早期的HotSpotJVM实现，很多人习惯于将方法区称为永久代（Permanent Generation）。Oracle JDK8中将永久代移除，同时增加了**元数据区（MetaSpace）**，存储在**本地堆内存**。

&ensp;&ensp;Java虚拟机规范对方法区的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者扩展外，还可以选择**不实现垃圾收集**。垃圾收集行为在这个区域是比较少出现的，并且回收能力总是不尽人意。根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError，常见的场景就是，项目引入的jar包过多，类过多。

## 运行时常量池（Runtime Constant Pool）

&ensp;&ensp;这是方法区的一部分。如果仔细分析过反编译的类文件结构，你能看到版本号、字段、方法、超类、接口等各种信息，还有一项信息就是常量池。Java的常量池可以存放各种常量信息，不管是编译器生成的各种字面量，还是需要在运行时决定的符号引用，所以，它比一般语言的符号表存储的信息更加宽泛。
运行时常量池是方法区的一部分，当无法再申请到内存时，也会抛出OutOfMemoryError。

## 直接内存（Direct Memory）

&ensp;&ensp;直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError 异常出现。在 JDK 1.4 中新加入了 NIO 类，引入了一种基于通道（Channel）与缓冲区（Buffer）的 I/O方式，它可以使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆里的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java 堆和 Native 堆中来回复制数据。

## 对象是在堆上分配的还是栈上？

&ensp;&ensp;随着**JIT**编译期的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化，所有的对象都分配到堆上也渐渐变得不那么“绝对”了，在编译期间，JIT会对代码做很多优化。其中有一部分优化的目的就是减少内存堆分配压力，其中一种重要的技术叫做**逃逸分析**
逃逸分析(Escape Analysis)是目前Java虚拟机中比较前沿的优化技术。这是一种可以有效减少Java 程序中同步负载和内存堆分配压力的跨函数全局数据流分析算法。通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围从而决定是否要将这个对象分配到堆上。逃逸分析的基本行为就是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他地方中，称为方法逃逸。

```java
public static StringBuffer craeteStringBuffer(String s1, String s2) {
   StringBuffer sb = new StringBuffer();
   sb.append(s1);
   sb.append(s2);
   return sb;
}

```

> 上述代码中，StringBuffer sb是一个方法内部变量，上述代码中直接将sb返回，这样这个StringBuffer有可能被其他方法所改变，这样它的作用域就不只是在方法内部，虽然它是一个局部变量，称其逃逸到了方法外部。甚至还有可能被外部线程访问到，譬如赋值给类变量或可以在其他线程中访问的实例变量，称为线程逃逸。

如果想要StringBuffer sb不逃出方法，可以这样写：

```java
public static String createStringBuffer(String s1, String s2) {
   StringBuffer sb = new StringBuffer();
   sb.append(s1);
   sb.append(s2);
   return sb.toString();
}

```

不直接返回 StringBuffer，那么StringBuffer将不会逃逸出方法。换句话说，可以使用逃逸分析将堆分配转化为栈分配。

## JVM常用参数配置

|参数名称| 含义 |默认值|描述
|--|--|--|--|
| -Xms |初始堆大小  |物理内存的1/64(<1GB)| 默认(MinHeapFreeRatio参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制.
| -Xmx   |最大堆大小  |物理内存的1/4(<1GB)| 默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 -Xms的最小限制
|-XX:NewSize   |设置年轻代大小(for 1.3/1.4)||增大年轻代后,将会减小年老代大小.此值对系统性能影响较大,Sun官方推荐配置为整个堆的3/8
|-XX:MaxNewSize   |年轻代最大值(for 1.3/1.4)||
|-XX:PermSize   |设置持久代(perm gen)初始值|物理内存的1/64|1.8及其以后废除|
|-XX:MaxPermSize   |设置持久代最大值|物理内存的1/4|1.8及其以后废除
|-XX:MetaspaceSize   |设置持久代最大值|物理内存的1/4|1.8
|-XX:MaxPermSize   |分配给类元数据空间的初始大小||JDK1.8后，及其以后废除此值为估计值。MetaspaceSize的值设置的过大会延长垃圾回收时间。垃圾回收过后，引起下一次垃圾回收的类元数据空间的大小可能会变大。
|-XX:MaxMetaspaceSize   |分配给类元数据空间的最大值||超过此值就会触发Full GC，此值默认没有限制，但应取决于系统内存的大小。JVM会动态地改变此值。
|-Xss   |每个线程的堆栈大小||JDK5.0以后每个线程堆栈大小为1M,以前每个线程堆栈大小为256K.更具应用的线程所需内存大小进行 调整.在相同物理内存下,减小这个值能生成更多的线程.但是操作系统对一个进程内的线程数还是有限制的,不能无限生成,经验值在3000~5000左右一般小的应用， 如果栈不是很深， 应该是128k够用的 大的应用建议使用256k。
|-XX:ThreadStackSize  |Thread Stack Size||线程堆栈大小
|-XX:NewRatio  |年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代)||-XX:NewRatio=4表示年轻代与年老代所占比值为1:4,年轻代占整个堆栈的1/5Xms=Xmx并且设置了Xmn的情况下，该参数不需要进行设置。
|-XX:SurvivorRatio  |Eden区与Survivor区的大小比值||设置为8,则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10
|-XX:LargePageSizeInBytes  |设置堆的内存页大小。|默认4m，amd64位：2m|内存页的大小不可设置过大， 会影响Perm的大小
|-XX:+UseFastAccessorMethods  |原始类型的快速优化|默认启用。|启用原始类型的getter方法优化。
|-XX:-DisableExplicitGC |关闭System.gc()|默认不启用|禁止在运行期显式地调用 System.gc()。开启该选项后，GC的触发时机将由GarbageCollector全权掌控。
|-XX:MaxTenuringThreshold  |垃圾最大年龄||如果设置为0的话,则年轻代对象不经过Survivor区,直接进入年老代. 对于年老代比较多的应用,可以提高效率.如果将此值设置为一个较大值,则年轻代对象会在Survivor区进行多次复制,这样可以增加对象再年轻代的存活 时间,增加在年轻代即被回收的概率该参数只有在串行GC时才有效。
|-XX:+AggressiveOpts |加快编译。|JDK6默认启用。| 启用JVM开发团队最新的调优成果。例如编译优化，偏向锁，并行年老代收集等。
|-XX:+UseBiasedLocking |启用偏向锁。|JDK6默认启用。| 锁机制的性能改善
|-Xnoclassgc |禁用垃圾回收。||
|-XX:SoftRefLRUPolicyMSPerMB |每兆堆空闲空间中SoftReference的存活时间。|1s| softly reachable objects will remain alive for some amount of time after the last time they were referenced. The default value is one second of lifetime per free megabyte in the heap
|-XX:PretenureSizeThreshold|对象超过多大是直接在旧生代分配。|0| 单位字节 新生代采用Parallel Scavenge GC时无效另一种直接在旧生代分配的情况是大的数组对象,且数组中无外部引用对象。
|-XX:TLABWasteTargetPercent |TLAB占eden区的百分比。|1%|
|-XX:+CollectGen0First|FullGC时是否先YGC。|false|
