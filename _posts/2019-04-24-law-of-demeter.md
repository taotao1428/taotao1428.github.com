---
layout:     post
title:      "设计原则-迪米特法则"
subtitle:   "减少类之间的耦合"
date:       2019-04-24
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - design-principle
---

> 迪米特法则（最少知识原则）：一个软件实体应当尽可能少地与其他实体发生相互作用

迪米特法则可以用于降低系统的耦合度，使类与类之间保持松散的耦合关系。



### 1. 对于引用者来说
**仅与具有直接关系的对象通信，也就是不要跨层引用**。这样可以有效的降低引用个数，减少类之间的耦合。

**当多个对象相互引用，可以通过构建一个中间者处理对象这件的引用关系**，从而可以减少每个类的引用个数。

例如：`Teacher` --> `GroupLeader` --> `Girl`

`Teacher`与`GroupLeader`具有直接关系，`GroupLeader`与`Girl`具有直接关系。

实现需求：老师需要知道女孩的个数
#### 不好的设计
直接将`Girl`放到`Teacher`中，其实它们没有直接关系
```java
public class Teacher {
    private GroupLeader groupLeader;
    private Set<Girl> girls;
    
    public Teacher(GroupLeader groupLeader, Set<Girl> girls) {
        this.groupLeader = groupLeader;
        this.girls = girls;
    }
    
    public int countGirls() {
        return groupLeader.countGirls(girls);
    }
}

// ------------------------

public class GroupLeader {
    public int countGirls(Set<Girls> girls) {
        return girls.size();
    }
}
```

#### 修改
将`Girl`放到`GroupLeader`中

```java
public class Teacher {
    private GroupLeader groupLeader;
    
    public Teacher(GroupLeader groupLeader) {
        this.groupLeader = groupLeader;
    }
    
    public int countGirls() {
        return groupLeader.countGirls();
    }
}

// ------------------------

public class GroupLeader {
    private Set<Girl> girls;
    
    public GroupLeader(Set<Girl> girls) {
        this.girls = girls;
    }
    
    public int countGirls() {
        return girls.size();
    }
}
```


### 2. 对被引用者来说
**应该尽量限制外部对属性与方法的访问**。也就是仅对外提供必要的方法，尽量不要让其他类涉及内部逻辑。


例如：`InstallSoftware` ---> `Wizard`

实现需要：安装软件

#### 不好的设计
安装的流程应该是由Wizard自己控制
```java
public class InstallSoftware {
    public void installWizard(Wizard wizard) {
        wizard.first();
        wizard.second();
        wizard.third();
    }
}

// -------------------

public class Wizard {
    public void first() {
        System.out.println("第一步");
    }
    
    public void second() {
        System.out.println("第二步");
    }
    
    public void third() {
        System.out.println("第三步");
    }
}
```

#### 改进
```java
public class InstallSoftware {
    public void installWizard(Wizard wizard) {
        wizard.install()
    }
}

// -------------------

public class Wizard {
    private void first() {
        System.out.println("第一步");
    }
    
    private void second() {
        System.out.println("第二步");
    }
    
    private void third() {
        System.out.println("第三步");
    }
    // 仅暴露一个方法给外部，将逻辑封装在内部
    public void install() {
        first();
        second();
        third();
    }
}
```


### 总结
迪米特法则用于降低类之间的耦合性
1. 当有明确的中介类，应该使用中介类，不要跨关系引用。
2. 当没有明确的中介类。需要根据实际情况，如果在不存在中介类时，类之间的耦合很严重，此时应当引入中介类。
3. 类应当暴露尽量少的信息给外部，减少其他类对它的耦合。