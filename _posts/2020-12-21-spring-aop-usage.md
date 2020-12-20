---
layout: post
title: "Spring Aop学习"
subtitle: "Spring Aop的使用"
date: 2020-12-21
header-img: "img/post-bg-js-version.jpg"
author: 何吴涛
tags: 
  - Spring
  - 笔记
---



# Spring中Aop的用法

## 1. 方法拦截

### 1.1 基本概念

Aop中有三个重要概念

#### 1.1.1 JoinPoint连接点

JoinPoint连接点。表示需要放入**增强逻辑**的**原逻辑**。当前Spring支持的**原逻辑和增强逻辑的粒度都为方法**。

#### 1.1.2 Pointcut切入点

切入点是连接点的表达，一个切入点可以表示一个或多个连接点。

Spring AOP支持的AspectJ切入点指示符如下：
execution：用于匹配方法执行的连接点；
within：用于匹配指定类型内的方法执行；
this：用于匹配当前AOP代理对象类型的执行方法；注意是AOP代理对象的类型匹配，这样就可能包括引入接口也类型匹配；
target：用于匹配当前目标对象类型的执行方法；注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配；
args：用于匹配当前执行的方法传入的参数为指定类型的执行方法；
@within：用于匹配所以持有指定注解类型内的方法；
@target：用于匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解；
@args：用于匹配当前执行的方法传入的参数持有指定注解的执行；
@annotation：用于匹配当前执行方法持有指定注解的方法；
bean：Spring AOP扩展的，AspectJ没有对于指示符，用于匹配特定名称的Bean对象的执行方法；
reference pointcut：表示引用其他命名切入点，只有@ApectJ风格支持，Schema风格不支持。
AspectJ切入点支持的切入点指示符还有： call、get、set、preinitialization、staticinitialization、initialization、handler、adviceexecution、withincode、cflow、cflowbelow、if、@this、@withincode；但Spring AOP目前不支持这些指示符，使用这些指示符将抛出IllegalArgumentException异常。这些指示符Spring AOP可能会在以后进行扩展。

#### 1.1.3 Advice增强

表示在增强位置插入的增强逻辑，也就是增强方法。通知有5种：Before，After，AfterReturning，AfterThrowing，Around。这五种通知不同之处在于它们执行的时间点不同

| Advice          | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| @Before         | 在原方法执行前执行。                                         |
| @After          | 在原方法执行后执行。不管原方法有没有抛出异常，都会执行增强方法 |
| @AfterReturning | 在原方法正常执行后执行。正常执行指原方法没有抛出异常         |
| @AfterThrowing  | 在原方法异常执行后执行。                                     |
| @Around         | 在增强方法内部执行原方法。增强方法可以根据内部逻辑决定是否执行原方法。 |

下图为一个原方法被增强后，Advice和原方法调用顺序。

![aop增强逻辑的执行顺序](/img/Spring中Aop的用法.assets/aop增强逻辑的执行顺序.png)



### 1.2 具体使用

#### 1.2.1 通配符

1. `*`表示任何数量的任何字符，`com.hwt.*Service`。
2. `+`表示类型的子类型，`com.hwt.IService+`。
3. `..`表示任意类型，在包路径中和参数列表中表示省略。`execution(* com.hwt.service..*.*(..))`

#### 1.2.2 pointcut的书写格式

##### 1.2.2.1 within(指定类型)

通过**目标对象的类型**匹配

| 模式                                 | 描述                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| within(cn.javass..*)                 | cn.javass包及子包下的任何方法执行                            |
| within(cn.javass..IPointcutService+) | cn.javass包或所有子包下IPointcutService类型及子类型的任何方法 |
| within(@cn.javass..Secure *)         | 持有cn.javass..Secure注解的任何类型的任何方法必须是在目标对象上声明这个注解，在接口上声明的对它不起作用 |



##### 1.2.2.2 this(指定类型全限定名)

1. this仅支持全限定名，不支持通配符
2. this匹配在**代理对象**中**指定类型**的所有方法（被重写？）

| 模式                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| this(cn.javass.spring.chapter6.service.IPointcutService)     | 当前AOP对象实现了 IPointcutService接口的任何方法             |
| this(cn.javass.spring.chapter6.service.IIntroductionService) | 当前AOP对象实现了 IIntroductionService接口的任何方法也可能是引入接口 |



