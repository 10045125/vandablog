---
title: 研究Notification时想知道的进程间通信Binder机制在应用程序框架层的Java接口源代码分析
date: 2017-03-17 15:20:55
tags:
---

#### 问题来源

在调研notification时，这里涉及多个进程间的通信。并且在调研push通知开关状态时，涉及到framework的源码分析。所有，对系统想做一个系统的分析学习。

在framework/services/java/com/android/server/SystemServer.java中

```
public class SystemServer {
 
 
...
...
    private static native void nativeInit();
 
 
    public static void main(String[] args) {
 
        SystemProperties.set("persist.sys.dalvik.vm.lib",
 
                             VMRuntime.getRuntime().vmLibrary());
 
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
 
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
 
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
 
        }
 
        if (SamplingProfilerIntegration.isEnabled()) {
 
            SamplingProfilerIntegration.start();
 
            timer = new Timer();
 
            timer.schedule(new TimerTask() {
 
                @Override
 
                public void run() {
 
                    SamplingProfilerIntegration.writeSnapshot("system_server", null);
 
                }
 
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
 
        }
 
        // Mmmmmm... more memory!
 
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
 
        // The system server has to run all of the time, so it needs to be
 
        // as efficient as possible with its memory usage.
 
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
 
        Environment.setUserRequired(true);
 
        System.loadLibrary("android_servers");
 
        Slog.i(TAG, "Entered the Android system server!");
 
        // Initialize native services.
 
        nativeInit();
        // This used to be its own separate thread, but now it is
 
        // just the loop we run on the main thread.
 
        ServerThread thr = new ServerThread();
 
        thr.initAndLoop();
 
    }
```


这里存在一个java的main函数的入口，这个是初始化应用层相关服务的，我们分析initAndLoop，这个方法里面包含初始化各种系统的Service，而这些都是通过ServiceManager去做的。
 
我们需要获取的ServiceManager的Java远程接口是一个ServiceManagerProxy对象的IServiceManager接口。ServiceManagerProxy类图结构：

{% asset_img img/ClassDiagram1.png %}


从上图可以知道ServiceManagerProxy类实现了IServiceManager接口，IServiceManager提供了getService和addService两个成员方法来管理系统中的Service。在ServiceManagerProxy类的构造函数可以得知需要一个BinderProxy对象的IBinder接口来作为参数。因此，要获取ServiceManager的Java远程接口ServiceManagerProxy，首先要有一个BinderProxy对象。BinderProxy对象是如何获得的？

我们先看看ServiceManager里面做了些什么


{% asset_img img/Main.png %}

我想知道这些Service是如何去通信的，我们看下在ServiceManager中的代码

```
/** @hide */
 
public final class ServiceManager {
 
    private static final String TAG = "ServiceManager";
    private static IServiceManager sServiceManager;
    private static HashMap<String, IBinder> sCache = new HashMap<String, IBinder>();
    private static IServiceManager getIServiceManager() {
 
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
 
    }
  
...
...
}
```

在上面贴出了ServiceManager的一个静态方法
getIServiceManager
这个方法是获取Java远程接口，getIServiceManager内部是通过方法
ServiceManagerNative.asInterface(BinderInternal.getContextObject())
去获取远程接口。在调用ServiceManagerNative.asInterface方法之前是需要传入一个参数，这个参数是
BinderInternal.getContextObject()返回的值。我们看下这个方法到底是什么鬼：


```
/**
 
 * Private and debugging Binder APIs.
 
 *
 
 * @see IBinder
 
 */
 
public class BinderInternal {
  
...
/**
 
     * Return the global "context object" of the system.  This is usually
 
     * an implementation of IServiceManager, which you can use to find
 
     * other services.
 
     */
 
    public static final native IBinder getContextObject();
  
...
}
```

BinderInternal.getContextObject是一个JNI方法，返回的值是IBinder类型，在framework搜索这个方法的JNI实现

```
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
 
{
 
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
 
    return javaObjectForIBinder(env, b);
 
}
```

首先是调用ProcessState::self函数，self函数是ProcessState的静态成员函数，它的作用是返回一个全局唯一的ProcessState实例变量，就是单例模式了，这个变量名为gProcess。如果gProcess尚未创建，就会执行创建操作，在ProcessState的构造函数中，会通过open文件操作函数打开设备文件/dev/binder，并且返回来的设备文件描述符保存在成员变量mDriverFD中。
接着调用gProcess->getContextObject函数来获得一个句柄值为0的Binder引用，即BpBinder了,相似的代码：

```
sp<IBinder> b = new BpBinder(0);
```

我们在看看ServiceManagerNative.asInterface里面的代码

```
/**
 
 * Native implementation of the service manager.  Most clients will only
 
 * care about getDefault() and possibly asInterface().
 
 * @hide
 
 */
 
public abstract class ServiceManagerNative extends Binder implements IServiceManager
 
{
 
/**
 
     * Cast a Binder object into a service manager interface, generating
 
     * a proxy if needed.
 
     */
 
    static public IServiceManager asInterface(IBinder obj)
 
    {
 
        if (obj == null) {
 
            return null;
 
        }
 
        IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
 
        if (in != null) {
 
            return in;
 
        }
 
        return new ServiceManagerProxy(obj);
 
    }
...
...
}
```

在静态方法asInterface里面需要一个IBinder的参数，就是一个代理对象，如果当地不存在描述的IServiceManager那么就通过这个BinderProxy去创建一个。
 
那我们在创建AIDL文件时，系统到底做了什么？
