---
title: Android 触摸事件派发
date: 2022-08-19 22:42:18
categories:
- [Android]
tags: Android, Touch Event Dispatch, 触摸事件派发
---

# Return true at onTouchEvent of the child view during action down
![touch down](https://s3.uuu.ovh/imgs/2023/12/11/13497e866aa01232.jpeg)
The subsequent move and up actions will be dispatched to the child view directly, but the parent views still have a chance to intercept the actions.
![](https://s3.uuu.ovh/imgs/2023/12/11/ddbbaf779e52b43f.jpeg)

Once the parent views intercept the touch event during move action by return true at `onInterceptToucheEvent`  

![](https://s3.uuu.ovh/imgs/2023/12/11/8596b7d5b40abf88.jpeg)
a cancel event is generated and dispatched to the intercepted child view, in the mean while, the subsequent move and up actions will be dispatched to parent view directly rather than the child view.
![](https://s3.uuu.ovh/imgs/2023/12/11/afccd4afe1ce2f1c.jpeg)

# Return false at onTouchEvent the child view during action down
![](https://s3.uuu.ovh/imgs/2023/12/11/499e016d52ba2a02.jpeg)
The subsequent move and up actions will not be dispatched to the child view any more.
![](https://s3.uuu.ovh/imgs/2023/12/11/770400864dc19db8.jpeg)



