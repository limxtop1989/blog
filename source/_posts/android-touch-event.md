---
title: Android Touch Event Dispatch
date: 2022-08-19 22:42:18
tags: Android, Touch Event Dispatch, 触摸事件派发
---

# Return true at onTouchEvent of the child view during action down
![touch down](https://tva1.sinaimg.cn/large/e6c9d24egy1h5cgobq43vj20u40u0n18.jpg)
The subsequent move and up actions will be dispatched to the child view directly, but the parent views still have a chance to intercept the actions.
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5ch3isl8pj20u00uwdk6.jpg)

Once the parent views intercept the touch event during move action by return true at `onInterceptToucheEvent`  

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5chap23ivj20vs0u042q.jpg)
a cancel event is generated and dispatched to the intercepted child view, in the mean while, the subsequent move and up actions will be dispatched to parent view directly rather than the child view.
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5ch8wfx5cj20u00uctcw.jpg)

# Return false at onTouchEvent the child view during action down
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5ch5h9serj20wl0u0jvn.jpg)
The subsequent move and up actions will not be dispatched to the child view any more.
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h5ch6if7mdj20u00uyq73.jpg)



