---
title: 设计模式之建造者（Builder）模式
date: 2019-9-5 22:00:00
author: xp
categories:
- Java
- 设计模式
tags:
- Java
- 设计模式
- 设计原则
---

# 设计模式之建造者（Builder）模式

> 将一个复杂对象的构建与它的表示分离,使得同样的构建过程可以创建不同的表示，Builder模式是一步一步创建一个复杂的对象,它允许用户可以只通过指定复杂对象的类型和内容就可以构建它们.用户不知道内部的具体构建细节。

## 如何给对象的属性赋值

### 构造器赋值

应用举例：假设我们有一个Person类，结构如下：

```java
public class Person {

    //必要参数
    private int id;
    private String name;
    //可选参数
    private int age;
    private String sex;
    private String phone;
    private String address;
    private String desc;

    public Person(int id, String name) {
        this(id, name, 0);
    }

    public Person(int id, String name, int age) {
        this(id, name, age, "");
    }

    public Person(int id, String name, int age, String sex) {
        this(id, name, age, sex, "");
    }

    public Person(int id, String name, int age, String sex, String phone) {
        this(id, name, age, sex, phone, "");
    }

    public Person(int id, String name, int age, String sex, String phone, String address) {
        this(id, name, age, sex, phone, address, "");
    }

    public Person(int id, String name, int age, String sex, String phone, String address, String desc) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.sex = sex;
        this.phone = phone;
        this.address = address;
        this.desc = desc;
    }
}
```

从上面的代码中，当你想要创建实例的时候，就利用构造器创建对象赋值，如果需要的参数过多的话就如下代码：
*Person person = new Persion(1, "李四", 20, "男", "18800000000", "China", "测试使用重叠构造器模式");*
创建使用代码会很难写，并且较难以阅读。

### JavaBean赋值

接下来，我们利用最熟悉的JavaBean模式进行替代：

```java
public class Person {

    //必要参数
    private int id;
    private String name;
    //可选参数
    private int age;
    private String sex;
    private String phone;
    private String address;
    private String desc;

    public void setId(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public void setDesc(String desc) {
        this.desc = desc;
    }
}
```

这种模式弥补了重叠构造器模式的不足。创建实例很容易，这样产生的代码读起来也很容易：

```java
Person person = new Person();
person.setId(1);
person.setName("李四");
person.setAge(20);
person.setSex("男");
person.setPhone("17255985656");
person.setAddress("China");
person.setDesc("使用JavaBeans模式");
```

遗憾的是，JavaBeans模式自身有着很重要的缺点。因为构造过程被分到了几个调用中，它并不是原子操作，在构造过程中JavaBean可能处于不一致的状态。类无法仅仅通过检验构造器参数的有效性来保证一致性。

### Builder模式

```java
public class Person {
    //必要参数
    private int id;
    private String name;
    //可选参数
    private int age;
    private String sex;
    private String phone;
    private String address;
    private String desc;

    private Person(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.age = builder.age;
        this.sex = builder.sex;
        this.phone = builder.phone;
        this.address = builder.address;
        this.desc = builder.desc;
    }

    public static class Builder {
        //必要参数
        private final int id;
        private final String name;
        //可选参数
        private int age;
        private String sex;
        private String phone;
        private String address;
        private String desc;

        public Builder(int id, String name) {
            this.id = id;
            this.name = name;
        }

        public Builder age(int val) {
            this.age = val;
            return this;
        }

        public Builder sex(String val) {
            this.sex = val;
            return this;
        }

        public Builder phone(String val) {
            this.phone = val;
            return this;
        }

        public Builder address(String val) {
            this.address = val;
            return this;
        }

        public Builder desc(String val) {
            this.desc = val;
            return this;
        }

        public Person build() {
            return new Person(this);
        }
    }
}
```

下面是客户端代码调用：

```java
public class Client {

    public static void main(String[] args) {
        Person person = new Person.Builder(1, "张三")
            .age(18).sex("男").desc("测试使用builder模式").build();
    }
}
```

#### 总结

Builder模式提供了良好的封装性：使用建造者模式可以使客户端不必知道产品内部组成的细节，上例中，我们不用关心构建的Person对象的过程，产生的对象就是Person类型。
Builder模式可以直观的构建复杂的对象，便于阅读，在很多开源组件中，我们都可以找到它的身影，比如ElasticSearch封装的JavaAPI中，`QueryBuilder`就是builder模式。
