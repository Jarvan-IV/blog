---
title: JVM之垃圾回收
date: 2018-10-27 20:51:48
author: xp
categories:
- Java
- JVM
tags:
- Java
- JVM
- 垃圾回收
---

# JVM之垃圾回收

> Java 语言的一大特点就是可以进行自动垃圾回收处理，而无需开发人员过于关注系统资源，例如内存资源的释放情况。自动垃圾收集虽然大大减轻了开发人员的工作量，但是也增加了软件系统的负担。拥有垃圾收集器可以说是 Java 语言与 C++语言的一项显著区别。在 C++语言中，程序员必须小心谨慎地处理每一项内存分配，且内存使用完后必须手工释放曾经占用的内存空间。当内存释放不够完全时，即存在分配但永不释放的内存块，就会引起内存泄漏，严重时甚至导致程序瘫痪。

&ensp;&ensp;在上一篇博文中，我大概介绍了JVM的内存模型，其中**程序计数器，虚拟机栈，本地方法栈**3个区域随线程而生灭，当方法结束或者线程结束时，内存自然就回收了，所以这3个区域就不需要过多考虑回收的问题。

## 如何判断对象已死

&ensp;&ensp;在堆里面存放着Java世界中几乎所有的对象实例，垃圾收集器在对堆进行回收前，第一件事情就是要确定这些对象之中哪些还活着或者死亡。有下面2种方法判断：

- 引用计数法
&ensp;&ensp;引用计数器的实现很简单，对于一个对象 A，只要有任何一个对象引用了 A，则 A 的引用计数器就加 1，当引用失效时，引用计数器就减 1。只要对象 A 的引用计数器的值为 0，则对象 A 就不可能再被使用。引用计数器的实现也非常简单，只需要为每个对象配置一个整形的计数器即可。
&ensp;&ensp;但是引用计数器有一个严重的问题，即无法处理循环引用的情况。因此，在 Java 的垃圾回收器中**没有使用这种算法**。一个简单的循环引用问题描述如下：有对象 A 和对象 B，对象 A 中含有对象 B 的引用，对象 B 中含有对象 A 的引用。此时，对象 A 和对象 B 的引用计数器都不为 0。但是在系统中却不存在任何第 3 个对象引用了 A 或 B。也就是说，A 和 B 是应该被回收的垃圾对象，但由于垃圾对象间相互引用，从而使垃圾回收器无法识别，引起内存泄漏。例如，如下代码就不会GC：

```java
public class Test {

    public Object instance = null;

    public static void main(String[] args) {
        Test object1 = new Test();
        Test object2 = new Test();

        object1.instance = object2;
        object2.instance = object1;

        object1 = null;
        object2 = null;
        System.gc();
    }
}
```

- 可达性分析（根搜索法）
&ensp;&ensp;为了解决上面的循环引用问题，Java采用了一种新的算法：可达性分析算法。
从GC Roots（每种具体实现对GC Roots有不同的定义）作为起点，向下搜索它们引用的对象，可以生成一棵引用树，树的节点视为可达对象，反之视为不可达。

