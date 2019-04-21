---
layout:     post
title:      "设计原则-依赖倒置"
subtitle:   "让类可以灵活的配置依赖"
date:       2019-04-17
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - design-principle
---

> 依赖倒置原则：抽象不应该依赖于细节，细节应该依赖于抽象。换言之，要针对接口变成，而不是针对实现编程

#### 抽象与细节（具体）
抽象相对于细节具有更高层次表示。例如车就是一个抽象的概念，它能代表一类具体的事物，而一辆自行车或者一辆卡车就是具体事物。

在java中，抽象类与接口就是抽象，而实现类就是具体


#### "倒置"的理解
依赖倒置就将本来应该由**内部决定的依赖关系转移到外部**。内部仅决定抽象，而外部决定具体。

~~通常来说上层依赖底层是直接依赖底层实现，此时底层实现是处于**被依赖**的状态。而当通过依赖倒置原则，上层依赖底层抽象，这就导致底层实现需要依赖底层抽象，也就是底层实现处于**依赖**状态。所以**倒置**就是从**被依赖**变成**依赖**。可以看出，这种设计会让上层对底层的具体实现具有更低的依赖，降低耦合。~~


#### 依赖注入
实现依赖倒置的方式就是依赖注入，依赖注入有3中方式
1. 构造函数注入
2. setter注入
3. 方法注入


#### 不好的设计
直接依赖具体类是不好的设计原则，会导致程序扩展困难。
```java
class Person {
    private Bike bike;
    public void go(String dest) {
        System.out.println("出发去" + dest);
        bike.drive();
    }
}
```

![person-bike-1](img/in-post/dependence-inversion/person-bike-1.png)

#### 改进
如果使用不同的交通工具将需要修改代码。所以可以将出行工具抽象成一个接口。而用户依赖接口，而不是直接依赖实现类

```java
class Person {
    private Drivable drivable;
    public Person(Drivable drivable) {
        this.drivable = this.drivable;
    }
    
    public void go(String dest) {
        System.out.println("出发去" + dest);
        drivable.drive();
    }    
    
}

// -------------------

public interface Drivable {
    void drive();
}

public class Bike implements Drivable {
    public void drive() {
        System.out.println("使用自行车");
    }
}

```

![person-bike-1](img/in-post/dependence-inversion/person-bike-2.png)

