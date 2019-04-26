---
layout:     post
title:      "实现一个简单的单线程执行器"
subtitle:   ""
date:       2019-04-26
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - netty学习
---

EventLoop是netty中的重要组成部分。在读取源码的时候，同时学习一些实现思路。

### Executor主要功能
1. 执行任务，可以通过队列实现
2. 定时执行任务，可以通过优先队列实现


```java
package com.hewutao.model.executor;

import java.util.concurrent.TimeUnit;

/**
 * 简单的执行器
 */
public interface EventExecutor {
    /**
     * 添加任务
     */
    void execute(Runnable task);

    /**
     * 添加延时执行的任务
     */
    void schedule(Runnable task, long delay, TimeUnit unit);

    /**
     * 关闭执行器
     */
    void shutdown();

    /**
     * 是否关闭
     */
    boolean isShutdown();

    /**
     * 执行器关闭之后是否执行完任务
     */
    boolean isTerminated();
}
```

### 实现思路

#### 1. 普通任务的添加
通过一个BlockingQueue保存普通任务，可以多线程添加和阻塞获取。

```java
private final BlockingQueue<Runnable> taskQueue; // 执行的任务

@Override
public void execute(Runnable task) {
    if (isShutdown() || !taskQueue.offer(task)) { // 任务被拒绝
        reject(task);
    }
    if (!inLoop()) { // 不是执行线程
        startThread(); // 开启执行线程

        if (isShutdown()) { // 添加完成后再检查一次，检查是否关闭，如果关闭就移除任务
            try {
                if (taskQueue.remove(task)) {
                    reject(task);
                }
            } catch (Exception e) {
                //
            }
        }
    }
}
```

#### 2. 定时任务的添加
定时任务不采用BlockingQueue保存，而是采用PriorityQueue保存。
1. 如果当前是执行线程，不存在冲突直接添加到优先队列中
2. 如果不是执行线程，存在冲突，将添加的动作封装成一个普通Task,通过executor执行，这样添加的动作就会在执行线程完成

```java
private final Queue<ScheduledTask> scheduledTaskQueue; // 定时任务

    @Override
    public void schedule(Runnable task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (delay <= 0) {
            throw new IllegalArgumentException("delay: " + delay + " (expected: > 0)");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }

        final ScheduledTask scheduledTask = new ScheduledTask(task, delay, unit);
        if (inLoop()) {
            scheduledTaskQueue.offer(scheduledTask);
        } else {
            // 可能出现异常，所以还是需要future跟踪任务的情况
            execute(() -> scheduledTaskQueue.offer(scheduledTask));
        }
    }
```


#### 3. 状态的管理
执行器有多个状态，通过原子操作更新
```java
private static final int NOT_START = 1; // 未启动线程
private static final int STARTED = 2; // 已启动
private static final int SHUTDOWN = 3; // 已关闭
private static final int TERMINATED = 4; // 所有任务原型结束
private static final AtomicIntegerFieldUpdater<SingleThreadExecutor> STATE_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(SingleThreadExecutor.class, "state");

private volatile int state = NOT_START; // 执行器的状态        
```

#### 4. 获取任务执行
当执行线程运行时，需要从两个队列中获取任务执行，我将获取任务的动作通过takeTask()方法完成。
1. 将准备好的定时任务移到taskQueue中
2. 从taskQueue中poll一个任务，如果存在任务，则返回；否者，执行下一步
3. 从scheduledTaskQueue中peek一个任务，如果返回null，则设置一个默认的timeout时间；如果不返回null，则timeout为该任务准备好需等待的时间。
4. 执行taskQueue.poll(timeout, TimeUnit.NANOSECOND) 获得任务。如果返回不为null，返回结果；否者，重新回到1；

```java
private Runnable takeTask() {
    for (;;) {
        long currentNanos = System.nanoTime();
        fetchReadyScheduledTask(currentNanos); // 将准备好的定时任务，放到taskQueue中

        if (!taskQueue.isEmpty()) { // 如果队列不为空，取出元素
            return taskQueue.remove();
        }


        ScheduledTask scheduledTask = scheduledTaskQueue.peek(); // 如果taskQueue为空，就等待定时任务或者新的任务
        long timeoutNanos;
        if (scheduledTask == null) {
            timeoutNanos = DEFAULT_TIMEOUT;
        } else {
            timeoutNanos = scheduledTask.deadlineNanos() - currentNanos + 5000000;
        }

        try {
            Runnable task = taskQueue.poll(timeoutNanos, TimeUnit.NANOSECONDS);
            if (task != null) {
                return task;
            }
        } catch (InterruptedException e) {
            // 不执行操作
        }
    }
}

private void run() {
    for (;;) {
        try {
            Runnable task = takeTask();
            if (task != WAKEUP_TASK) {
                safeExecute(task);
            }
        } catch (Throwable t) {
            // 运行任务出错
        }

        if (isShutdown()) { // 如果状态为SHUTDOWN,退出循环
            return;
        }
    }
}
```

