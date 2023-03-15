---
title: Android 双指触控 位移 缩放 旋转
date: 2023-03-11 21:13:48
tags: Android Translate Scale Rotate, multiple touch event, Android 双指触控，位移，缩放，旋转
sticky: true
math: true
category:
- [Android]

---
# 效果
![demo](https://pic.imgdb.cn/item/6410989cebf10e5d53bddfb4.gif)
# 代码实现
```kotlin
package com.maxim.opengl

import android.content.Context
import android.graphics.*
import android.view.MotionEvent
import android.view.View
import com.maxim.opengl.utils.Constants
import com.maxim.opengl.utils.VLog
import kotlin.math.acos
import kotlin.math.asin
import kotlin.math.sqrt

class TouchEventView(context: Context) : View(context) {

    private val rect: RectF = RectF(0f, 0f, 400f, 400f)
    private val paint: Paint = Paint()

    init {
        paint.color = Color.RED
    }

    private var downX: Float = 0f
    private var downY: Float = 0f
    private var downPointerX: Float = 0f
    private var downPointerY: Float = 0f

    private var dx: Float = 0f
    private var dy: Float = 0f
    private var scale: Float = 1f
    private var angle: Float = 0f

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                downX = event.getX(0)
                downY = event.getY(0)
            }
            MotionEvent.ACTION_POINTER_DOWN -> {
                downX = event.getX(0)
                downY = event.getY(0)
                downPointerX = event.getX(1)
                downPointerY = event.getY(1)
            }
            MotionEvent.ACTION_MOVE -> {
                if (event.pointerCount <= 1) {
                    translate(event)
                } else {
                    rotate(event)
                    zoom(event)
                }
            }
            MotionEvent.ACTION_POINTER_UP -> {
                val remainIndex = if (event.actionIndex == 0) 1 else 0
                downX = event.getX(remainIndex) - dx
                downY = event.getY(remainIndex) - dy
            }
            MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                // NOP.
            }
            else -> {
                // NOP.
            }
        }
        invalidate()

        return true
    }

    private fun translate(event: MotionEvent) {
        dx = event.getX(0) - downX
        dy = event.getY(0) - downY
    }

    private fun zoom(event: MotionEvent) {
        val prev = distance(downX, downY, downPointerX, downPointerY)
        val next = distance(
            event.getX(0), event.getY(0),
            event.getX(1), event.getY(1)
        )
        scale = next / prev
    }

    private fun rotate(event: MotionEvent) {
        val pointerX = event.getX(1)
        val pointerY = event.getY(1)
        // multiply y component by -1 to make y-axis points up. 
        // which maps view coordinate system to right-hand coordinate system.
        val prevVector = Vector(downPointerX - downX, -(downPointerY - downY), 0f)
        val nextVector = Vector(pointerX - event.x, -(pointerY - event.y), 0f)

        val cp: Vector = prevVector.crossProduct(nextVector)
        // direction is 1 if the angle the next vector rotates with clockwise direction by
        // from previous vector locates at [0, PI], otherwise is -1 if locates at [PI , 2 PI]
        val direction = if (cp.z < 0) Rotate.CLOCKWISE.direction  else Rotate.COUNTER_CLOCKWISE.direction
        // radian represent the angle between the two vectors, whose value locates in [0, PI] always.
        val radian = acos(prevVector.dotProduct((nextVector)) / (prevVector.length() * nextVector.length()))
        angle = (radian * 180 / Math.PI).toFloat() * direction
        VLog.d("$cp, $radian, $angle")
    }

    private fun distance(x1: Float, y1: Float, x2: Float, y2: Float): Float {
        val deltaX = x1 - x2
        val deltaY = y1 - y2
        return sqrt((deltaX * deltaX + deltaY * deltaY).toDouble()).toFloat()
    }

    override fun onDraw(canvas: Canvas?) {
        super.onDraw(canvas)
        val desRect = RectF()
        val matrix = Matrix()
        matrix.reset()
        matrix.postScale(scale, scale, rect.centerX(), rect.centerY())
        matrix.postTranslate(dx, dy)
        matrix.mapRect(desRect, rect)
        canvas?.save()
        // clockwise when degrees is positive.
        canvas?.rotate(angle, desRect.centerX(), desRect.centerY())
        canvas?.drawRect(desRect, paint)
        canvas?.restore()
    }
    enum class Rotate(val direction: Int) {
        CLOCKWISE(1),
        COUNTER_CLOCKWISE(-1),
    }

    class Vector(val x: Float, val y: Float, val z: Float) {

        fun dotProduct(v: Vector): Float {
            return x * v.x + y * v.y + z * v.z
        }

        fun crossProduct(v: Vector): Vector {
            return Vector(
                y * v.z - z * v.y,
                -(x * v.z - z * v.x),
                x * v.y - y * v.x
            )
        }

        fun length(): Double {
            return sqrt((x * x + y * y + z * z).toDouble())
        }

        override fun toString(): String {
            return "($x, $y, $z)"
        }

        companion object {
            fun crossProduct(v1: Vector, v2: Vector): Float {
                return v1.x * v2.y - v1.y * v2.x
            }

            fun dotProduct(v1: Vector, v2: Vector): Float {
                return v1.x * v2.x + v1.y * v2.y
            }
        }
    }
}
```
![cross product and dot product](https://s3.uuu.ovh/imgs/2023/03/15/3519b041b587e7e5.png)
# 向量点乘 Dot Product
$$
\begin{equation}
\vec{AB} \cdot \vec{AC} = \parallel \vec{AB} \parallel * \parallel \vec{AC} \parallel * \cos\theta  \label{dotproduct} \tag{1}
\end{equation}
$$
$$
\vec{AB} \cdot \vec{AD} = \parallel \vec{AB} \parallel * \parallel \vec{AD} \parallel * \cos\theta \tag{2}
$$
$\theta$ 的取值范围为 $[0, \pi]$，由于从$\vec{AB}$ 顺时针旋转到 $\vec{AC}$ 和从 $\vec{AB}$ 顺时针旋转到 $\vec{AD}$ 计算出来的$\theta$ 是一样的，无法区分。而对于`Canvas.rotate(degree)`，当`degree > 0 `，表示从向右方向顺时针旋转，当`degree < 0 `时，表示向右方向逆时针旋转。所以，对于 从$\vec{AB}$ 顺时针旋转 $\theta + 2 * (\pi - \theta) =  2 \pi - \theta$ 到 $\vec{AD}$，相当于从$\vec{AB}$ 逆时针旋转 $\theta$ 到 $\vec{AD}$。 也即公示$\ref{dotproduct}$ 所求 $\theta$ * 方向即可。+1 表示顺时针， -1 表示逆时针，而方向的值可由叉乘计算得出。
# 向量叉乘 Cross Product
如图左手坐标系，使用左手法则。
$$\vec{AB} \times \vec{AC} = \vec{N_c} \tag{3}$$ 
$\vec{N_c}$指向 $-z$
$$\vec{AB} \times \vec{AD} = \vec{N_d} \tag{4}$$
$\vec{N_d}$指向 $z$


# 参考文档
- [Android multiple touch event](https://developer.android.com/develop/ui/views/touch-and-input/gestures/multi)
- [使用斜率计算旋转角度](https://juejin.cn/post/6844904162744860685)
- [使用向量计算旋转角度](https://juejin.cn/post/6844904166700089351)
- [三维叉乘 & 坐标系](https://zhuanlan.zhihu.com/p/64707259)