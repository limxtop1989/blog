---
title: Semaphore
date: 2023-07-29 17:03:58
tags: semaphore，java 并发，信号量
---

计数方式实现的旗语，一个旗语维护一组概念上的许可。每个线程 acquire 都会在必要时阻塞，直到有许可证可用，然后获取它，许可数量减一。每次释放都会增加一个许可，从而有可能释放阻塞在 acquire 的线程。不过， Semaphore 并不使用实际的许可对象；它只是对可用数量进行计数，并采取相应的行动。

如此 Semaphores 通常用于限制可以访问某些（物理或逻辑）资源的线程数量。也就是说，当你希望只有 permits 个线程可以并发访问这些资源是，通过 Semaphore(int permits, boolean fair) 实例化 permits 个许可资源，线程 aquire 许可后，获取资源，完成后，release 释放许可回许可池。

>A counting semaphore. Conceptually, a semaphore maintains a set of permits. Each acquire blocks if necessary until a permit is available, and then takes it. Each release adds a permit, potentially releasing a blocking acquirer. However, no actual permit objects are used; the Semaphore just keeps a count of the number available and acts accordingly.
>Semaphores are often used to restrict the number of threads than can access some (physical or logical) resource.

在 aquire & release 之前，没有同步锁被持有，
> Note that no synchronization lock is held when acquire is called as that would prevent an item from being returned to the pool. The semaphore encapsulates the synchronization needed to restrict access to the pool, separately from any synchronization needed to maintain the consistency of the pool itself.

与 Lock 的实现不同， semaphore release permit 的线程，不需要以 aquire 该 permit 为前提。
>A semaphore initialized to one, and which is used such that it only has at most one permit available, can serve as a mutual exclusion lock. This is more commonly known as a binary semaphore, because it only has two states: one permit available, or zero permits available. When used in this way, the binary semaphore has the property (unlike many java.util.concurrent.locks.Lock implementations), that the "lock" can be released by a thread other than the owner (as semaphores have no notion of ownership). This can be useful in some specialized contexts, such as deadlock recovery.