#### 5. 从启动到结束的状态转换
1. 当开始第一个任务时，启动执行线程，状态切换为STARTED
2. 当执行shutdown方法时，将状态切换为SHUTDOWN
3. 当执行完所有任务时，将状态切换为TEMINATE

```java
private void startThread() {
    int state = this.state;
    if (state == NOT_START && STATE_UPDATER.compareAndSet(this, state, STARTED)) {
        try {
            doStartThread();
        } catch (Throwable t) {
            // 启动失败，重置状态
            STATE_UPDATER.set(this, NOT_START); // 可能会擦除shutdown的状态
        }
    }
}


private void doStartThread() {
    executor.execute(() -> {
        SingleThreadExecutor.this.thread = Thread.currentThread(); //将线程放入SingleThreadExecutor中
        try {
            SingleThreadExecutor.this.run();
        } catch (Throwable t) {
            // 抛出异常
        } finally {
            for (;;) { // 确保状态为SHUTDOWN
                int oldState = this.state;
                if (isShutdown(oldState)
                        || STATE_UPDATER.compareAndSet(SingleThreadExecutor.this, oldState, SHUTDOWN)) {
                    break;
                }
            }

            try {
                runAllTasksFromQueue(taskQueue); // 将最后的任务执行结束
                cancelAllScheduledTask(scheduledTaskQueue);
            } finally {
                STATE_UPDATER.set(SingleThreadExecutor.this, TERMINATED);
            }
        }
    });
}

@Override
public void shutdown() {
    if (isShutdown()) {
        return;
    }
    for (;;) {
        int oldState = this.state;
        if (isShutdown(oldState)) {
            return;
        }
        if (STATE_UPDATER.compareAndSet(this, oldState, SHUTDOWN)) {
            ensureThreadStart(oldState); // 确保任务线程已经启动，这样可以正常执行完关闭流程
            wakeup(); // 避免在takeTask方法中死循环
            break;
        }
    }
}
```

