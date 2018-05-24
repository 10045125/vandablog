---
title: 通过Service拉起进程，同时开始Service的全过程分析学习
date: 2017-03-17 15:27:26
tags:
---

#### 背景

我们通过提供一个对外部的Service拉起相关的进程和服务，这样能够确保我们有一个进程能够一直在后台，从而进行相关的后台操作。那么通过Service去启动所在进程都做了哪些事情，我们做个深入的分析和探讨。


#### 使用场景

一般来说，我们在Activity中开启一个服务的代码如下

```
try {

    Intent intent = new Intent();
    if (Build.VERSION.SDK_INT >= 12)
        intent.addFlags(0x00000020); // Intent.FLAG_INCLUDE_STOPPED_PACKAGES
    intent.setAction("com.vanda.intent.action.FRIEND");
    intent.setClassName("com.vanda", "com.vanda.bridge");
    intent.putExtra("source", getPackageName());
    startService(intent);
} catch (Throwable tr) {
    // No such service or security error or oom error.
}
```

以上代码是一个APP调起另一个APP中的Service进程，这属于跨进程之间的通信范畴，接下来，我们来分析startService是如何工作的。


#### Android进程间通信

Android系统是基于Linux内核的，而Linux内核继承和兼容了丰富的Unix系统进程间通信（IPC（Inter process communication））机制。有传统的管道（Pipe）、信号（Signal）和跟踪（Trace），这三项通信手段只能用于父进程与子进程之间，或者兄弟进程之间；后来又增加了命令管道（Named Pipe），使得进程间通信不再局限于父子进程或者兄弟进程之间；为了更好地支持商业应用中的事务处理，在AT&T的Unix系统V中，又增加了三种称为“System V IPC”的进程间通信机制，分别是报文队列（Message）、共享内存（Share Memory）和信号量（Semaphore）；后来BSD Unix对“System V IPC”机制进行了重要的扩充，提供了一种称为插口（Socket）的进程间通信机制。

Android系统没有采用上述提到的各种进程间通信机制，而是采用Binder机制。在Android系统的Binder机制中，由一些系统组件组成，分别是Client、Server、Service Manager和Binder驱动程序，其中Client、Server和Service Manager运行在用户空间，Binder驱动程序运行内核空间。Binder就是一种把这四个组件粘合在一起的粘结剂了，其中，核心组件便是Binder驱动程序了，Service Manager提供了辅助管理的功能，Client和Server正是在Binder驱动和Service Manager提供的基础设施上，进行Client-Server之间的通信。Service Manager和Binder驱动已经在Android平台中实现好，开发者只要按照规范实现自己的Client和Server组件就可以了。


{% asset_img img/Binder.png %}


 1. Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中
        
 2. Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server
 3. Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信
 4. Client和Server之间的进程间通信通过Binder驱动程序间接实现
 5. Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力
     （感谢老罗说的那么多）
     
     
     
#### Java层的ServiceManager

Service Manager（我觉得是C/C++层）是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力，我们看下类图构成：

{% asset_img img/ServiceManager.jpg %}


1. ServiceManager提供了外部查询的和添加Service的方法，最为核心的是具备实现IServiceManager的ServiceManagerNative类。

2. ServiceManagerNative需要一个实现IBinder的对象，这个对象通过BinderInternal.getContextObject()获得一个C层的BinderProxy对象，这是系统中全局的"context object"。

3. ServiceManagerNative通过一个代理类ServiceManagerProxy，ServiceManagerProxy类实现了IServiceManager接口，IServiceManager提供了getService和addService两个成员函数来管理系统中的Service。从ServiceManagerProxy类的构造函数可以看出，它需要一个BinderProxy对象的IBinder接口来作为参数。因此，要获取Service Manager的Java远程接口ServiceManagerProxy，首先要有一个BinderProxy对象。再来看一下是通过什么路径来获取Service Manager的Java远程接口ServiceManagerProxy的。这个主角就是ServiceManager了，ServiceManager类有一个静态成员函getIServiceManager，它的作用就是用来获取Service Manager的Java远程接口了，而这个函数又是通过ServiceManagerNative来获取Service Manager的Java远程接口的。


```
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }
 
    // Find the service manager
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
```

