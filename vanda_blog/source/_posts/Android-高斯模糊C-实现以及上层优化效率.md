---
title: Android 高斯模糊C++ 实现以及上层优化效率
date: 2018-04-25 15:19:56
tags:
---

#### 产品需求
应用背景的高斯模糊

#### bitmap matrix RenderScript
#### 自己实现，bitmap matrx

粗略的Google了下，Android系统提供了一套实现，调用和简单，如下：

```
public static Bitmap apply(Context context, Bitmap sentBitmap, int radius) {
        final Bitmap bitmap = sentBitmap.copy(sentBitmap.getConfig(), true);
        final RenderScript rs = RenderScript.create(context);
        final Allocation input = Allocation.createFromBitmap(rs, sentBitmap, Allocation.MipmapControl.MIPMAP_NONE, Allocation.USAGE_SCRIPT);
        final Allocation output = Allocation.createTyped(rs, input.getType());
        final ScriptIntrinsicBlur script = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));

        script.setRadius(radius);
        script.setInput(input);
        script.forEach(output);
        output.copyTo(bitmap);

        sentBitmap.recycle();
        rs.destroy();
        input.destroy();
        output.destroy();
        script.destroy();

        return bitmap;
    }
```

其中radius最大值是25，模糊度一般。

处理如下图耗时66ms

{% asset_img img/Re.png %}


#### 自己实现

不限制radius，这里设置25，耗时48ms，效果图：

{% asset_img img/Re2.png %}


将radius设置为60，耗时101ms，达到设计要求，效果如图：

{% asset_img img/Re3.png %}