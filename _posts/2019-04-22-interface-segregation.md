---
layout:     post
title:      "设计原则-接口隔离"
subtitle:   "让使用与实现都更简单"
date:       2019-04-22
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - design-principle
---

> 接口隔离原则：使用多个专门的接口，而不是使用单一的总接口，即客户端不应该依赖那些它不需要的接口

接口隔离要求根据客户端的需求，对于不同的功能使用适合的接口，而不是使用一个较大的接口。这样对实现与使用都有好处
1. 对使用来说，提供适合的接口，使用者的学习成本较低，同时不需要考虑哪些方法不能调用
2. 对于实现来说，仅需要实现需要的方法，不需要去实现不需要的功能。


单一责任原则要求接口尽可能功能单一，而接口隔离原则要求根据需求书写合适的接口。使用这个两个原则需要进行权衡，否者会产生较多接口，导致难以维护。

#### 不好的设计
通常提供一个过大的接口，就不满足接口隔离原则

```java
// 一个接口中同时提供处理Chart与Report的功能
// 但是客户端在某些地方只需要Chart或Report功能
public interface DataHandler {
    Chart createChart(File file);
    void displayChart(Chart chart);
    
    Report createReport(File file);
    void displayReport(Report report);
}
```


#### 修改
将DataHandler拆分成两个接口，这样使用者可以选择合适的接口

```java
public interface ChartHandler {
    Chart createChart(File file);
    void displayChart(Chart chart);
}


public interface ReportHandler {
    Report createReport(File file);
    void displayReport(Report report);
}
```