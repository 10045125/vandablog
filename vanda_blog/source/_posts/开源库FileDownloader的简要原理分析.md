---
title: 开源库FileDownloader的简要原理分析
date: 2016-06-21 17:34:42
tags:
---

最近一直在着手处理文件下载相关的东西，介于没有足够的精力重头开始写，github发现一个文件下载库，其设计还是非常不错的，这里就简要的分析下其结构构成。非常感谢这位小伙伴的开源精神，作者的[Jacksgong github](https://github.com/lingochamp/FileDownloader),博客[Jacksgong blog](http://blog.dreamtobe.cn)


## 框架的模块构成


#### 一个下载框架主要需要包含以下几个模块：
1. 网络请求和文件流模块
2. 消息回调机制
3. 跨进程间通信模块
4. 信息存储模块
5. 异常处理模块


以下是我对下载框架的一个类图结构分析，这部分主要涉及的是独立进程中文件下载需要处理的任务 和 提供远程接口端的类图，暂时没有涉及上层逻辑结构。


{% asset_img img/FileDownloader1.png %}

首先跨进程间的通信方式是采用：

1. 通过Binder来实现远程调用(IPC)：这种方式是Android的最大特色之一，让你调用远程Service的接口，就像调用本地对象一样，实现非常灵活，写起来也相对复杂。
2. 在FileDownloadService里面handler（IFileDownloadServiceHandler）持有实现一个IFileDownloadIPCService.Stub接口的Binder，并且在onBind()中返回此Binder对象
3. 同时在实现IFileDownloadServiceHandler接口的FDServiceSeparateHandler类中拥有了文件下载的远程消息的回调，同时在这个handler中封装了对文件下载管理操作类FileDownloadMgr，远程接口IFileDownloadIPCService中的实现都交由FileDownloadMgr来实现和管理，Service所有操作都委托交由FDServiceSeparateHandler来处理，FDServiceSeparateHandler也实现了MessageReceiver来处理进程间的消息回调，其真正处理回调处理是由IFileDownloadIPCCallback远程接口来实现。
4. 文件的下载中拥有文件下载的线程池FileDownloadThreadPool，线程池中的单个子单元是由一个Runnable构成，在FileDownloadRunnable中有下载文件的相关信息以及相关的操作，在这里我们可以看出，在执行Runnable的线程中是单个的，Runnable中涉及网络请求和数据流的传输都在Runnable中完成，文件的读写操作是堵塞的，Runnable中的文件下载的相关状态信息是通过MessageSnapshotFlow.getImpl().inflow(MessageSnapshotTaker.take(status, model, this))发出的，消息发出交由FDServiceSeparateHandler的receive(MessageSnapshot snapShot),随之RemoteCallbackList<IFileDownloadIPCCallback> callbackList将发起进程间回调通信，来完成通知下载状态的更新，通过以上的步骤来完成Service与其他进程的通信。
5. 文件的下载状态都会记录到指定的数据库中，以便支持完善的下载进度和断点续传功能。

Service代理以及和Service所在进程的通信：

1. FileDownloadServiceProxy实现了IFileDownloadServiceProxy接口，其接口真正的实现是由FileDownloadServiceUIGuard来处理，在FileDownloadServiceUIGuard中提供了进程间通信需要的CALLBACK和INTERFACE
2. BaseFileServiceUIGuard抽象类中实现了ServiceConnection，在onServiceConnected中完成registerCallback，将registerCallback实现交给子类FileDownloadServiceUIGuard，在onServiceConnected中返回了IBinder
3. 通过IFileDownloadIPCService.Stub.asInterface取得远程接口对象，之后通过这个获取的远程接口实现注册相关的操作，同时通过这个远程的接口实现进程间的通信。
4. 在FileDownloadServiceUIGuard中提供了远程接口注册时需要的IFileDownloadIPCService.Stub接口的Binder。
5. FileDownloadServiceUIGuard继承了BaseFileServiceUIGuard，所以FileDownloadServiceUIGuard必须实现FileDownloadServiceProxy接口。同时在BaseFileServiceUIGuard中已经拥有IFileDownloadIPCService的远程接口，
所以FileDownloadServiceProxy接口实现是通过获得的远程接口去实现的，同时这个就是我们说的跨进程通信的方式。
6. 那么现在还有一个比较重要的点，现在怎样把文件下载进度信息交给其他进程呢？不难发现我们注册了一个CallBack的Binder。client端与server不在一个进程，server是无法得知client解注册时传入的回调接口是哪一个（client调用解注册时，是通过binder传输到server端，所以解注册时的回调接口是新创建的，而不是注册时的回调接口）。为了解决这个问题，android提供了RemoteCallbackList这个类来专门管理remote回调的注册与解注册。
7. 通过上述分析，client和Service通信以及Service和client的通信，client和Service通信通过onServiceConnected中返回的IBinder，进而取得远程接口，实现client和Service通信。Service和client通信通过RemoteCallbackList，通过client获取的远程接口，同时通过这个远程接口回调注册，实现Service与client的通信。
8. 经过上述的分析，我们已经能够知道所有的连接通信逻辑都已经由FileDownloadServiceProxy委托代理。


## 上层逻辑类图结构逻辑分析


上层的结构逻辑主要是对远程结构代理的封装，由外部组装必要的数据以及外部等待下载状态的更新。类图结构如下：


{% asset_img img/FileDownloader2.png %}




client的结构构成：

1. 每一个下载任务对应一个单独的FileDownloadTask，其和Service的跨进程通信通过FileDownloadServiceProxy进行关联
2. Service和client的下载状态的回调是通过MessageReceiver接口来实现的，其正确的流程是Service进程中的Runnable下载文件状态   ->  MessageSnapshotFlow.getImpl().inflow()(Service进程)   -> FDServiceSeparateHandler(实现了MessageReceiver接口)   -> FDServiceSeparateHandler的receive()调用RemoteCallbackList<IFileDownloadIPCCallback> callbackList来完成Service和client的通信。
3. FileDownloader是统一收敛、数据组装、状态查询的直接外部入口类，其内部核心处理还是通过FileDownloadTask来构建任务、FileDownloadServiceProxy来处理跨进程数据通信。

通过以上的分析介绍，我们能够知道，其下载其实主要就是通信模块的构建，也就是我们常说的外壳，外壳保证了通信逻辑，我们先构建出来通信外壳模块，将网络连接和文件下载抽象独立出来。外壳保证通信的准确性，下载内核保证了数据的准确性。
同时通过对以上框架的分析，我们需要做到将下载核抽象出来，这样能够更灵活的配置下载核，同时也给下载核更多的发挥空间。


简单的创建demo任务：

```
final BaseDownloadTask task = FileDownloader.getImpl().create(url)
        .setPath(path)
        .setCallbackProgressTimes(100)
        .setListener(taskDownloadListener);
```
这样就完成了一个任务下载，任务的状态信息回调在监听中处理就OK了。





