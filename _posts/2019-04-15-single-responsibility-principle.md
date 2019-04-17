---
layout:     post
title:      "设计原则-单一责任原则"
subtitle:   "思考对接口，类，抽象类的适用"
date:       2019-04-15
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - design-principle
---

> 单一责任原则：一个类值负责一个功能领域的相应职责，或者可以定义为：就一个类，应该只有一个引起它变化的原因。

#### 对于接口而言
1. 在设计低级接口时，应该尽量满足单一责任原则，这样可以增加接口及其实现类的复用，同时方便组合出更加高级的接口
2. 对于高级接口，它本身就是代表着更多的功能，所以没有办法满足单一责任原则。


```java
/**
 * 仅实现过滤器功能
 */
public interface Filter {
    void doFilter(HttpServletRequest, req, HttpServletResponse, resp, FilterChain chain);
}
```


```java
/**
 * 高级的接口组合了多个接口的功能，通过多个简单接口实现
 */
public interface ApplicationContext extends ResourceLoader, BeanFactory {
    
}
```



#### 对于类而言
1. 简单的类，根据实际的逻辑，可能一个功能的实现是需要其他功能才能完成。此时类应该专注于自身的功能，将其他的功能委托给其他的类完成。
2. 对于复杂接口的实现类，同样可以通过委托的方法，将一些功能委托给其他类完成。


```java
/** 数据访问层 */
public interface UserDao {
    User findById(long id);
    User save(User user);
}

// -------------

/** 用户服务层 */
public interface UserService {
    void changeName(long id, String newName);
}

// --------------

/** 实现类 */
public class DefaultUserService implements UserService {
    private UserDao userDao; // 将需要数据访问的功能委托给UserDao
    public DefaultUserService(UserDao userDao) {
        this.userDao = userDao;
    }
    @Override
    public void changeName(long id, String newName) {
        User user = userDao.findById(id);
        if (user == null) {
            throw new UserNotFoundException();
        }
        user.setName(newName);
        userDao.save(user);
    }
}

```


#### 对于抽象类而言
抽象类介于接口与实现类之间。抽象类用于实现接口的重用逻辑，这样可以让实现类仅专注于自身的逻辑，减少实现类的工作量。


```java
public abstract class PathFilter {
    public void void doFilter(HttpServletRequest, req, HttpServletResponse, resp, FilterChain chain) {
        if (match(req)) {
            doFilterInternal(req, resp, chain);
        } else {
            chain.doFilter();
        }
    }
    
    // 匹配请求路径
    protected abstract boolean match(HttpServletRequest req);
    
    protected abstract void doFilterInternal(HttpServletRequest, req, HttpServletResponse, resp, FilterChain chain);
    
}

```
