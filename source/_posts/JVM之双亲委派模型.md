---
title: JVM之双亲委派模型
date: 2018-10-28 23:42:31
author: xp
categories:
- Java
- JVM
tags:
- Java
- JVM
- 双亲委派
---

# JVM之双亲委派模型

## 类加载器

&nbsp;&nbsp;在类的加载阶段，需要通过一个类的全限定名来获取定义此类的二进制字节流，这**二进制字节流**并没有指明需要从Class文件中获取，也可以通过ZIP包、网络、动态代理方式或者是其它文件数据库等资源类型中获取，完成这个动作的代码块就是类加载器。这一动作是放在Java虚拟机外部去实现的，以便让应用程序自己决定如何获取所需的类。类加载器（ClassLoader）是Java语言的一项创新，也是Java流行的一个重要原因。

### 唯一性

&nbsp;&nbsp;&nbsp;&nbsp;类加载器虽然只用于实现类的加载动作，但是对于任意一个类，都需要由加载它的类加载器和这个类本身共同确立其在Java虚拟机中的唯一性。通俗的说，JVM中两个类是否“相等”，首先就必须是同一个类加载器加载的，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要类加载器不同，那么这两个类必定是不相等的。
&nbsp;&nbsp;&nbsp;&nbsp;这里的“相等”，包括代表类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括使用instanceof关键字做对象所属关系判定等情况。
&nbsp;&nbsp;&nbsp;&nbsp;以下代码说明了不同的类加载器对instanceof关键字运算的结果的影响。

```java
public class ClassLoaderTest {

    public static void main(String[] args)
            throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        ClassLoader classLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                InputStream inputStream = getClass().getResourceAsStream(fileName);
                if (inputStream == null){
                    return super.loadClass(name);
                }
                byte[] bytes;
                try {
                    bytes = new byte[inputStream.available()];
                    inputStream.read(bytes);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
                return defineClass(name,bytes,0,bytes.length);
            }
        };
        // 使用ClassLoaderTest的类加载器加载本类
        /*Object obj1 = ClassLoaderTest.class.getClassLoader().loadClass("factory.ClassLoaderTest").newInstance();
        System.out.println(obj1.getClass());
        System.out.println(obj1 instanceof factory.ClassLoaderTest);*/

        // 使用自定义类加载器加载本类
        Object object = classLoader.loadClass("factory.ClassLoaderTest").newInstance();
        System.out.println(object.getClass());
        System.out.println(object instanceof factory.ClassLoaderTest);
    }
}
```

&nbsp;&nbsp;&nbsp;&nbsp;运行结果为：
**&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;class factory.ClassLoaderTest
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;true
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;class factory.ClassLoaderTest
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;false**

> 上述例子中，虚拟机中存在了两个**ClassLoaderTest**，就是因为所使用的类加载器不同，一个是由系统应用程序类加载器加载的，另外一个是由我们自定义的类加载器加载的，虽然都来自同一个Class文件，但依然是两个独立的类，做对象所属类型检查时结果自然为false。

## 双亲委派模型

&nbsp;&nbsp;从Java虚拟机的角度来说，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言实现（HotSpot虚拟机中），是虚拟机自身的一部分；另一种就是所有其他的类加载器，这些类加载器都有Java语言实现，独立于虚拟机外部，并且全部继承自java.lang.ClassLoader。
&nbsp;&nbsp;从开发者的角度，类加载器可以细分为：

- 启动类加载器（Bootstrap ClassLoader）
&nbsp;&nbsp;&nbsp;&nbsp;负责将放在<JAVA_HOME>\lib目录中的，或被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如rt.jar）类库加载到虚拟机内存中。 
启动类加载器无法被Java程序直接引用，用户再编写自定义加载器时，如果需要把加载器请求委派给启动类加载器，那么直接使用null代替即可。
- 扩展类加载器（Extension ClassLoader）
&nbsp;&nbsp;&nbsp;&nbsp;由sun.misc.Launcher$ExtClassLoader实现，它负责加载在<JAVA_HOME>\lib\ext目录中的，或被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。
- 应用程序类加载器（Application ClassLoader）
&nbsp;&nbsp;&nbsp;&nbsp;由sun.misc.Launcher$App-ClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值。所以一般也称它为系统类加载器。 它负责加载用户路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

&nbsp;&nbsp;还有自定义的类加载器，它们之间的层次关系被称为类加载器的双亲委派模型。该模型要求除了顶层的启动类加载器外，其余的类加载器都应该有自己的父类加载器，而这种父子关系一般通过组合（Composition）关系来实现，而不是通过继承（Inheritance）。

![类加载器双亲委派模型](https://img-blog.csdnimg.cn/20181028233129991.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_27,color_FFFFFF,t_70)

### 工作过程

&nbsp;&nbsp;&nbsp;&nbsp;如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上。 因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。
&nbsp;&nbsp;&nbsp;&nbsp;使用双亲委派模型的好处在于Java类随着它的类加载器一起具备了一种带有**优先级的层次关系**。例如类java.lang.Object，它存在在rt.jar中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的Bootstrap ClassLoader进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。相反，如果没有双亲委派模型而是由各个类加载器自行加载的话，如果用户编写了一个java.lang.Object的同名类并放在ClassPath中，那系统中将会出现多个不同的Object类，程序将混乱。因此，如果开发者尝试编写一个与rt.jar类库中重名的Java类，可以正常编译，但是永远无法被加载运行。

### 双亲委派模型实现

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // ①首先，检查请求的类是否已经被加载过了
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);//②委派父类加载
                    } else {
                        c = findBootstrapClassOrNull(name);//使用启动类加载器
                    }
                } catch (ClassNotFoundException e) {
                    // 如果父类加载器抛出异常
                    // 说明父类加载器无法完成加载请求
                }
                if (c == null) {
                    // ④在父类加载器无法加载的时候
                    // 再调用本身的findClass方法来进行类加载
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;在java.lang.ClassLoader的loadClass()方法中，先检查是否已经被加载过，若没有加载则调用父类加载器的loadClass()方法，若父加载器为空则默认使用启动类加载器作为父加载器。如果父加载失败，则抛出ClassNotFoundException异常后，再调用自己的findClass()方法进行加载。
