---
layout:     post
title:      "设计原则-里氏原则"
subtitle:   "严格限制继承关系"
date:       2019-04-17
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - design-principle
---

> 里式替换原则：所有引用基类的地方必须能透明地使用其子类对象

里式替换原则告诉我们，将一个子类对象替换成父类对象，程序将不会产生任何错误与异常。为了满足里式替换原则，
1. 我们需要要求**子类必须拥有所有父类的特性**，否者不具有严格的继承关系
2. 要求**使用父类的地方对特性的要求需要更加明确**，否者有些子类可能并不适合,导致逻辑不严谨。


另外也可以使用里式替换原则**检查是否类之间存在相同的特征**，如果存在可以将相同的特征抽象成接口或者抽象类。

#### 不好的设计
如果Killer使用ToyGun去杀人，肯定是不合适。所以应该在Killer中对Gun有更高的特性要求。
```java
public interface Gun {
    void shoot();
}

//----------------

public class Killer {
    private Gun gun;
    // 通过Gun的shoot方法
    public void kill() {
        gun.shoot();
    }
}

//------------------
public class ToyGun implements Gun {
    public void shoot() {
        System.out.println("玩具枪");
    }
}

public class HandGun implements Gun {
    public void shoot() {
        System.out.println("手枪");
    }
}

```


#### 改进
增加另一个接口，用于区分真枪与假枪

```java
public interface Gun {
    void shoot();
}
// 标识真枪，具有杀伤力
public interface RealGun extends Gun {
}
// 标识假枪，不具有杀伤力
public interface FakeGun extends Gun {
}

//-------------------------
public class Killer {
    // 将枪的类型改为RealGun
    private RealGun gun;
    public void kill() {
        gun.shoot();
    }
}

//------------------
public class ToyGun implements FakeGun {
    public void shoot() {
        System.out.println("玩具枪");
    }
}

public class HandGun implements RealGun {
    public void shoot() {
        System.out.println("手枪");
    }
}

```
