---
title: Android 触摸事件派发
date: 2022-08-19 22:42:18
categories:
- [Android]
tags: Android, Touch Event Dispatch, 触摸事件派发
---

# Return true at onTouchEvent of the child view during action down
![touch down](https://p.ipic.vip/349h30.jpg)
The subsequent move and up actions will be dispatched to the child view directly, but the parent views still have a chance to intercept the actions.
![](https://p.ipic.vip/9g9w2u.jpg)

Once the parent views intercept the touch event during move action by return true at `onInterceptToucheEvent`  

![](https://p.ipic.vip/usy7yu.jpg)
a cancel event is generated and dispatched to the intercepted child view, in the mean while, the subsequent move and up actions will be dispatched to parent view directly rather than the child view.
![](https://p.ipic.vip/xyt8ic.jpg)

# Return false at onTouchEvent the child view during action down
![](https://p.ipic.vip/ytljka.jpg)
The subsequent move and up actions will not be dispatched to the child view any more.
![](https://p.ipic.vip/7eocw3.jpg)