![可达性分析算法](https://img-blog.csdnimg.cn/20181027164313889.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

在Java语言中，可作为GCRoots的对象包括下面几种：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象。
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI（即一般说的Native方法）引用的对象

## 回收方法区

&ensp;&ensp;很多人认为方法区（或者HotSpot虚拟机中的永久代）是没有垃圾回收的，Java虚拟机规范中确实说过可以不要求虚拟机在方法区实现垃圾收集，而且在方法区中进行垃圾收集的性价比一般比较低。
&ensp;&ensp;永久代的垃圾收集主要回收两部分内容：**废弃常量和无用的类**。

> 由于 PermGen 内存经常会溢出，所以在JDK1.8中，PermGen 最终被移除，方法区移至 Metaspace，字符串常量移至 Java Heap。JDK 8 开始把类的元数据放到本地堆内存(native heap)中，这一块区域就叫 Metaspace，中文名叫元空间。默认的类的元数据分配只受本地内存大小的限制

## 垃圾收集算法

### 标记-清除算法 (Mark-Sweep)

&ensp;&ensp;标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。一种可行的实现是，在标记阶段首先通过根节点，标记所有从根节点开始的较大对象。因此，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。该算法最大的问题是存在大量的空间碎片，因为回收后的空间是不连续的。在对象的堆空间分配过程中，尤其是大对象的内存分配，不连续的内存空间的工作效率要低于连续的空间。

### 复制算法 (Copying)

 &ensp;&ensp;将现有的内存空间分为两快，每次只使用其中一块，在垃圾回收时将正在使用的内存中的存活对象复制到未被使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。
 &ensp;&ensp;如果系统中的垃圾对象很多，复制算法需要复制的存活对象数量并不会太大。因此在真正需要垃圾回收的时刻，复制算法的效率是很高的。又由于对象在垃圾回收过程中统一被复制到新的内存空间中，因此，可确保回收后的内存空间是没有碎片的。该算法的缺点是将系统内存折半。
 &ensp;&ensp;Java 的新生代串行垃圾回收器中使用了复制算法的思想。例如堆中的新生代分为 eden 空间、from 空间、to 空间 3 个部分。其中 from 空间和 to 空间可以视为用于复制的两块大小相同、地位相等，且可进行角色互换的空间块。from 和 to 空间也称为 survivor 空间，即幸存者空间，用于存放未被回收的对象。
![Java堆](https://img-blog.csdnimg.cn/20181027170934985.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)
&ensp;&ensp;在垃圾回收时，eden 空间中的存活对象会被复制到未使用的 survivor 空间中 (假设是 to)，正在使用的 survivor 空间 (假设是 from) 中的年轻对象也会被复制到 to 空间中 (大对象，或者老年对象会直接进入老年带，如果 to 空间已满，则对象也会直接进入老年代)。此时，eden 空间和 from 空间中的剩余对象就是垃圾对象，可以直接清空，to 空间则存放此次回收后的存活对象。这种改进的复制算法既保证了空间的连续性，又避免了大量的内存空间浪费。

### 标记-压缩算法 (Mark-Compact)

&ensp;&ensp;复制算法的高效性是建立在存活对象少、垃圾对象多的前提下的。这种情况在年轻代经常发生，但是在老年代更常见的情况是大部分对象都是存活对象。如果依然使用复制算法，由于存活的对象较多，复制的成本也将很高。
&ensp;&ensp;标记-压缩算法是一种老年代的回收算法，它在标记-清除算法的基础上做了一些优化。也首先需要从根节点开始对所有可达对象做一次标记，但之后，它并不简单地清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。这种方法既避免了碎片的产生，又不需要两块相同的内存空间，因此，其性价比比较高。

### 分代收集算法 (Generational Collecting)

&ensp;&ensp;根据垃圾回收对象的特性，不同阶段最优的方式是使用合适的算法用于本阶段的垃圾回收，分代算法即是基于这种思想，它将内存区间根据对象的特点分成几块，根据每块内存区间的特点，使用不同的回收算法，以提高垃圾回收的效率。以 Hot Spot 虚拟机为例，它将所有的新建对象都放入称为年轻代的内存区域，年轻代的特点是对象会很快回收，因此，在年轻代就选择效率较高的复制算法。当一个对象经过几次回收后依然存活，对象就会被放入称为老生代的内存空间。在老生代中，几乎所有的对象都是经过几次垃圾回收后依然得以幸存的。因此，可以认为这些对象在一段时期内，甚至在应用程序的整个生命周期中，将是常驻内存的。所以根据分代的思想，可以对老年代的回收使用与新生代不同的标记-压缩算法，以提高垃圾回收效率。

## 垃圾收集过程

- 第一，Java应用不断创建对象，通常都是分配在Eden区域，当其空间占用到达一定阈值时，触发Minor GC。仍然被引用的对象（存活的对象，绿色方块），被**复制**到JVM选择的**Survivor 区域其中的from（下图中的S0）**，复制到S0之后，Eden区域可以直接回收，此时Eden的对象就为已经存活的对象（已被复制到S0）和未被引用的对象（即将回收的对象），Eden整个区域就可以直接清除。注意，我给存活对象标记了“数字 1”，这是为了表明对象的存活时间
![from区域](https://img-blog.csdnimg.cn/20181027214024606.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

- 第二，经过一次Minor GC，Eden区域就会空闲，直到再次达到Minor GC 触发条件，这时候，另一个**空闲的Survivor 区域**则只有to（下图中的S1）区域，**Eden区域和From区域存活的对象，都会被复制到to区域**，并且存活的年龄计数会被加1。
![to区域](https://img-blog.csdnimg.cn/20181027214316751.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)
- 第三，类似地尔不得过程会发生很多次，知道有对象年龄计数达到阈值，这时候就会发生所谓的晋升（Promotion），如下图所示，超过阈值的对象会被晋升到老年代。
![老年代](https://img-blog.csdnimg.cn/20181027214530317.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

这个阈值是可以通过参数指定：

```xml
-XX:MaxTenuringThreshold
```

- 后面就是老年代GC，具体取决于选择的GC选项，对应不同的算法。下面是一个简单标记-压缩算法过程示例图，老年代中的无用对象被清除后，GC会将对象进行整理，以防止内存碎片化。

![标记压缩](https://img-blog.csdnimg.cn/2018102721593985.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)
> 通常我们把老年代GC 叫做 Major GC，将对整个堆进行清理的GC 叫做 Full GC ，但是这个也没有那么绝对，因为不同的老年代GC算法差异很大，例如，CMS ”concurrent“就体现在清理工作是与工作线程一起并发运行的。

## 垃圾收集器

&ensp;&ensp;上面讲述了4种垃圾收集算法，但这仅仅是理论，那么垃圾收集器就是对这些理论的具体实现。Java虚拟机规范中对垃圾收集器应该如何实现并没有任务规定，因此不同的厂商，不同版本的虚拟机所提供的垃圾收集器都可能会有很大差别，并且一般都会提供参数供用户自己的应用特点和要求组合出各个年代（区域）所使用的收集器。
&ensp;&ensp;从不同角度分析垃圾收集器，可以将其分为不同的类型。

- 按线程数分，可以分为串行垃圾回收器和并行垃圾回收器。串行垃圾回收器一次只使用一个线程进行垃圾回收；并行垃圾回收器一次将开启多个线程同时进行垃圾回收。在并行能力较强的 CPU 上，使用并行垃圾回收器可以缩短 GC 的停顿时间。

- 按照工作模式分，可以分为并发式垃圾回收器和独占式垃圾回收器。并发式垃圾回收器与应用程序线程交替工作，以尽可能减少应用程序的停顿时间；独占式垃圾回收器 (Stop the world) 一旦运行，就停止应用程序中的其他所有线程，直到垃圾回收过程完全结束。
- 按碎片处理方式可分为压缩式垃圾回收器和非压缩式垃圾回收器。压缩式垃圾回收器会在回收完成后，对存活对象进行压缩整理，消除回收后的碎片；非压缩式的垃圾回收器不进行这步操作。
- 按工作的内存区间，又可分为新生代垃圾回收器和老年代垃圾回收器。
![HotSpot虚拟机垃圾收集器](https://img-blog.csdnimg.cn/20181027194216145.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

### Serial收集器

&ensp;&ensp;&ensp;&ensp;&ensp;&ensp; 有两个特点：第一，它仅仅使用单线程进行垃圾回收；第二，它独占式的垃圾回收。使用-XX:+UseSerialGC参数指定使用串行回收器，Jvm运行在Client模式下的默认值，使用Serial + Serial Old的收集器组合进行内存回收。
![Serial收集器](https://img-blog.csdnimg.cn/20181027180905609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)
Serial进行垃圾收集时，必须暂停其他所有的工作线程，直到它收集结束。

### Serial Old收集器

&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;它是Serial收集器的老年代版本，和新生代串行收集器一样，它也是一个串行的、独占式的垃圾回收器。它可以作为CMS收集器的后被预案，在并发收集发生 Concurrent Model Failure时使用。
&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;使用以下参数：-XX:+UseSerialGC: 新生代、老年代都使用串行回收器。

### ParNew收集器

&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;它仅仅是Serial收集器的多线程版本。并行回收器也是独占式的回收器，在收集过程中，应用程序会全部暂停。但由于并行回收器使用多线程进行垃圾回收，因此，在并发能力比较强的 CPU 上，它产生的停顿时间要短于串行回收器，而在单 CPU 或者并发能力较弱的系统中，并行回收器的效果不会比串行回收器好，由于多线程的压力，它的实际表现很可能比串行回收器差。
&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;开启并行回收器可以使用参数-XX:+UseParNewGC，该参数设置新生代使用并行收集器，老年代也使用串行收集器。
![ParNew/SerialOld收集器](https://img-blog.csdnimg.cn/20181027181628220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)
&ensp;&ensp;设置参数-XX:+UseConcMarkSweepGC 可以要求新生代使用并行收集器，老年代使用 CMS。
&ensp;&ensp;并行收集器工作时的线程数量可以使用-XX:ParallelGCThreads 参数指定。一般，最好与 CPU 数量相当，避免过多的线程数影响垃圾收集性能。在默认情况下，当 CPU 数量小于 8 个，ParallelGCThreads 的值等于 CPU 数量，大于 8 个，ParallelGCThreads 的值等于 3+[5*CPU_Count]/8]。以下测试显示了笔者笔记本上运行 8 个线程时耗时最短，本人笔记本是 8 核 IntelCPU。

### Parallel Scavenge收集器

&ensp;&ensp;&ensp;&ensp;&ensp;  该收集器是一个新生代收集器，使用复制算法。从表面上看，它和并行收集器一样都是多线程、独占式的收集器。但是，并行回收收集器有一个重要的特点：它非常关注系统的吞吐量。
&ensp;&ensp;&ensp;&ensp;&ensp;  -XX:+UseParallelGC:新生代使用**并行**回收收集器，老年代使用串行收集器。

> &ensp;&ensp;所谓的吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值，即 吞吐量 = 运行用户代码时间/（运行用户代码时间+垃圾收集时间），例如，虚拟机总共运行了100分钟，其中垃圾收集花掉了1分钟，那么运行用户代码时间就花费了99分钟，所以，吞吐量就是99％
&ensp;&ensp;在其启动时，可以指定如下参数：
&ensp;&ensp;&ensp;&ensp;-XX:+MaxGCPauseMills:设置最大垃圾收集停顿时间，它的值是一个大于 0 的整数。收集器在工作时会调整 Java 堆大小或者其他一些参数，尽可能地把停顿时间控制在 MaxGCPauseMills 以内。如果希望减少停顿时间，而把这个值设置得很小，为了达到预期的停顿时间，JVM 可能会使用一个较小的堆 (一个小堆比一个大堆回收快)，而这将导致垃圾回收变得很频繁，从而增加了垃圾回收总时间，降低了吞吐量。
&ensp;&ensp;&ensp;&ensp;-XX:+GCTimeRatio：设置吞吐量大小，它的值是一个 0-100 之间的整数。假设 GCTimeRatio 的值为 n，那么系统将花费不超过 1/(1+n) 的时间用于垃圾收集。比如 GCTimeRatio 等于 19，则系统用于垃圾收集的时间不超过 1/(1+19)=5%。默认情况下，它的取值是 99，即不超过 1%的时间用于垃圾收集。
&ensp;&ensp;&ensp;&ensp;除此之外，并行回收收集器与并行收集器另一个不同之处在于，它支持一种自适应的 GC 调节策略，使用-XX:+UseAdaptiveSizePolicy 可以打开自适应 GC 策略。在这种模式下，新生代的大小、eden 和 survivor 的比例、晋升老年代的对象年龄等参数会被自动调整，以达到在堆大小、吞吐量和停顿时间之间的平衡点。在手工调优比较困难的场合，可以直接使用这种自适应的方式，仅指定虚拟机的最大堆、目标的吞吐量 (GCTimeRatio) 和停顿时间 (MaxGCPauseMills)，让虚拟机自己完成调优工作。

### Parallel Old

&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;它是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。
&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;-XX:+UseParallelOldGC:新生代和老年代都是用**并行**回收收集器。
&ensp;&ensp;&ensp;&ensp; &ensp;&ensp;参数-XX:ParallelGCThreads 也可以用于设置垃圾回收时的线程数量。

### CMS收集器

&ensp;&ensp;与并行回收收集器不同，CMS 收集器主要关注于系统停顿时间。CMS 是 Concurrent Mark Sweep 的缩写，意为并发标记清除，从名称上可以得知，它使用的是标记-清除算法，同时它又是一个使用多线程并发回收的垃圾收集器。
&ensp;&ensp;CMS 工作时，主要步骤有：初始标记、并发标记、重新标记、并发清除和并发重置。其中初始标记和重新标记是独占系统资源的，而并发标记、并发清除和并发重置是可以和用户线程一起执行的。因此，从整体上来说，CMS 收集不是独占式的，它可以在应用程序运行过程中进行垃圾回收。
&ensp;&ensp;根据标记-清除算法，初始标记、并发标记和重新标记都是为了标记出需要回收的对象。并发清理则是在标记完成后，正式回收垃圾对象；并发重置是指在垃圾回收完成后，重新初始化 CMS 数据结构和数据，为下一次垃圾回收做好准备。并发标记、并发清理和并发重置都是可以和应用程序线程一起执行的。
&ensp;&ensp;CMS 收集器在其主要的工作阶段虽然没有暴力地彻底暂停应用程序线程，但是由于它和应用程序线程并发执行，相互抢占 CPU，所以在 CMS 执行期内对应用程序吞吐量造成一定影响。CMS 默认启动的线程数是 (ParallelGCThreads+3)/4),ParallelGCThreads 是新生代并行收集器的线程数，也可以通过-XX:ParallelCMSThreads 参数手工设定 CMS 的线程数量。当 CPU 资源比较紧张时，受到 CMS 收集器线程的影响，应用程序的性能在垃圾回收阶段可能会非常糟糕。
&ensp;&ensp;由于 CMS 收集器不是独占式的回收器，在 CMS 回收过程中，应用程序仍然在不停地工作。在应用程序工作过程中，又会不断地产生垃圾。这些新生成的垃圾在当前 CMS 回收过程中是无法清除的。同时，因为应用程序没有中断，所以在 CMS 回收过程中，还应该确保应用程序有足够的内存可用。因此，CMS 收集器不会等待堆内存饱和时才进行垃圾回收，而是当前堆内存使用率达到某一阈值时，便开始进行回收，以确保应用程序在 CMS 工作过程中依然有足够的空间支持应用程序运行。
&ensp;&ensp;这个回收阈值可以使用-XX:CMSInitiatingOccupancyFraction 来指定，默认是 68。即当老年代的空间使用率达到 68%时，会执行一次 CMS 回收。如果应用程序的内存使用率增长很快，在 CMS 的执行过程中，已经出现了内存不足的情况，此时，CMS 回收将会失败，JVM 将启动老年代串行收集器进行垃圾回收。如果这样，应用程序将完全中断，直到垃圾收集完成，这时，应用程序的停顿时间可能很长。因此，根据应用程序的特点，可以对-XX:CMSInitiatingOccupancyFraction 进行调优。如果内存增长缓慢，则可以设置一个稍大的值，大的阈值可以有效降低 CMS 的触发频率，减少老年代回收的次数可以较为明显地改善应用程序性能。反之，如果应用程序内存使用率增长很快，则应该降低这个阈值，以避免频繁触发老年代串行收集器。
&ensp;&ensp;标记-清除算法将会造成大量内存碎片，离散的可用空间无法分配较大的对象。在这种情况下，即使堆内存仍然有较大的剩余空间，也可能会被迫进行一次垃圾回收，以换取一块可用的连续内存，这种现象对系统性能是相当不利的，为了解决这个问题，CMS 收集器还提供了几个用于内存压缩整理的算法。
&ensp;&ensp;-XX:+UseCMSCompactAtFullCollection 参数可以使 CMS 在垃圾收集完成后，进行一次内存碎片整理。内存碎片的整理并不是并发进行的。-XX:CMSFullGCsBeforeCompaction 参数可以用于设定进行多少次 CMS 回收后，进行一次内存压缩。

### G1 收集器 (Garbage First)

&ensp;&ensp;G1 收集器的目标是作为一款服务器的垃圾收集器，因此，它在吞吐量和停顿控制上，预期要优于 CMS 收集器。
&ensp;&ensp;与 CMS 收集器相比，G1 收集器是基于标记-压缩算法的。因此，它不会产生空间碎片，也没有必要在收集完成后，进行一次独占式的碎片整理工作。G1 收集器还可以进行非常精确的停顿控制。它可以让开发人员指定当停顿时长为 M 时，垃圾回收时间不超过 N。使用参数-XX:+UnlockExperimentalVMOptions –XX:+UseG1GC 来启用 G1 回收器，设置 G1 回收器的目标停顿时间：-XX:MaxGCPauseMills=20,-XX:GCPauseIntervalMills=200。

**比较不幸的是CMS GC**，因为其算法的理论缺陷等原因，虽然现在还有非常大的用户群体，到那时已经被标记为废弃，如果没有组织主动承担CMS的维护，很有可能在未来版本移除。如果你有关注目前尚处于开发中的JDK11，你会发现，JDK又增加了两种全新的GC方式，分别是：

- Epsilon GC：简单说就是个不做垃圾收集的GC，似乎有点奇怪，有的情况下，例如在进行性能测试的时候，可能需要明确判断GC本身产生了多大的开销，这就是其典型的应用场景
- ZGC：这是Oracle 开源出来的一个超级GC实现，具备令人惊讶的扩展能力，比如支持T byte级别的堆大小，并且保证绝大部分情况下，延迟都不会超过10 ms。虽然目前还处于实验阶段，仅支持Linux64位的平台，但其已经表现出的能力和潜力都非常令人期待。

> 垃圾回收的使用没有那么绝对，调优永远是针对 特定场景、特定需求、不存在一劳永逸的指标，一般建议堆30G以上慎用CMS，Cassandra的官方指南建议用在16G以内。

## GC 相关参数总结

#### 与串行回收器相关的参数

- -XX:+UseSerialGC:在新生代和老年代使用串行回收器。
- -XX:+SurvivorRatio:设置 eden 区大小和 survivor 区大小的比例。
- -XX:+PretenureSizeThreshold:设置大对象直接进入老年代的阈值。当对象的大小超过这个值时，将直接在老年代分配。
- -XX:MaxTenuringThreshold:设置对象进入老年代的年龄的最大值。每一次 Minor GC 后，对象年龄就加 1。任何大于这个年龄的对象，一定会进入老年代。

#### &ensp;&ensp;与并行 GC 相关的参数

- -XX:+UseParNewGC: 在新生代使用并行收集器。
- -XX:+UseParallelOldGC: 老年代使用并行回收收集器。
- -XX:ParallelGCThreads：设置用于垃圾回收的线程数。通常情况下可以和 CPU 数量相等。但在 CPU 数量比较多的情况下，设置相对较小的数值也是合理的。
- -XX:MaxGCPauseMills：设置最大垃圾收集停顿时间。它的值是一个大于 0 的整数。收集器在工作时，会调整 Java 堆大小或者其他一些参数，尽可能地把停顿时间控制在 MaxGCPauseMills 以内。
- -XX:GCTimeRatio:设置吞吐量大小，它的值是一个 0-100 之间的整数。假设 GCTimeRatio 的值为 n，那么系统将花费不超过 1/(1+n) 的时间用于垃圾收集。
- -XX:+UseAdaptiveSizePolicy:打开自适应 GC 策略。在这种模式下，新生代的大小，eden 和 survivor 的比例、晋升老年代的对象年龄等参数会被自动调整，以达到在堆大小、吞吐量和停顿时间之间的平衡点。

#### 与 CMS 回收器相关的参数

- -XX:+UseConcMarkSweepGC: 新生代使用并行收集器，老年代使用 CMS+串行收集器。
- -XX:+ParallelCMSThreads: 设定 CMS 的线程数量。
- -XX:+CMSInitiatingOccupancyFraction:设置 CMS 收集器在老年代空间被使用多少后触发，默认为 68%。
- -XX:+UseFullGCsBeforeCompaction:设定进行多少次 CMS 垃圾回收后，进行一次内存压缩。
- -XX:+CMSClassUnloadingEnabled:允许对类元数据进行回收。
- -XX:+CMSParallelRemarkEndable:启用并行重标记。
- -XX:CMSInitatingPermOccupancyFraction:当永久区占用率达到这一百分比后，启动 CMS 回收 (前提是-XX:+CMSClassUnloadingEnabled 激活了)。
- -XX:UseCMSInitatingOccupancyOnly:表示只在到达阈值的时候，才进行 CMS 回收。
- -XX:+CMSIncrementalMode:使用增量模式，比较适合单 CPU。

#### 与 G1 回收器相关的参数

- -XX:+UseG1GC：使用 G1 回收器。
- -XX:+UnlockExperimentalVMOptions:允许使用实验性参数。
- -XX:+MaxGCPauseMills:设置最大垃圾收集停顿时间。
- -XX:+GCPauseIntervalMills:设置停顿间隔时间。

#### 其他参数

- -XX:+DisableExplicitGC: 禁用显示 GC。