在C/C++层将对象进行了转换，结果是sServiceManager = ServiceManagerNative.asInterface(new BinderProxy()); 那么我们看下ServiceManagerNative.asInterface做了什么事情

```
static public IServiceManager asInterface(IBinder obj)
{
    if (obj == null) {
        return null;
    }
    IServiceManager in =
        (IServiceManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
     
    return new ServiceManagerProxy(obj);
}
```

从上面的静态方法中看出，如果当地查询不到IServiceManager，in为null ,obj是一个BinderProxy对象,那么就会以BinderProxy对象为参数创建一个ServiceManagerProxy对象。实质上等价的代码为
sServiceManager = new ServiceManagerProxy(new BinderProxy()); 
ServiceManager就是这为了这句代码写的，就是在Java层，我们拥有了一个Service Manager远程接口ServiceManagerProxy，而这个ServiceManagerProxy对象在JNI层有一个句柄值为0的BpBinder对象与之通过gBinderProxyOffsets关联起来。这样获取Service Manager的Java远程接口ServiceManagerProxy的过程就完成了。


我们简单的看下C层做了些什么

```
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& caller)
{
    return getStrongProxyForHandle(0);
}
```

```
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
 
{
 
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
 
    return javaObjectForIBinder(env, b);
 
}
 
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
 
{
    if (val == NULL) return NULL;
    if (val->checkSubclass(&gBinderOffsets)) {
        // One of our own!
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object);
        return object;
    }
    // For the rest of the function we will hold this lock, to serialize
    // looking/creation of Java proxies for native Binder proxies.
    AutoMutex _l(mProxyLock);
    // Someone else's...  do we know about it?
    jobject object = (jobject)val->findObject(&gBinderProxyOffsets);
    if (object != NULL) {
        jobject res = jniGetReferent(env, object);
        if (res != NULL) {
            ALOGV("objectForBinder %p: found existing %p!\n", val.get(), res);
            return res;
        }
        LOGDEATH("Proxy object %p of IBinder %p no longer in working set!!!", object, val.get());
        android_atomic_dec(&gNumProxyRefs);
        val->detachObject(&gBinderProxyOffsets);
        env->DeleteGlobalRef(object);
    }
    object = env->NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
    if (object != NULL) {
        LOGDEATH("objectForBinder %p: created new proxy %p !\n", val.get(), object);
        // The proxy holds a reference to the native object.
        env->SetIntField(object, gBinderProxyOffsets.mObject, (int)val.get());
        val->incStrong((void*)javaObjectForIBinder);
        // The native object needs to hold a weak reference back to the
        // proxy, so we can retrieve the same proxy if it is still active.
        jobject refObject = env->NewGlobalRef(
                env->GetObjectField(object, gBinderProxyOffsets.mSelf));
        val->attachObject(&gBinderProxyOffsets, refObject,
                jnienv_to_javavm(env), proxy_cleanup);
        // Also remember the death recipients registered on this proxy
        sp<DeathRecipientList> drl = new DeathRecipientList;
        drl->incStrong((void*)javaObjectForIBinder);
        env->SetIntField(object, gBinderProxyOffsets.mOrgue, reinterpret_cast<jint>(drl.get()));
        // Note that a new object reference has been created.
        android_atomic_inc(&gNumProxyRefs);
        incRefsCreated(env);
    }
    return object;
}
```

返回一个句柄值为0的BpBinder对象
sp<IBinder> b = new BpBinder(0); 
然后将这个BpBinder对象转换成一个BinderProxy，这样Java层代理类就获得这个对象。
说了这么一大坨就是为了方便我们之后获取Binder填好坑。


#### 关联startService相关类图

不难发现startService可以有多个入口调用，原因看如下类图：

{% asset_img img/startService.png %}


1. Activity、Application、Service都是继承ContextWrapper，其中Service拥有Applcation实例。

2. 在ContextWrapper类中，实现了startService函数。在ContextWrapper类中，有一个成员变量mBase，它是一个ContextImpl实例。

3. ContextImpl类和ContextWrapper类一样继承于Context类，ContextWrapper类的startService函数最终过调用ContextImpl类的startService函数来实现。这种类设计方法在设计模式里面，就称之为装饰模式（Decorator），或者包装模式（Wrapper）。

