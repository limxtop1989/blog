---
title: 线程池
date: 2023-07-27 23:10:17
tags: Thread Pool，线程池
categories:
- [Java]
---

本文的目的为掌握线程池的正确使用。如果英语水平不错，仅仅看类注释，半个小时就可以初步掌握如何正确使用了，但如果读懂源码相关的实现，则不仅可以辅助加深对注释描述的理解，也可以学习好的代码设计和实现。

# Overview
An ExecutorService that executes each submitted task using one of possibly several pooled threads, normally configured using Executors factory methods.

## Core and maximum pool sizes
1. 在方法 `execute(Runnable)` 中提交新任务时，如果正在运行的线程少于 `corePoolSize`，即使其他工作线程处于空闲状态，也会创建一个新线程来处理请求。
2. 线程数达到`corePoolSize`后，新来的任务进入队列，等待 core 线程处理。
3. 当队列满了以后，又有新任务提交，在`corePoolSize`外，继续创建新线程。
4. 当队列满 & 线程数达到 `maximumPoolSize`，提交的新任务走 Rejected tasks 流程。

文字描述或许干枯难记，当阅读源码后，回来总结这些规则时，自己应该依据这些规则，就生产者线程（client）往线程池提交任务时，线程池如何分配任务给core thread，任务队列，maximum thread，在脑海里构建一张动态序列图。这样可以更好的帮助理解和记忆。

> On-demand construction
> By default, even core threads are initially created and started only when new tasks arrive, but this can be overridden dynamically using method `prestartCoreThread` or `prestartAllCoreThreads`. You probably want to prestart threads if you construct the pool with a non-empty queue.

## Keep-alive times
超过`corePoolSize`的线程，当他们空闲时长超过 `getKeepAliveTime(TimeUnit)`，会terminate。通过` allowCoreThreadTimeOut(boolean)`可以设置这个超时策略适用于 core 线程。

