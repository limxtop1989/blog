---
title: CountDownLatch
date: 2023-07-29 15:30:00
tags: CountDownLatch, 并发，多线程
categories:
- [Java]
---

## Overview
一种同步辅助工具，允许一个或多个线程等待其他线程正在执行的一组操作完成。
> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

CountDownLatch 以给定的计数初始化。await 方法会阻塞，直到当前计数因调用 CountDown 方法而归零，之后所有等待的线程都会被释放，并且 await 的任何后续调用都会立即返回。这是一种一次性现象--计数无法重置。如果需要重置计数的版本，可以考虑使用 CyclicBarrier。
> A CountDownLatch is initialized with a given count. The await methods block until the current count reaches zero due to invocations of the countDown method, after which all waiting threads are released and any subsequent invocations of await return immediately. This is a one-shot phenomenon -- the count cannot be reset. If you need a version that resets the count, consider using a CyclicBarrier.
CountDownLatch 是一种多功能同步工具，可用于多种用途。初始化计数为 1 的 CountDownLatch 可用作简单的开/关锁存器或门：所有调用 await 的线程都在门上等待，直到调用 countDown 的线程打开它。初始化为 N 的 CountDownLatch 可用于让一个线程等待，直到 N 个线程完成某个操作，或某个操作完成 N 次。
> A CountDownLatch is a versatile synchronization tool and can be used for a number of purposes. A CountDownLatch initialized with a count of one serves as a simple on/off latch, or gate: all threads invoking await wait at the gate until it is opened by a thread invoking countDown. A CountDownLatch initialized to N can be used to make one thread wait until N threads have completed some action, or some action has been completed N times.
CountDownLatch 的一个有用特性是，它并不要求调用 countDown 的线程在继续执行之前需要等待计数归零，也就是它调用 countDown 之后，可以继续一路高歌猛进，它只是阻止其他调用 await 的任何线程越过 await 继续执行，直到所有 await 的线程都能通过为止（即 countDown 为0）。
> A useful property of a CountDownLatch is that it doesn't require that threads calling countDown wait for the count to reach zero before proceeding, it simply prevents any thread from proceeding past an await until all threads could pass.

Sample usage: Here is a pair of classes in which a group of worker threads use two countdown latches:
The first is a start signal that prevents any worker from proceeding until the driver is ready for them to proceed;
The second is a completion signal that allows the driver to wait until all workers have completed.

``` java
 class Driver { // ...
   void main() throws InterruptedException {
     CountDownLatch startSignal = new CountDownLatch(1);
     CountDownLatch doneSignal = new CountDownLatch(N);

     for (int i = 0; i < N; ++i) // create and start threads
       new Thread(new Worker(startSignal, doneSignal)).start();

     doSomethingElse1();            // don't let run yet
     startSignal.countDown();      // let all threads proceed
     doSomethingElse2();
     doneSignal.await();           // wait for all to finish
   }
 }

 class Worker implements Runnable {
   private final CountDownLatch startSignal;
   private final CountDownLatch doneSignal;
   Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
     this.startSignal = startSignal;
     this.doneSignal = doneSignal;
   }
   public void run() {
     try {
       startSignal.await();
       doWork();
       doneSignal.countDown();
     } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }
```
![](https://s3.uuu.ovh/imgs/2023/07/30/47ebfd56d12f728c.png)
WorkerThread 初始化启动后， 在 `startSignal.await();` 集结等待，driver 在 doSomethingElse1 后， startSignal.countDown(); 释放门闩，所有等待线程从等待态，进入就绪态，纷纷执行 doWork() 与此同时， Driver 执行 doneSignal.await() 等所有 WorkerThread 完成工作，都执行 doneSignal.countDown();  
待计数器归零，Driver 线程从 await 中返回，从等待态进入就绪态，

> Another typical usage would be to divide a problem into N parts, describe each part with a Runnable that executes that portion and counts down on the latch, and queue all the Runnables to an Executor. When all sub-parts are complete, the coordinating thread will be able to pass through await. (When threads must repeatedly count down in this way, instead use a CyclicBarrier.)

``` java
 class Driver2 { // ...
   void main() throws InterruptedException {
     CountDownLatch doneSignal = new CountDownLatch(N);
     Executor e = ...

     for (int i = 0; i < N; ++i) // create and start threads
       e.execute(new WorkerRunnable(doneSignal, i));

     doneSignal.await();           // wait for all to finish
   }
 }

 class WorkerRunnable implements Runnable {
   private final CountDownLatch doneSignal;
   private final int i;
   WorkerRunnable(CountDownLatch doneSignal, int i) {
     this.doneSignal = doneSignal;
     this.i = i;
   }
   public void run() {
     try {
       doWork(i);
       doneSignal.countDown();
     } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }
 ```
 > Memory consistency effects: Until the count reaches zero, actions in a thread prior to calling countDown() happen-before actions following a successful return from a corresponding await() in another thread.