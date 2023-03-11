---
title: Activity 转移类型检测
date: 2021-07-17 20:26:30
categories:
- [Android]
tags:
---

Requirement:：

应用内有些Activity数据敏感，需要区别对待，这里需要识别首次进入红色Activity集合内，在红色Activity集合内跳转，和跳出红色Activity集合三种情况。【当红色Activity集合是应用内的Activity集合时，就相当于首次进入应用，应用内界面跳转和退出应用的判定】

[![W1aZWR.png](https://z3.ax1x.com/2021/07/17/W1aZWR.png)](https://imgtu.com/i/W1aZWR)

算法描述：

1. 定义一个保存Activity Simple Name的集合。

2. Activity onResume时，将此Activity Simple Name加入集合。

3. 1. 当集合元素个数等于**1**时，为首次进入该应用打开的第一个Activity。
   2. 当集合元素个数等于**2**时，表示打开应用内的其他Activity。

4. Activity onStop时，将此Activity Simple Name移出集合。

5. 1. 当集合元素个数为**1**时，表示在应用内跳转。
   2. 当集合元素个数为**0**时，表示打开其他应用。



原理：

[![W1aVY9.png](https://z3.ax1x.com/2021/07/17/W1aVY9.png)](https://imgtu.com/i/W1aVY9)



由上图可知，Activity A启动Activity B时，执行顺序是Activity A's onPause() -> Activity B's onResume() -> Activity A's onStop()。应用内的Activity在执行 onResume() & onStop()时，我们规约将其Activity Simple Name加入&移出集合，而启动应用外的Activity我们没有能力如此操作。

1. 首次进入Activity集合，应用内的Activity onResume被调用；
2. 在Activity集合间跳转，应用内的Activity onResume & onStop被调用；
3. 跳出Activity集合，应用内的Activity onStop 被调用；



所以，启动应用内还是应用外的Activity在此产生差异，而差异让程序识别成为可能。

1. 当Activity B是**应用内Activity**时，集合内元素个数变迁如下：

   Activity A's onPause() -> Activity B's onResume() -> Activity A's onStop()

   **1**（Activity A）         **2**（Activity A&B）        **1**（Activity B）

2. 当Activity B是**应用外Activity**时，集合内元素个数变迁如下：

   Activity A's onPause() -> Activity B's onResume() -> Activity A's onStop()

   **1**（Activity A）         **1**（Activity A）              **0**（）