## Queuing
1. Direct handoffs：需要无边界的 `maximumPoolSizes`，避免没有线程，导致新提交的任务被拒绝。
> Direct handoffs. A good default choice for a work queue is a SynchronousQueue that hands off tasks to threads without otherwise holding them. Here, an attempt to queue a task will fail if no threads are immediately available to run it, so a new thread will be constructed. This policy avoids lockups when handling sets of requests that might have internal dependencies. Direct handoffs generally require unbounded maximumPoolSizes to avoid rejection of new submitted tasks. This in turn admits the possibility of unbounded thread growth when commands continue to arrive on average faster than they can be processed.
2. Unbounded queues: `LinkedBlockingQueue` 没有容量限制，`maximumPoolSize`不会生效，适用于任务彼此独立。
>Unbounded queues. Using an unbounded queue (for example a LinkedBlockingQueue without a predefined capacity) will cause new tasks to wait in the queue when all corePoolSize threads are busy. Thus, no more than corePoolSize threads will ever be created. (And the value of the maximumPoolSize therefore doesn't have any effect.) This may be appropriate when each task is completely independent of others, so tasks cannot affect each others execution; for example, in a web page server. While this style of queuing can be useful in smoothing out transient bursts of requests, it admits the possibility of unbounded work queue growth when commands continue to arrive on average faster than they can be processed.
3. Bounded queues： ` ArrayBlockingQueue`  是前两种策略的折衷，队列大小和最大线程数大小彼此折衷。
> Queue sizes and maximum pool sizes may be traded off for each other: Using large queues and small pools minimizes CPU usage, OS resources, and context-switching overhead, but can lead to artificially low throughput. If tasks frequently block (for example if they are I/O bound), a system may be able to schedule time for more threads than you otherwise allow. Use of small queues generally requires larger pool sizes, which keeps CPUs busier but may encounter unacceptable scheduling overhead, which also decreases throughput.


# Source code
1. 不要通篇顺序阅读，需要分层，跳过干扰次要代码，重点阅读想要懂的逻辑部分。由于我们现在是要读懂线程池是如何管理线程和任务的，所以，线程池的状态等代码都不必深入理解。
```java java.util.concurrent.ThreadPoolExecutor#execute
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    // case 1: 线程数少于corePoolSize，新建core 线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // case 2: 线程数已经到达corePoolSize，加入队列处理
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // case 3: 队列满了，加入新任务失败，新建(core, maximum] 区间的线程来处理
    else if (!addWorker(command, false))
        reject(command);
}
```
# Case 1:  addWorker(command, true)
```java java.util.concurrent.ThreadPoolExecutor#addWorker
private boolean addWorker(Runnable firstTask, boolean core) {
    // 状态判断，大概理解即可，并非重点 begin
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
// 状态判断，大概理解即可，并非重点 end
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 新建一个worker thread，并将第一个task 传入
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 将worker 加入worker 列表，他们循环从任务队列取任务执行
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
            // 启动worker thread，开始循环获取，执行任务
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

## new Worker
``` java java.util.concurrent.ThreadPoolExecutor.Worker#Worker
/**
    * Creates with given first task and thread from ThreadFactory.
    * @param firstTask the first task (null if none)
    */
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```
初始化Worker 实例，其中实例化 `firstTask` & `thread` 成员变量。Worker实现 `Runnable` interface，在初始化thread时，作为构造器参数传入，作为Thread.target的值。Worker 继承 `AbstractQueuedSynchronizer`，这是Java 并发包的核心类，使用CAS操作实现锁，将放在其他博文展开分析。
```java java.util.concurrent.ThreadPoolExecutor#ThreadPoolExecutor(int, int, long, TimeUnit, BlockingQueue<Runnable>)
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            Executors.defaultThreadFactory(), defaultHandler);
}
```
ThreadPool 默认的ThreadFactory 是Executors.defaultThreadFactory()，看下其newThread代码，便知worker实例是Thread的 target了。
```java java.util.concurrent.Executors.DefaultThreadFactory#DefaultThreadFactory
public Thread newThread(Runnable r) {
    Thread t = new Thread(group, r,
                            namePrefix + threadNumber.getAndIncrement(),
                            0);
    if (t.isDaemon())
        t.setDaemon(false);
    if (t.getPriority() != Thread.NORM_PRIORITY)
        t.setPriority(Thread.NORM_PRIORITY);
    return t;
}
```
当Thread start后，Thread.run 会执行，target.run也就是Worker.run会被调用。
```java java.lang.Thread#Thread(java.lang.ThreadGroup, java.lang.Runnable, java.lang.String, long)
private void init(ThreadGroup g, Runnable target, String name,
                    long stackSize, AccessControlContext acc,
                    boolean inheritThreadLocals) {
// ignore irrelevant codes
    this.target = target;
// ignore irrelevant codes
}
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```            

## workers.add
```java java.util.concurrent.ThreadPoolExecutor#workers
/**
 * Set containing all worker threads in pool. Accessed only when
 * holding mainLock.
 */
private final HashSet<Worker> workers = new HashSet<Worker>();
```    
## t.start
```java java.util.concurrent.ThreadPoolExecutor.Worker#run
/** Delegates main run loop to outer runWorker  */
public void run() {
    runWorker(this);
}
```
```java java.util.concurrent.ThreadPoolExecutor#runWorker
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
            // 初始化时的第一个任务先执行，后续都从任务队列获取任务执行
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```    
# Case 2: workQueue.offer(command)
将任务塞入队列，待线程池的线程轮询获取任务执行。队列有多种实现，主要区分Array & Link
1. ArrayBlockingQueue 有固定边界，当任务加满后，workQueue.offer(command) 会返回false，进入线程 (core, maximum] 扩张流程。
2. LinkedBlockingQueue 默认没有边界，所以workQueue.offer(command) 一般返回true，即任务成功加入队列。除非初始化时，设置capacity。

# Case 3: addWorker(command, false)
与addWorker(command, true) 类似，当线程数没到达maximum时，创建新线程，返回true，否则不创建新线程，返回false，当前任务交由 `RejectedExecutionHandler`处理，进入reject流程。
``` java java.util.concurrent.ThreadPoolExecutor#addWorker

private boolean addWorker(Runnable firstTask, boolean core) {
    int wc = workerCountOf(c);
    if (wc >= CAPACITY ||
        wc >= (core ? corePoolSize : maximumPoolSize))
        return false;
}
```                    
