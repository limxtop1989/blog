---
title: Android 动画原理
date: 2022-07-24 22:42:18
categories:
- [Android]
tags: Android, Animation, Animator
math: true
---

# 动画原理

动画的本质是，动画对象随着时间，从初始值到最终值的连续变化。所以，定义一个动画有三个要素，动画时间即duration，动画初始和最终值。最基础原始的关系是动画时间和动画值成正比例（线性插值）。
$$
\frac{t}{duration} = \frac{AnimateValue}{end - start} \tag{1}
$$

![](https://p.ipic.vip/3vgwve.jpg)

# KeyFrame扩展

在Android内部实现上，会将初始值到最终值切分成诸多在动画时间轴均匀分布的关键帧KeyFrame，这个类有两个重要成员变量：

1. mFraction：当前关键帧对应动画时间与duration的比值。
2. mValue：在当前动画时间比时，动画对象的状态值。

抽象来看，公式$1$中，start & end 对应起始关键帧：KF(0, start)和结束关键帧: KF(1, end)。但如果只有这两个关键帧，物体就只能介于这两个关键帧的动画值之间线性变化，为了支持动画值可以挣脱这个局限，动画框架提供了关键帧的API，指定某关键帧（mFraction, mValue），其中mValue 可以在[start, end]之外。也可以提供关键帧计算函数，实现曲线运动轨迹而非线段。为此动画时间轴上t时，对应的动画值V计算方法扩展为：
$$
\frac{\frac{t}{duration} - preKF.mFraction}{nextKF.mFraction - prevKF.mFraction} = \frac{AnimateValue}{nextKF.value - prevKF.value} \tag{2}
$$

它根据$\frac{t}{duration}$和KeyFrame.mFraction相比较大小，计算位于哪两个KeyFrame区间，应用公式$1$。随着动画时间流逝，就可以在所有的KeyFrame应用公式$1$了。

| KeyFrame.mFraction & mValue均匀线性分布                      | KeyFrame.mFraction线性，mValue 曲线分布                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src="https://p.ipic.vip/xw8u0r.jpg" style="zoom:50%;" /> | <img src="https://p.ipic.vip/9vuhf0.jpg" style="zoom:50%;" /> |



注：右侧第四个红点挪到end右侧，动画轨迹就可以越出end的范围了。



# 变速扩展

为支持非匀速动画，使用Interpolator 将t篡改， $\frac{Interpolator(t)}{duration}$，转换成别的比例。
$$
\frac{\frac{Interpolator(t)}{duration} - preKF.mFraction}{nextKF.mFraction - prevKF.mFraction} = \frac{AnimateValue}{nextKF.value - prevKF.value} \tag{3}
$$

$Interpolator(t)\text{ where } t\in[0, 1]$其实就是一个函数，输入原始时间t，输出篡改后的时间，约束是Interpolator(0) = 0 && Interpolator(1) = 1。由于AnimateValue 是t的函数，变速动画成为可能，且容易实现。比如未引入Interpolator概念的公式，AnimateValue和t是等比关系，只能是匀速动画，是Interpolator(t) = t 这一特例，但它可以做的更多，比如匀加速，匀减速……具体是怎么变速，AnimateValue对t一阶求导可以得到速度，二阶求导可以得到加速度，这样描述应该可以更容易理解。

| 匀速运动 $Interpolator(t) = t$                               | 匀加速$Interpolator(t) = t^2$                                | 匀减速$Interpolator(t) = 1 - ( t - 1 ) ^ 2$                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src="https://p.ipic.vip/nwt407.jpg" style="zoom:50%;" /> | <img src="https://p.ipic.vip/ajhcam.jpg" style="zoom:50%;" /> | <img src="https://p.ipic.vip/6whz0k.jpg" style="zoom:50%;" /> |



# 向量扩展

对于有些动画值，并非标量，不能简单的如：$kf2.value - kf1.value$直接相减，因此引入Evaluator，从其方法签名`evaluate(float fraction, Object startValue, Object endValue)`，fraction 参数值为$3$的左值，可以看出其职责为，计算出startValue 和 endValue之间，fraction所对应的动画值。
$$
AnimateValue = evaluate(fraction, \ prevKF.value, \ nextKF.value) \tag{4}
$$
以颜色值为例

```java
public Object evaluate(float fraction, Object startValue, Object endValue) {
    int startInt = (Integer) startValue;
    int startA = (startInt >> 24) & 0xff;
    int startR = (startInt >> 16) & 0xff;
    int startG = (startInt >> 8) & 0xff;
    int startB = startInt & 0xff;

    int endInt = (Integer) endValue;
    int endA = (endInt >> 24) & 0xff;
    int endR = (endInt >> 16) & 0xff;
    int endG = (endInt >> 8) & 0xff;
    int endB = endInt & 0xff;

    return (int)((startA + (int)(fraction * (endA - startA))) << 24) |
        (int)((startR + (int)(fraction * (endR - startR))) << 16) |
        (int)((startG + (int)(fraction * (endG - startG))) << 8) |
        (int)((startB + (int)(fraction * (endB - startB))));
  }
```

# 多属性动画

诸多关键帧只能描述单一对象的动画轨迹，将属性和其对应的关键帧封装为PropertyValuesHolder，使得能够支持多个动画对象的多个属性同时动画。

# 动画引擎

动画的内部实现使用Choreographer提供定时引擎，周期性回调执行上述动画值的计算，实现动画对象从上一个关键帧到下一个关键帧的动画渐变效果。



