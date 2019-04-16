---
layout:     post
title:      "设计原则-开闭原则"
subtitle:   "如何增强程序的可扩展性"
date:       2019-04-16
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - design-principle
---

> 开闭原则(open-closed Principle): 一个软件实体应当对扩展开放，对修改关闭。即软件应该在尽量不修改原有代码的情况下进行扩展。

在开闭原则的定义中，软件实体可以指一个模块，一个由多个类组成的局部结构，或者一个独立的类。

系统的扩展性对于一个大系统是非常重要的。如果设计时候不满足开闭原则，那在对系统扩展时，将需要需要修改原有代码，将会导致维护成本越来越高。

**开闭原则应当尽量使用接口，抽象类。**
1. 对于创建新类，通过接口和抽象类可以让类具有统一性，并且抽象类可更加方便扩展
2. 对于使用新类，作为类的使用方，如果是通过接口或者抽象类使用目标类，那将不需要修改代码，否者需要修改。


#### 糟糕的设计
当新增Chart时，将会导致ChartDisplay类也需要修改。
```java
public class ChartDisplay {
    void display(String type) {
        if (type.equals("pie")) {
            PieChart chart = new PieChart();
            chart.display();
        } else if (type.equals("bar")) {
            BarChart chart = new BarChart();
            chart.display();
        }
        throw new IllegalArgumentException();
    }
}
```

#### 改进
通过对Chart使用接口设计，而ChartDisplay也通过接口去使用实现类。这是对于Chart的扩展将不会影响Display

```java
// Display
public class ChartDisplay {
    void display(Chart chart) {
        chart.display();
    }
}

// ------------------
// chart接口
public interface Chart {
    void display();
}

public class PieChart implement Chart {
    public void display() {
        System.out.println("PieChart");
    }
}

public class BarChart implement Chart {
    public void display() {
        System.out.println("BarChart");
    }
}


```
