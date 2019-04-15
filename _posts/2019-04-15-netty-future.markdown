---
layout:     post
title:      "netty的Future接口"
subtitle:   "分析Future接口的功能以及实现"
date:       2019-04-15
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - netty
---

netty主要通过异步编程的方式提高并发。而在异步编程中，future是重要的组成部分，因为它可以表示异步返回的结果，是一种很好的抽象。

### jdk中Future的方法

#### 1. `boolean isDone()`
表示结果是否计算完成

#### 2. `boolean cancel(boolean mayInterrupt)`
用于取消计算

#### 3. `boolean isCancelled()`
判断是否已经被取消

#### 4. `V get() throws InterruptedException, ExecutionException`
获得结果，会阻塞线程，直到获得结果

#### 5. `V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException`
获得结果，会阻塞线程，直到获得结果或者时间超时


#### 如何实现
##### 1. 如何实现状态转换
由于Future的状态仅能由一个线程操作，并且不可逆，所以可以使用**原子操作的方式**实现。另外除了将状态之外，还需要设置结果，这是两个操作，为了避免使用使用锁，可以通过一个变量同时表示状态与结果。通过属性Object result表示

状态 | 值
-----|------
没有结果 | 放置null，表示任务没有完成
成功 | 放置执行的结果，如果结果为null，放置SUCCESS常量
失败 | **放置失败的原因，通过CauseHolder包装，用于区分结果**
取消 | 放置CANCEL常量

> 对对象的原子操作可以直接使用AtomicReferrence实现。

##### 2. 如何get方式
使用`Object.wait()`与`Object.wait(long timeout, int nanos)`。在任务结束之后，通知所有等待的线程。
1. wait需要使用使用while包裹，确实满足条件才能跳出循环
2. 合理处理InterruptedException，因为有些等待是不能中断，此时需要将InterruptedException捕获


#### 总结
1. future应该可以用于判断**任务是否执行完成**，**获得结果**，**取消任务**
2. 对于会阻塞的方法，应该要抛出异常**InterruptedException**，而对于有限时间阻塞的方法，如果会抛出**TimeoutException**，应该也要添加



----

netty中的Future对jdk中的future进行了增强，增加更多方法使它能实现更加复杂的功能

### 主要增加的方法
#### 1. `boolean isSuccess()`
判断任务成功执行

#### 2. `boolean isCancellable()`
是否可以取消，可以设置不可取消的任务

#### 3. `Throwable cause()`
获取失败的原因

#### 4. `void addListener(GenericFutureListener<? extends Future<? super V>> listener)`
用于添加监听器，在任务完成之后自动执行

#### 5. `Future<V> sync()`
用于同步等待任务的执行，任务执行完成会抛出异常

#### 6. `Future<V> await()`
用于等待任务执行完成，不会抛出任务执行的异常

#### 7. `boolean await(long timeout, TimeUnit unit)`
等待指定时间。

#### 8. `V getNow()`
用于获取当前结果，只有成功执行完成后才会返回结果，否者返回null，不会阻塞线程


主要增加两个功能
1. 监听器listener
2. sync，await两个阻塞方法


#### 如何实现？
##### 1. 监听器如何实现？
监听器是在任务结束之后被执行，需要使用一个列表，专门放置添加的listener，在执行时去这个列表中获得监听器。另一需要注意的地方是，在添加和获取时一定要加锁，否者会造成多线程问题。
1. 如果添加listener时，任务已经结束，需要立即触发listener
2. 在触发任务时，listener可能会执行执行addListener，此时又会触发执行任务。如果程序比较复杂可能导致调用栈过深，可以通过一个变量检测listener的调用深度，如果超过阈值，使用异步的方式执行后面的listener。

##### 2. sync,await函数怎么实现
同样可以使用`Object.wait()`和`Object.wait(long timeout, int nanos)`实现。






