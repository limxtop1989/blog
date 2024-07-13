---
title: canvas
tags: Android Canvas API, Matrix
categories:
- [Android]
---
```java
    canvas.drawColor(Color.BLUE);
    paint.setColor(Color.GRAY);
    canvas.drawRect(new Rect(0, 0, 400, 400),paint);

    canvas.save();
    canvas.translate(200, 200);
    canvas.scale(0.5f, 0.5f);
    paint.setColor(Color.YELLOW);
    canvas.drawRect(new Rect(0, 0, 400, 400),paint);
```
<img src="https://img-blog.csdn.net/20160908173552188?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="">
Matrix_I * M_t * M_s * Position

```java
    canvas.drawColor(Color.BLUE);
    paint.setColor(Color.GRAY);
    canvas.drawRect(new Rect(0, 0, 400, 400),paint);

    canvas.save();
    canvas.scale(0.5f, 0.5f);
    canvas.translate(200, 200);
    paint.setColor(Color.YELLOW);
    canvas.drawRect(new Rect(0, 0, 400, 400),paint);
```
<img src="https://img-blog.csdn.net/20160908173556780?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt="">
Matrix_I * M_s * M_t * Position

Canvas 上的矩阵变换都是变更 canvas 坐标系
```java
com.android.gallery3d.common.BitmapUtils#resizeAndCropCenter
public static Bitmap resizeAndCropCenter(Bitmap bitmap, int size, boolean recycle) {
	int w = bitmap.getWidth();
	int h = bitmap.getHeight();
	if (w == size && h == size) return bitmap;

	// scale the image so that the shorter side equals to the target;
	// the longer side will be center-cropped.
	float scale = (float) size / Math.min(w,  h);

	Bitmap target = Bitmap.createBitmap(size, size, getConfig(bitmap));
	int width = Math.round(scale * bitmap.getWidth());
	int height = Math.round(scale * bitmap.getHeight());
	Canvas canvas = new Canvas(target);
	canvas.translate((size - width) / 2f, (size - height) / 2f);
	canvas.scale(scale, scale);
	Paint paint = new Paint(Paint.FILTER_BITMAP_FLAG | Paint.DITHER_FLAG);
	canvas.drawBitmap(bitmap, 0, 0, paint);
	if (recycle) bitmap.recycle();
	return target;
}
```