### 完整代码
```java
package com.hewutao.model.executor;

import java.util.PriorityQueue;
import java.util.Queue;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

public class SingleThreadExecutor implements EventExecutor {
    private static final int NOT_START = 1;
    private static final int STARTED = 2;
    private static final int SHUTDOWN = 3;
    private static final int TERMINATED = 4;
    private static final AtomicIntegerFieldUpdater<SingleThreadExecutor> STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(SingleThreadExecutor.class, "state");

    private static final Runnable WAKEUP_TASK = () -> {};

    // 如果没有任务也没有定时任务，应该等待多长时间
    private static final long DEFAULT_TIMEOUT = TimeUnit.MILLISECONDS.toNanos(100);

    private final BlockingQueue<Runnable> taskQueue; // 执行的任务
    private final Queue<ScheduledTask> scheduledTaskQueue; // 定时任务
    private final Executor executor; // 执行器

    private volatile int state = NOT_START; // 执行器的状态

    private Thread thread; // 执行任务的线程

    public SingleThreadExecutor(Executor executor, int maxPendingTasks) {
        if (executor == null) {
            throw new NullPointerException("executor");
        }
        this.executor = executor;
        this.taskQueue = newBlockingQueue(maxPendingTasks);
        this.scheduledTaskQueue = newPriorityQueue();
    }

    private BlockingQueue<Runnable> newBlockingQueue(int maxPendingTasks) {
        return new LinkedBlockingQueue<>(maxPendingTasks);
    }

    private Queue<ScheduledTask> newPriorityQueue() {
        return new PriorityQueue<>();
    }

    @Override
    public void execute(Runnable task) {
        if (isShutdown() || !taskQueue.offer(task)) {
            reject(task);
        }
        if (!inLoop()) {
            startThread(); // 开启线程

            if (isShutdown()) { // 检查是否关闭，如果关闭就移除任务
                try {
                    if (taskQueue.remove(task)) {
                        reject(task);
                    }
                } catch (Exception e) {
                    //
                }
            }
        }
    }

    private void startThread() {
        int state = this.state;
        if (state == NOT_START && STATE_UPDATER.compareAndSet(this, state, STARTED)) {
            try {
                doStartThread();
            } catch (Throwable t) {
                // 启动失败，重置状态
                STATE_UPDATER.set(this, NOT_START); // 可能会擦除shutdown的状态
            }
        }
    }


    private void doStartThread() {
        executor.execute(() -> {
            SingleThreadExecutor.this.thread = Thread.currentThread(); //将线程放入SingleThreadExecutor中
            try {
                SingleThreadExecutor.this.run();
            } catch (Throwable t) {
                // 抛出异常
            } finally {
                for (;;) { // 确保状态为SHUTDOWN
                    int oldState = this.state;
                    if (isShutdown(oldState)
                            || STATE_UPDATER.compareAndSet(SingleThreadExecutor.this, oldState, SHUTDOWN)) {
                        break;
                    }
                }

                try {
                    runAllTasksFromQueue(taskQueue); // 将最后的任务执行结束
                    cancelAllScheduledTask(scheduledTaskQueue);
                } finally {
                    STATE_UPDATER.set(SingleThreadExecutor.this, TERMINATED);
                }
            }
        });
    }

    private void run() {
        for (;;) {
            try {
                Runnable task = takeTask();
                if (task != WAKEUP_TASK) {
                    safeExecute(task);
                }
            } catch (Throwable t) {
                // 运行任务出错
            }

            if (isShutdown()) {
                return;
            }
        }
    }

    /**
     * 从两个队列中取出元素，返回值不为null
     */
    private Runnable takeTask() {
        for (;;) {
            long currentNanos = System.nanoTime();
            fetchReadyScheduledTask(currentNanos); // 将准备好的定时任务，放到taskQueue中

            if (!taskQueue.isEmpty()) { // 如果队列不为空，取出元素
                return taskQueue.remove();
            }


            ScheduledTask scheduledTask = scheduledTaskQueue.peek(); // 如果taskQueue为空，就等待定时任务或者新的任务
            long timeoutNanos;
            if (scheduledTask == null) {
                timeoutNanos = DEFAULT_TIMEOUT;
            } else {
                timeoutNanos = scheduledTask.deadlineNanos() - currentNanos + 5000000;
            }

            try {
                Runnable task = taskQueue.poll(timeoutNanos, TimeUnit.NANOSECONDS);
                if (task != null) {
                    return task;
                }
            } catch (InterruptedException e) {
                // 不执行操作
            }
        }
    }

    /**
     * 将准备好的定时任务转移到taskQueue中
     * @param currentNanos 当前时间
     */
    private void fetchReadyScheduledTask(long currentNanos) {
        ScheduledTask task;
        while ((task = scheduledTaskQueue.peek()) != null
                && task.deadlineNanos() <= currentNanos // 存在定时任务，并准备好
                && taskQueue.offer(task)) { // 如果添加失败，说明队列满了，就不必再添加
            scheduledTaskQueue.remove();
        }
    }

    /**
     * 执行队列中的任务
     */
    private static void runAllTasksFromQueue(Queue<Runnable> queue) {
        while (!queue.isEmpty()) {
            Runnable task = queue.remove();
            safeExecute(task);
        }
    }

    /**
     * 取消队列中定时任务
     */
    private static void cancelAllScheduledTask(Queue<ScheduledTask> queue) {
        queue.clear();
    }



    @Override
    public void schedule(Runnable task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (delay <= 0) {
            throw new IllegalArgumentException("delay: " + delay + " (expected: > 0)");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }

        final ScheduledTask scheduledTask = new ScheduledTask(task, delay, unit);
        if (inLoop()) {
            scheduledTaskQueue.offer(scheduledTask);
        } else {
            // 可能出现异常，所以还是需要future跟踪任务的情况
            execute(() -> scheduledTaskQueue.offer(scheduledTask));
        }
    }

    @Override
    public void shutdown() {
        if (isShutdown()) {
            return;
        }
        for (;;) {
            int oldState = this.state;
            if (isShutdown(oldState)) {
                return;
            }
            if (STATE_UPDATER.compareAndSet(this, oldState, SHUTDOWN)) {
                ensureThreadStart(oldState); // 确保任务线程已经启动，这样可以正常执行完关闭流程
                wakeup();
                break;
            }
        }
    }

    /**
     * 唤醒线程
     */
    private void wakeup() {
        taskQueue.offer(WAKEUP_TASK);
    }

    private void ensureThreadStart(int oldState) {
        if (oldState == NOT_START) {
            try {
                doStartThread();
            } catch (Throwable t) {
                STATE_UPDATER.set(this, TERMINATED); // 如果启动失败直接将state设置为terminate
            }
        }
    }


    /**
     * 当前线程是否为执行线程
     */
    private boolean inLoop() {
        return thread == Thread.currentThread();
    }

    /**
     * 拒绝任务
     */
    private void reject(Runnable task) {
        throw new RejectedExecutionException();
    }

    @Override
    public boolean isShutdown() {
        return isShutdown(state);
    }

    private boolean isShutdown(int state) {
        return state >= SHUTDOWN;
    }

    @Override
    public boolean isTerminated() {
        return state == TERMINATED;
    }


    private static class ScheduledTask implements Comparable<ScheduledTask>, Runnable {
        private final Runnable task;
        private final long deadlineNanos;

        private ScheduledTask(Runnable task, long timeout, TimeUnit unit) {
            this.task = task;
            this.deadlineNanos = System.nanoTime() + unit.toNanos(timeout);
        }

        @Override
        public void run() {
            task.run();
        }

        @Override
        public int compareTo(ScheduledTask o) {
            return Long.compare(deadlineNanos, o.deadlineNanos);
        }

        private long deadlineNanos() {
            return deadlineNanos;
        }
    }

    private static void safeExecute(Runnable task) {
        try {
            task.run();
        } catch (Throwable t) {
            //
        }
    }
}

```