---
title: Android消息循环
date: 2023-02-04 15:59:56
tags: Android, message loop, 消息循环
---

# 题记
本文又几年前给同事们培训准备的资料整理而来，只是凭借印象，粗略记录框架原理和要点，需要新人花不少功夫自学探索。待闲暇时，待我重新阅读源码，再详细展开。本文需要具备英语阅读能力，如果尚缺，可以阅读[英语自学指南](https://bewaters.me/limxtop/2021/08/18/English-introduction/)。

# 用法
线程是不具备消息循环的，需要调用`Looper.prepare()` & `Looper.loop()`才开启消息循环。系统已经在`ActivityThread` 帮忙主线程开启了消息循环，代码如下。
```java
public static void main(String[] args) {
        Looper.prepareMainLooper();
        ActivityThread thread = new ActivityThread();
          if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
所以，主线程是不可以调用`looper.quit()`，不然抛出`new RuntimeException("Main thread loop unexpectedly exited")`，而WorkerThread如果开启消息循环，不再需要时，得调用`looper.quit()`，否则内存泄漏。

# 类图
![](https://s3.bmp.ovh/imgs/2023/02/04/e2a2de9b66a113c6.png)
每个线程都有自己的消息循环机制和队列，实现的关键是将Looper对象保存在当前`Thread`的`ThreadLocalMap`，<key, value> = <Looper.sThreadLocal, Looper>.

android.os.Looper#prepare(boolean)
```java
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
java.lang.ThreadLocal#set
```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```
# Overview
消息队列的消息有指定执行时间，消息循环机制是如何做到准时处理消息的？
1. 消息循环机制初始化时，在native层，监听`mWakeEventFd。
2. 消息循环开始时，从消息队列里取消息，自然取不到，就在native 等待`mWakeEventFd`事件
3. handler 发送消息，并进入消息队列，native 层 在mWakeEventFd 写入数据，唤醒第二步的等待事件。
4. 第二步从等待中唤醒，读取消息队列的消息，如果消息执行时间是现在，则处理消息。否则计算执行时间和当前的时间差`nextPollTimeoutMillis`，重新插入消息队列，并在native层等待，等待超时参数是`nextPollTimeoutMillis`

# nextPollTimeoutMillis
`nextPollTimeoutMillis` 看似不起眼，其实是很重要的变量，它有三种类型的值，分别代表不同的含义。
1. -1：消息队列没消息，native 使用`wait()`
2. 0：（TODO： ）
3. 自然数：消息队列最早执行的消息（队列头部的消息）在未来，当前没有消息处理，可以歇息。native 使用 `wait(int timeout)` ，超时后自动从等待中返回，无需write事件唤醒。

android.os.MessageQueue#next
```java
        int nextPollTimeoutMillis = 0;
        for (;;) {
            nativePollOnce(ptr, nextPollTimeoutMillis);
            // Try to retrieve the next message.  Return if found.
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    return msg;// Got a message.
                }
             } else {
                    nextPollTimeoutMillis = -1;// No more messages.
              }
            // A new message could have been delivered, so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
```

# Prepare
![](https://s3.bmp.ovh/imgs/2023/02/04/4b24b7075c896855.png)
使用Epoll API，在mWakeEventFd注册监听

# Loop And Handle Message
![](https://s3.bmp.ovh/imgs/2023/02/04/c8230250364e5395.png)
此时消息队列还没有消息，在`epoll_wait`等待

# Insert Message
![](https://s3.bmp.ovh/imgs/2023/02/04/4b347d81a5ccef35.png)
往消息队列插入消息后，往`mWakeEventFd`写入一个整数，唤醒`poll_wait`，从而loop循环中取出消息，接着处理Message。

# 线程间通信
![](https://s3.bmp.ovh/imgs/2023/02/04/ef7b63e078f95c70.png)
通信线程双方，持有对方的handler实例的引用，通过handler.sendMessage往对方线程的消息队列发送消息，如此两个线程就可以彼此发送消息，实现通信。

# 消息作用域
![](https://s3.bmp.ovh/imgs/2023/02/07/be19a511e4b0ac2b.png)
哪个Handler 发送的消息，最后出消息队列，仍由该Handler处理，所以不同的Handler实例，可以有相同的What，而不会相互干扰。具体实现是`handler.sendMessage(message)`时，`handler`对象保存在`message.target`字段。

# ThreadLocal

# 雷区


# Reference
[1] https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642
[2 ] https://man7.org/linux/man-pages/man7/epoll.7.html
[3] The Design of the UNIX Operating System
[4] /base/core/java/android/os/MessageQueue.java
[5] /frameworks/base/core/jni/android_os_MessageQueue.cpp
[6] /system/core/libutils/Looper.cpp