##### 1.2.2.3 target(指定类型全限定名)

1. target仅支持全限定名，不支持通配符
2. target匹配在**目标对象**中指定类型的所有方法

| 模式                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| target(cn.javass.spring.chapter6.service.IPointcutService)   | 当前目标对象（非AOP对象）实现了 IPointcutService接口的任何方法 |
| target(cn.javass.spring.chapter6.service.IIntroductionService) | 当前目标对象（非AOP对象） 实现了IIntroductionService 接口的任何方法不可能是引入接口 |



##### 1.2.2.4 bean(指定bean名称)

通过**bean的名称**匹配。bean是Spring对AspectJ的扩展。

| 模式           | 描述                                        |
| -------------- | ------------------------------------------- |
| bean(*Service) | 匹配所有以Service命名（id或name）结尾的Bean |



##### 1.2.2.5 args(参数类型列表)

1. 参数类型仅支持全限定名，不支持通配符
2. 参数类列表匹配**实际传入参数的类型**，不是被调用方法的参数类型

| 模式                           | 描述                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| args (java.io.Serializable,..) | 任何一个以接受“传入参数类型为 java.io.Serializable” 开头，且其后可跟任意个任意类型的参数的方法执行，args指定的参数类型是在运行时动态匹配的 |



##### 1.2.2.6 @within(指定注解类型全限定名)

1. 注解类型仅支持全限定名，不支持通配符
2. 在**目标对象类型**中持有，并**声明类（不能是接口）**有指定注解的方法



##### 1.2.2.7 @target(指定注解类型全限定名)

1. 注解类型仅支持全限定名，不支持通配符
2. **目标对象类型**有指定注解。**父类和父接口都不行**



##### 1.2.2.8 @args(注解类型列表)

1. 注解类型仅支持全限定名，不支持通配符
2. **实际传入的参数的类型**是否有目标注解



##### 1.2.2.9 @annotation(指定注解类型)

执行的方法上有指定注解



##### 1.2.2.10 execution(方法表达式)

根据**被执行方法**的签名匹配