4. 在ContextImpl类的startService类，最终又调用了ActivityManagerProxy类的startService来实现启动服务的操作



```
private ComponentName startServiceCommon(Intent service, UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess();
        ComponentName cn = ActivityManagerNative.getDefault().startService(
            mMainThread.getApplicationThread(), service,
            service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier());
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException(
                        "Unable to start service " + service
                        + ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        return null;
    }
}
```

结合ServiceManager和类图我们可以知道ActivityManagerProxy是一个Binder对象的远程接口了，ServiceManagerProxy远程对象是C层的ServiceManager，而ActivityManagerProxy这个Binder远程对象就是ActivityManagerService了。在手机启动时会初始化这个Binder，我们看下源码：


```
public class SystemServer {
    private static final String TAG = "SystemServer";
 
    public static final int FACTORY_TEST_OFF = 0;
    public static final int FACTORY_TEST_LOW_LEVEL = 1;
    public static final int FACTORY_TEST_HIGH_LEVEL = 2;
 
    static Timer timer;
    static final long SNAPSHOT_INTERVAL = 60 * 60 * 1000; // 1hr
 
    // The earliest supported time.  We pick one day into 1970, to
    // give any timezone code room without going into negative time.
    private static final long EARLIEST_SUPPORTED_TIME = 86400 * 1000;
 
    /**
     * Called to initialize native system services.
     */
    private static native void nativeInit();
 
    public static void main(String[] args) {
 
        /*
         * In case the runtime switched since last boot (such as when
         * the old runtime was removed in an OTA), set the system
         * property so that it is in sync. We can't do this in
         * libnativehelper's JniInvocation::Init code where we already
         * had to fallback to a different runtime because it is
         * running as root and we need to be the system user to set
         * the property. http://b/11463182
         */
        SystemProperties.set("persist.sys.dalvik.vm.lib",
                             VMRuntime.getRuntime().vmLibrary());
 
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            // If a device's clock is before 1970 (before 0), a lot of
            // APIs crash dealing with negative numbers, notably
            // java.io.File#setLastModified, so instead we fake it and
            // hope that time from cell towers or NTP fixes it
            // shortly.
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
}
```

```
class ServerThread {
    private static final String TAG = "SystemServer";
    private static final String ENCRYPTING_STATE = "trigger_restart_min_framework";
    private static final String ENCRYPTED_STATE = "1";
 
    ContentResolver mContentResolver;
 
    void reportWtf(String msg, Throwable e) {
        Slog.w(TAG, "***********************************************");
        Log.wtf(TAG, "BOOT FAILURE " + msg, e);
    }
 
    public void initAndLoop() {
        try {
                ...
            context = ActivityManagerService.main(factoryTest);
        } catch (RuntimeException e) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting bootstrap service", e);
        }
  
        try {
            ...
            ActivityManagerService.setSystemProcess();
        }catch (RuntimeException e) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting core service", e);
        }
    }
```


系统启动时，会通过ActivityManagerService.main()创建出实例，通过ActivityManagerService.setSystemProcess();将服务添加到Service Manager中


```
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
  
    public static final Context main(int factoryTest) {
        AThread thr = new AThread();
        thr.start();
 
        synchronized (thr) {
                while (thr.mService == null) {
                    try {
                        thr.wait();
                    } catch (InterruptedException e) {
                    }
                }
        }
 
        ActivityManagerService m = thr.mService;
        mSelf = m;
     
        ...
 
        m.startRunning(null, null, null, null);
 
        return context;
    }  
  
  
    public static void setSystemProcess() {
        try {
                ActivityManagerService m = mSelf;
 
                ServiceManager.addService(Context.ACTIVITY_SERVICE, m, true);
                ...
        } catch (PackageManager.NameNotFoundException e) {
                throw new RuntimeException(
                "Unable to find android system package", e);
        }
    }
  
}
```

这样就把ActivityManagerService这个Binder添加到Service Manager中，之后通信的时候，需要通过代理去获取它。这样ActivityManagerService就启动了。


1. 现在知道ActivityManagerProxy.startService是通过ActivityManagerService的startService


2. ActivityManagerProxy的mBinder持有ActivityManagerService Binder的引用，执行接口赋予的功能


       
       
       