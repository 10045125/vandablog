---
title: Android内存优化之OOM
date: 2016-06-30 18:16:45
tags:
---

<link href="http://kevinburke.bitbucket.org/markdowncss/markdown.css" rel="stylesheet"></link>

> Android的内存优化是性能优化中很重要的一部分，而避免OOM又是内存优化中比较核心的一点，这是一篇关于内存优化中如何避免OOM的总结性概要文章，内容大多都是和OOM有关的实践总结概要。理解错误或是偏差的地方，还请多包涵指正，谢谢！


###(一) Android的内存管理机制

Google在Android的官网上有这样一篇文章，初步介绍了Android是如何管理应用的进程与内存分配：[http://developer.android.com/training/articles/memory.html](http://developer.android.com/training/articles/memory.html) Android系统的Dalvik虚拟机扮演了常规的内存垃圾自动回收的角色，Android系统没有为内存提供交换区，它使用paging与memory-mapping(mmapping)的机制来管理内存，下面简要概述一些Android系统中重要的内存管理基础概念。

####1）共享内存

Android系统通过下面几种方式来实现共享内存：

* Android应用的进程都是从一个叫做Zygote的进程fork出来的。Zygote进程在系统启动并且载入通用的framework的代码与资源之后开始启动。为了启动一个新的程序进程，系统会fork Zygote进程生成一个新的进程，然后在新的进程中加载并运行应用程序的代码。这使得大多数的RAM pages被用来分配给framework的代码，同时使得RAM资源能够在应用的所有进程之间进行共享。
大多数static的数据被mmapped到一个进程中。这不仅仅使得同样的数据能够在进程间进行共享，而且使得它能够在需要的时候被paged out。常见的static数据包括Dalvik Code，app resources，so文件等。
大多数情况下，Android通过显式的分配共享内存区域(例如ashmem或者gralloc)来实现动态RAM区域能够在不同进程之间进行共享的机制。例如，Window Surface在App与Screen Compositor之间使用共享的内存，Cursor Buffers在Content Provider与Clients之间共享内存。
 