| 模式                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public * *(..)                                               | 任何公共方法的执行                                           |
| * cn.javass..IPointcutService.*()                            | cn.javass包及所有子包下IPointcutService接口中的任何无参方法  |
| * cn.javass..*.*(..)                                         | cn.javass包及所有子包下任何类的任何方法                      |
| * cn.javass..IPointcutService.*(*)                           | cn.javass包及所有子包下IPointcutService接口的任何只有一个参数方法 |
| * (!cn.javass..IPointcutService+).*(..)                      | 非“cn.javass包及所有子包下IPointcutService接口及子类型”的任何方法 |
| * cn.javass..IPointcutService+.*()                           | cn.javass包及所有子包下IPointcutService接口及子类型的的任何无参方法 |
| * cn.javass..IPointcut*.test*(java.util.Date)                | cn.javass包及所有子包下IPointcut前缀类型的的以test开头的只有一个参数类型为java.util.Date的方法，注意该匹配是根据方法签名的参数类型进行匹配的，而不是根据执行时传入的参数类型决定的如定义方法：public void test(Object obj);即使执行时传入java.util.Date，也不会匹配的； |
| * cn.javass..IPointcut*.test*(..) throws IllegalArgumentException, ArrayIndexOutOfBoundsException | cn.javass包及所有子包下IPointcut前缀类型的的任何方法，且抛出IllegalArgumentException和ArrayIndexOutOfBoundsException异常 |
| * (cn.javass..IPointcutService+&& java.io.Serializable+).*(..) | 任何实现了cn.javass包及所有子包下IPointcutService接口和java.io.Serializable接口的类型的任何方法 |
| @java.lang.Deprecated * *(..)                                | 任何持有@java.lang.Deprecated注解的方法                      |
| @java.lang.Deprecated @cn.javass..Secure  * *(..)            | 任何持有@java.lang.Deprecated和@cn.javass..Secure注解的方法  |
| @(java.lang.Deprecated \|\| cn.javass..Secure) * *(..)       | 任何持有@java.lang.Deprecated或@ cn.javass..Secure注解的方法 |
| (@cn.javass..Secure  *) *(..)                                | 任何返回值类型持有@cn.javass..Secure的方法                   |
| *  (@cn.javass..Secure *).*(..)                              | 任何定义方法的类型持有@cn.javass..Secure的方法               |
| * *(@cn.javass..Secure (*) , @cn.javass..Secure (*))         | 任何签名带有两个参数的方法，且这个两个参数都被@ Secure标记了，如public void test(@Secure String str1, @Secure String str1); |
| * *((@ cn.javass..Secure *))或* *(@ cn.javass..Secure *)     | 任何带有一个参数的方法，且该参数类型持有@ cn.javass..Secure；如public void test(Model model);且Model类上持有@Secure注解 |
| * *(@cn.javass..Secure (@cn.javass..Secure *) ,@ cn.javass..Secure (@cn.javass..Secure *)) | 任何带有两个参数的方法，且这两个参数都被@ cn.javass..Secure标记了；且这两个参数的类型上都持有@ cn.javass..Secure； |
| * *(java.util.Map<cn.javass..Model, cn.javass..Model>, ..)   | 任何带有一个java.util.Map参数的方法，且该参数类型是以< cn.javass..Model, cn.javass..Model >为泛型参数；注意只匹配第一个参数为java.util.Map,不包括子类型；如public void test(HashMap<Model, Model> map, String str);将不匹配，必须使用“* *(java.util.HashMap<cn.javass..Model,cn.javass..Model>, ..)”进行匹配；而public void test(Map map, int i);也将不匹配，因为泛型参数不匹配 |
| * *(java.util.Collection<@cn.javass..Secure *>)              | 任何带有一个参数（类型为java.util.Collection）的方法，且该参数类型是有一个泛型参数，该泛型参数类型上持有@cn.javass..Secure注解；如public void test(Collection<Model> collection);Model类型上持有@cn.javass..Secure |
| * *(java.util.Set<? extends HashMap>)                        | 任何带有一个参数的方法，且传入的参数类型是有一个泛型参数，该泛型参数类型继承与HashMap；Spring AOP目前测试不能正常工作 |
| * *(java.util.List<? super HashMap>)                         | 任何带有一个参数的方法，且传入的参数类型是有一个泛型参数，该泛型参数类型是HashMap的基类型；如public voi test(Map map)；Spring AOP目前测试不能正常工作 |
| * *(*<@cn.javass..Secure *>)                                 | 任何带有一个参数的方法，且该参数类型是有一个泛型参数，该泛型参数类型上持有@cn.javass..Secure注解；Spring AOP目前测试不能正常工作 |



## 2. 类型扩展

Spring Aop 除了可以增强方法，还可以增强类型

通过`@DeclareParents`注解，可以扩展bean的实现

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface DeclareParents {

    // 类型匹配表达式。通过类路径匹配，支持通配符，例如com..*ServiceImpl
    String value();

    // 扩展接口的实现类
    Class defaultImpl() default DeclareParents.class;

    // note - a default of "null" is not allowed,
    // hence the strange default given above.
}
```



### 2.1 定义扩展的接口

```java
package com.hwt.service;

public interface OtherService {
    void doOther();
}

```

### 2.2 定义扩展接口的实现

```java
package com.hwt.service.impl;

import com.hwt.service.OtherService;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class OtherServiceImpl implements OtherService {
    private static final Logger log = LogManager.getLogger(OtherService.class);
    @Override
    public void doOther() {
        log.info("haha!");
    }
}
```



### 2.3 配置切面

```java
package com.hwt.aop;

import com.hwt.service.OtherService;
import com.hwt.service.impl.OtherServiceImpl;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class UserServiceAop {
    @DeclareParents(value = "com.hwt.service.UserService+", defaultImpl = OtherServiceImpl.class)
    public OtherService other;
}

```



### 2.4 使用

```java
package com.hwt;

import com.hwt.service.OtherService;
import com.hwt.service.UserService;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class AopStudyMain {
    private static final Logger log = LogManager.getLogger(AopStudyMain.class);
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring-context.xml");
        UserService userService = context.getBean(UserService.class);
        OtherService otherService = (OtherService) userService;
        otherService.doOther();
    }
}

// 输出
/*
2020-12-21 00:19:10,164 [INFO ] haha! (com.hwt.service.impl.OtherServiceImpl.doOther(OtherServiceImpl.java:11))
*/
```





 