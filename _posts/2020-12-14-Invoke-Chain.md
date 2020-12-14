---
layout: post
title: "spring知识点学习"
subtitle: "调用链实现"
date: 2020-12-14
header-img: "img/post-bg-js-version.jpg"
author: 何吴涛
tags: 

  - Spring
  - 笔记
---

# 调用链机制

在Servlet中的Filter和Spring中的方法拦截都使用了调用链机制。



## 1. 什么是调用链？

调用链是多个方法的一种特殊组合方式，它有以下几个特点

1. 调用链的方法依次排列
2. 第一个方法由外部执行
3. 最后一个方法是目标方法，也就是真正想被执行的方法
4. 前一个方法仅可以执行后一个方法。前一个方法可以根据自己逻辑选择是否执行后一个方法，也就是调用链可能不会被完整执行，可以在任意一个方法返回（断开）。





## 2. 调用链的作用

主要是用于拦截调用过程，拦截后可以实现控制是否执行调用，修改调用参数和返回值，记录调用参数和返回值等



## 3. 调用链的实现

### 3.1 首先定义Filter接口和Chain接口

Filter表示调用链中的某个方法

```java
package com.hwt.chain;

public interface Filter {
    Object doFilter(Chain chain) throws Exception;
}
```

Chain表示整个调用链

 ```java
package com.hwt.chain;

import java.lang.reflect.Method;

public interface Chain {
    Object proceed() throws Exception;
    Object proceed(Object[] args) throws Exception;
    Object[] getArgs();
    Method getMethod();
}

 ```



### 3.2 实现对对象的代理

ProxyFactory用于创建对象的代理

```java
package com.hwt.chain;

import java.lang.reflect.Proxy;
import java.util.List;

public class ProxyFactory {
    @SuppressWarnings("unchecked")
    public Object create(Object target, Class<?>[] interfaces, List<Filter> filters) {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), interfaces, new FilterInvocationHandler(target, filters));
    }
}

```



FilterInvocationHandler用于处理对方法的调用。FilterInvocationHandler会将所有的Filter包装成一个Chain。最后调用chain.proceed()方法触发整个调用链

```java
package com.hwt.chain;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.List;

public class FilterInvocationHandler implements InvocationHandler {
    private final Object target;
    private final List<Filter> filters;


    public FilterInvocationHandler(Object target, List<Filter> filters) {
        this.target = target;
        this.filters = filters;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        AbstractChain chain = buildLastChain(target, method);
        for (int i = filters.size() - 1; i >= 0; i--) {
            Filter filter = filters.get(i);
            chain = buildChain(method, filter, chain);
        }
        chain.setArgs(args);
        return chain.proceed();
    }

    private AbstractChain buildChain(Method method, Filter filter, AbstractChain nextChain) {
        return new AbstractChain() {
            @Override
            public Object proceed() throws Exception {
                nextChain.setArgs(args);
                return filter.doFilter(nextChain);
            }

            @Override
            public Object proceed(Object[] args) throws Exception {
                nextChain.setArgs(args);
                return filter.doFilter(nextChain);
            }

            @Override
            public Method getMethod() {
                return method;
            }
        };
    }

    private AbstractChain buildLastChain(Object target, Method method) {
        return new AbstractChain() {
            @Override
            public Object proceed() throws Exception {
                return method.invoke(target, args);
            }

            @Override
            public Object proceed(Object[] args) throws Exception {
                return method.invoke(target, args);
            }

            @Override
            public Method getMethod() {
                return method;
            }
        };
    }

    private static abstract class AbstractChain implements Chain {
        protected Object[] args;

        protected void setArgs(Object[] args) {
            this.args = args;
        }

        @Override
        public Object[] getArgs() {
            return args;
        }
    }
}
```



### 3.3 自定义Filter

ReplaceArgsFilter可以替换调用过程中的参数

```java
package com.hwt.chain.impl;

import com.hwt.chain.Chain;
import com.hwt.chain.Filter;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class ReplaceArgsFilter implements Filter {
    private static final Logger log = LogManager.getLogger(ReplaceArgsFilter.class);
    @Override
    public Object doFilter(Chain chain) throws Exception {
        Object[] args = chain.getArgs();
        Object[] replaceArgs = args.clone();
        replaceArgs[0] = "hwt";
        log.info("received args: {}, replaced args: {}", args, replaceArgs);
        return chain.proceed(replaceArgs);
    }
}
```

ReplaceReturnFilter可以用于替换方返回值

```java
package com.hwt.chain.impl;

import com.hwt.chain.Chain;
import com.hwt.chain.Filter;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class ReplaceReturnFilter implements Filter {
    private static final Logger log = LogManager.getLogger(ReplaceReturnFilter.class);
    @Override
    public Object doFilter(Chain chain) throws Exception {
        Object value = chain.proceed();
        Object replacedValue = 20L;
        log.info("received value: {}, replaced value: {}", value, replacedValue);
        return replacedValue;
    }
}
```



### 3.4 测试

定义UserService接口，和其实现类UserServiceImpl。

```java
package com.hwt.chain;

public interface UserService {
    Long add(String name);
}

```



```java
package com.hwt.chain.impl;

import com.hwt.chain.UserService;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class UserServiceImpl implements UserService {
    private static final Logger log = LogManager.getLogger(UserServiceImpl.class);
    @Override
    public Long add(String name) {
        log.info("name: {}, return: {}", name, 10L);
        return 10L;
    }
}
```



对UserServiceImpl进行代理

```java
import com.hwt.chain.impl.ReplaceReturnFilter;
import com.hwt.chain.impl.UserServiceImpl;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.Arrays;
import java.util.List;

public class ChainMain {
    private static final Logger log = LogManager.getLogger(ChainMain.class);
    public static void main(String[] args) {
        ProxyFactory proxyFactory = new ProxyFactory();
        UserServiceImpl userService = new UserServiceImpl();
		
        // 添加两个Filter
        List<Filter> filters = Arrays.asList(
                new ReplaceArgsFilter(),
                new ReplaceReturnFilter()
        );
		
        // 创建代理
        UserService proxied = (UserService) proxyFactory.create(userService, new Class[]{UserService.class}, filters);

        String name = "outerName";
        log.info("start add. name: {}", name);
        Long value = proxied.add(name);
        log.info("end add. return value: {}", value);

    }
}

// 输出
/*
2020-12-15 00:20:50,273 [INFO ] start add. name: outerName (com.hwt.chain.ChainMain.main(ChainMain.java:27))
2020-12-15 00:20:50,273 [INFO ] received args: [outerName], replaced args: [hwt] (com.hwt.chain.impl.ReplaceArgsFilter.doFilter(ReplaceArgsFilter.java:15))
2020-12-15 00:20:50,273 [INFO ] params: hwt, return: 10 (com.hwt.chain.impl.UserServiceImpl.add(UserServiceImpl.java:11))
2020-12-15 00:20:50,273 [INFO ] received value: 10, replaced value: 20 (com.hwt.chain.impl.ReplaceReturnFilter.doFilter(ReplaceReturnFilter.java:14))
2020-12-15 00:20:50,273 [INFO ] end add. return value: 20 (com.hwt.chain.ChainMain.main(ChainMain.java:29))
*/
```

可以看出UserServiceImpl的参数被ReplaceArgsFilter替换了，返回值被ReplaceReturnFilter替换了。



## 4. 总结

1. FilterInvocationHandler的实现中，invoke方法会在每次调用都重新创建一个Chain，并不是一个很好的实现方式。
2. 可以通过Chain接口给用户提供更多的调用信息