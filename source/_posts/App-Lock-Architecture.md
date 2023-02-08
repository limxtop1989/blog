---
title: 应用锁架构
date: 2023-02-04 13:05:02
tags: App Lock, AMS, ActivityManagerService, 应用锁, 应用锁架构
---

# 需求描述
提供给指定应用加锁机制，应用处于上锁状态时，启动该应用需要做解锁操作，否则不仅用户不能与之交互，其应用界面也不可见，包括在SystemUI的Recent界面，也不可见其缩略图。

# 入职接手的第一版架构
ActivityThread handleActivityResume 发送广播给应用锁的接收器，启动UnlockService去查询数据库判断是否当前启动的Activity 所在应用是否是上锁状态。图我就不画了，那代码不值得投入时间画架构图。主要缺点有
1. 监听应用启动，判断是否上锁状态耗时，被锁应用已经启动完毕，用户可见可能长达一秒钟后，UnlockActivity 才启动覆盖在上面。
2. UnlockActivity & 上锁状态的Activity不在同一个ActivityTask，UnlockActivity 界面按back，只能强制显示Launcher，而不是上一个Activity。
3. 应用锁数据并没有提供API给SystemUI。

# 第二版重构架构
![](https://s3.bmp.ovh/imgs/2023/02/04/726cb4bd64b70c2f.png)
在AMS的AppLock 模块，绑定AppLock的Service，并保持连接，Activity的启动事件都通知给AppLock。AppLock 实现ContentProvider对外提供数据访问API，以此解决第一版1 & 3问题。 
遗留缺点：
1. 对于需要登陆等操作的互联网应用，会启动MainActivity后，判断是跳转登陆Activity，由于连续启动两个上锁状态的Activity，两个UnlockActivity也随之而来，屏幕闪缩。对此，可以单独为MainActivity 加入过滤列表，不检测上锁解决，但这并非是好的方案。

# 第三版重构架构
![](https://s3.bmp.ovh/imgs/2023/02/04/f55970d40056afff.png)
开机后，读取并监听应用锁的数据，（主要是应用是否是上锁状态），应用启动后，在AMS检测是否上锁状态，将Intent保留在IntentSender，原始Intent篡改为启动UnlockActivity的Intent，制定当前ActivityTask启动UnlockActivity，如此移花接木。解锁后，从Intent取出IntentSender 重新发送请求给AMS。至此，应用锁几乎完美无缺，久经测试同学的诸多考验。