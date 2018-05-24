---
title: Android Studio no debuggable application
date: 2016-06-24 10:47:18
tags:
---

在Android调试的时候我们经常会遇到no debuggable application，这里说下我自己的解决办法，以便以后查找。

`通过AndroidStudio中 Tools->Android->Enable ADB Integration active.
之后需等待一会，可能adb会重启，之后就会发现那个框框正常显示你已启动的app`

这个时候可能还会遇到依然是no debuggable application，我的解决办法是找到下拉通知栏中的`正在通过USB`，如图：

{% asset_img img/img1.png %}

进入到选择对话框，选择一个试试。

{% asset_img img/img2.png %}
