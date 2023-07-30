---
title: Java 并发框架 AbstractQueuedSynchronizer
date: 2023-07-30 08:58:02
tags:
categories:
- [Java]
---
![](https://s3.uuu.ovh/imgs/2023/07/30/84de061a940ab561.png)
关于 AbstractQueuedSynchronizer 的源码分析，我以 ReentrantLock 为例，类图如上。

> A reentrant mutual exclusion Lock with the same basic behavior and semantics as the implicit monitor lock accessed using synchronized methods and statements, but with extended capabilities.

> A ReentrantLock is owned by the thread last successfully locking, but not yet unlocking it. A thread invoking lock will return, successfully acquiring the lock, when the lock is not owned by another thread. The method will return immediately if the current thread already owns the lock. This can be checked using methods isHeldByCurrentThread, and getHoldCount.

Lock接口定义了关于Lock的三个主要操作：

1. lock();
2. unLock();
3. newCondition();

`ReentrantLock` 实现 `Lock` 接口，但实现的细节全都委派给内部类 `Sync` 的具体实例。其中值得注意的是，创建 `ReentrantLock` 实例时，可以传递 `fair` 参数，指明实现的是公平锁还是非公平锁，默认为非公平锁。 

总体流程如图
![](https://s3.uuu.ovh/imgs/2023/07/30/602b88085d477f84.png)


```java
/**
    * Creates an instance of {@code ReentrantLock}.
    * This is equivalent to using {@code ReentrantLock(false)}.
    */
public ReentrantLock() {
    sync = new NonfairSync();
}

/**
    * Creates an instance of {@code ReentrantLock} with the
    * given fairness policy.
    *
    * @param fair {@code true} if this lock should use a fair ordering policy
    */
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```    
公平锁指线程 `lock` 时得按顺序排队，等待队列为空时才抢占锁，否则进入队列排队等待。

# Thread B 加锁失败，进入等待
```java java.util.concurrent.locks.ReentrantLock.FairSync#lock
final void lock() {
    acquire(1);
}
```        
```java java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire
/**
    * Acquires in exclusive mode, ignoring interrupts.  Implemented
    * by invoking at least once {@link #tryAcquire},
    * returning on success.  Otherwise the thread is queued, possibly
    * repeatedly blocking and unblocking, invoking {@link
    * #tryAcquire} until success.  This method can be used
    * to implement method {@link Lock#lock}.
    *
    * @param arg the acquire argument.  This value is conveyed to
    *        {@link #tryAcquire} but is otherwise uninterpreted and
    *        can represent anything you like.
    */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
先`tryAcquire` 尝试获得锁，尝试失败，则 `acquireQueued` 。在 AbstractQueuedSynchronizer `tryAcquire` 是模版方法，强制子类实现
```java java.util.concurrent.locks.ReentrantLock.FairSync#tryAcquire
/**
    * Fair version of tryAcquire.  Don't grant access unless
    * recursive call or no waiters or is first.
    */
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 没有线程持有锁，(线程成功获得锁时，会将 state 递增 acquires)
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 已经有线程持有锁，判断该锁拥有者是不是当前线程，是则递增 acquires 直接返回 true 表示成功加锁。
        // 线程加锁后，可以继续加锁，是故可重入锁
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 该锁被其他线程持有，返回false， 加锁失败，执行 acquireQueued 进入等待队列。
    return false;
}
```
1. 获得锁的逻辑很清晰简单，获取状态，当 state == 0 时，意味着没有线程持有该锁，再判断队列没有前驱并且用CAS操作更新state，使其大于0，如果更新操作成功，当前线程获得锁，在setExclusiveOwnerThread方法里将exclusiveOwnerThread指向当前线程。(compare expectedValue with the state value in the memory, and set the state value only if they are equal, which is atomic operation.由于 state & exclusiveOwnerThread 是共享变量，此代码块是critical section(临界区)，所以比较和更新必须是原子操作，防止线程A比较更新中间，线程B更新state，导致线程AB同时获得锁)
2. 如果state > 0, 意味着已经有线程持有该锁，但不必灰心，代码进一步判断当前锁的持有者是否是当前线程，如果是，则累加锁数量。
3. 我们重点来看加锁失败，加入等待队列的情况。
```java java.util.concurrent.locks.AbstractQueuedSynchronizer#addWaiter
/**
    * Creates and enqueues node for current thread and given mode.
    *
    * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
    * @return the new node
    */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```    
将当前线程和锁模式 `exclusive` 封装在 `Node` 里，加入队列末尾。
```java java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued

/**
    * Acquires in exclusive uninterruptible mode for thread already in
    * queue. Used by condition wait methods as well as acquire.
    *
    * @param node the node
    * @param arg the acquire argument
    * @return {@code true} if interrupted while waiting
    */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 等待队列有前驱等待节点，才进入此方法，但此时可能前驱节点已经移除，
            // 是故这里判断等待队列为空，重新尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 重点在这里
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
先简略理解下 `shouldParkAfterFailedAcquire`，如果前驱节点
1. 是Signal，则返回 true，执行 parkAndCheckInterrupt。
2. 是Cancel，则移除Cancel的等待节点.
3. 将非Cancel的前驱节点，状态升级为Signal，下次循环进来，就是 case 1.

在这里，我们记住，当前等待线程的前驱节点，到最后是Signal状态。

```java java.util.concurrent.locks.AbstractQueuedSynchronizer#shouldParkAfterFailedAcquire

/**
    * Checks and updates status for a node that failed to acquire.
    * Returns true if thread should block. This is the main signal
    * control in all acquire loops.  Requires that pred == node.prev.
    *
    * @param pred node's predecessor holding status
    * @param node the node
    * @return {@code true} if thread should block
    */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
        return true;
    if (ws > 0) {
        /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```    

接下来，我们看 `parkAndCheckInterrupt` 则当前线程休眠，开始进入等待态（没什么特别的，就设置一个状态标记，告诉线程调度算法不用拿此线程参与调度计算），并检查中断状态。
```java java.util.concurrent.locks.AbstractQueuedSynchronizer#parkAndCheckInterrupt

/**
    * Convenience method to park and then check if interrupted
    *
    * @return {@code true} if interrupted
    */
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);// 这行代码执行完毕，线程进入休眠，等待其他线程调用 unpark 后，执行下一行的retrun
    return Thread.interrupted();
}
```
```java java.util.concurrent.locks.LockSupport#park(java.lang.Object)

/**
 * Disables the current thread for thread scheduling purposes unless the
 * permit is available.
 *
 * <p>If the permit is available then it is consumed and the call returns
 * immediately; otherwise
 * the current thread becomes disabled for thread scheduling
 * purposes and lies dormant until one of three things happens:
 *
 * <ul>
 * <li>Some other thread invokes {@link #unpark unpark} with the
 * current thread as the target; or
 *
 * <li>Some other thread {@linkplain Thread#interrupt interrupts}
 * the current thread; or
 *
 * <li>The call spuriously (that is, for no reason) returns.
 * </ul>
 *
 * <p>This method does <em>not</em> report which of these caused the
 * method to return. Callers should re-check the conditions which caused
 * the thread to park in the first place. Callers may also determine,
 * for example, the interrupt status of the thread upon return.
 *
 * @param blocker the synchronization object responsible for this
 *        thread parking
 * @since 1.6
 */
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
```    
到这里，Thread B 线程加锁失败，进入等待队列，休眠，等待其他线程释放锁，通知唤醒等待队列的线程，重新竞争获得锁。我们开始看其他线程释放锁流程。
# Thread B 释放锁
```java java.util.concurrent.locks.ReentrantLock#unlock

/**
 * Attempts to release this lock.
 *
 * <p>If the current thread is the holder of this lock then the hold
 * count is decremented.  If the hold count is now zero then the lock
 * is released.  If the current thread is not the holder of this
 * lock then {@link IllegalMonitorStateException} is thrown.
 *
 * @throws IllegalMonitorStateException if the current thread does not
 *         hold this lock
 */
public void unlock() {
    sync.release(1);
}
```
```java java.util.concurrent.locks.AbstractQueuedSynchronizer#release

/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
尝试释放锁，然后unpark 锁等待队列的第一个线程，使其进入等待调度状态。
```java java.util.concurrent.locks.ReentrantLock.Sync#tryRelease
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
```java java.util.concurrent.locks.AbstractQueuedSynchronizer#unparkSuccessor

/**
* Wakes up node's successor, if one exists.
*
* @param node the node
*/
private void unparkSuccessor(Node node) {
    /*
    * If status is negative (i.e., possibly needing signal) try
    * to clear in anticipation of signalling.  It is OK if this
    * fails or if status is changed by waiting thread.
    */
    int ws = node.waitStatus;
    if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);

    /*
    * Thread to unpark is held in successor, which is normally
    * just the next node.  But if cancelled or apparently null,
    * traverse backwards from tail to find the actual
    * non-cancelled successor.
    */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
        s = t;
    }
    if (s != null)
    LockSupport.unpark(s.thread);
}
```
传入h 指向的是哨兵头节点，找到锁等待队列里第一个 `waitStatus <= 0` 即 SIGNAL CONDITION PROPAGATE 三者之一的节点，并 unpark 唤醒节点里的线程。
当上文 Thread B 加锁失败，进入等待锁队列的节点
1. 是这个头节点，加锁线程被唤醒，thread schedule 分配它运行后，它从park中返回，并检测记录 interrupted，进入下一次循环，此时它的前驱就是head 哨兵节点，再次执行 `tryAcquire` 流程。
2. 不是头节点，那么等待前面的节点都纷纷通过 `tryAcquire` 获得锁，然后释放锁后，文中上一个加锁的线程所在节点最终会成为头节点。

获得锁成功后，返回 interrupted 中断状态，如果该线程是 interrupted 而进入等待态，则返回后重置还原 interrupted 状态。

线程从休眠进入等待态有三种cases 见 `java.util.concurrent.locks.LockSupport#park(java.lang.Object)`：
1. Some other thread invokes unpark with the current thread as the target; or
2. Some other thread interrupts the current thread; or
3. The call spuriously (that is, for no reason) returns.

``` java java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```