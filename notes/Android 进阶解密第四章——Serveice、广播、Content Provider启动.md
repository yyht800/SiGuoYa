## Android 进阶解密第四章——Serveice、广播、Content Provider启动

### 1. Service

Service的启动有两种方式，一种是启动流程一种是绑定流程，下面先看启动流程。

#### 1.1 Service的启动流程

##### 1.1.1 ContextImpl到AMS

context.StartService方法，会调用到ContextWrapper.startService，实际最后调用是ContextImpl中的startServiceCommon方法中

```java
// frameworks/base/core/java/android/app/ContextImpl.java

class ContextImpl extends Context {

    @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
    }

    private ComponentName startServiceCommon(Intent service, boolean requireForeground, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            // 通过 Binder 进程间通信进入 ActivityManagerService 的 startService() 方法
            ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                    getContentResolver()), requireForeground, getOpPackageName(), user.getIdentifier());
            ... ...

            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

在上面的注释中可以看到和Activity的启动一样，通过Binder的机制进入到AMS的方法中。

##### 1.1.2 AMS启动Service

ActivityManagerService中startService方法最重要就是调用ActiveServices中startServiceLocked方法

```java
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

public final class ActiveServices {

    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        return startServiceLocked(caller, service, resolvedType, callingPid, callingUid, fgRequired,
                callingPackage, userId, false);
    }

    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage,
            final int userId, boolean allowBackgroundActivityStarts)
            throws TransactionTooLargeException {
        ... ...
        
        //1. 解析 AndroidManifest.xml 文件中配置 Service 的 intent-filter 相关内容信息
        ServiceLookupResult res = retrieveServiceLocked(service, null, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false, false);
        if (res == null) {
            return null;
        }
        ... ...

        ServiceRecord r = res.record;
        ... ...

        //2. 调用 startServiceInnerLocked() 方法
        ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        return cmp;
    }

}
```

注释1处，就是通过retrieveServiceLocked方法，解析清单文件中的配置，相关信息会保存在ServiceRecord，类似ActivityRecord。

然后看注释2处的代码，最终调用的是bringUpServiceLocked()，看下代码：

```java
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

public final class ActiveServices {

    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        ... ...
        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);    // 获取 app 对象
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid + " app=" + app);					
            if (app != null && app.thread != null) {//1.判断进程是否存在
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    //2. 运行 service 的进程已经启动，调用 realStartServiceLocked() 方法
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortInstanceName, e);
                }
                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }
        } else {
            ... ...
        }

        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (app == null && !permissionsReviewRequired) {//3. 如果用来运行Service的进程不存在
            if ((app = mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingRecord, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
        ... ...
        return null;
    }
}
```

在注释1处，会对Service需要运行的进程判断，是否存在，如果存在直接在进程中执行。

在注释3处，会创建进程，AMS创建应用程序的进程和Activity启动一章中差不多，这里不做赘述。

我们接下来看下注释2处，调用的realStartServiceLocked方法：

```java
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

public final class ActiveServices {

    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        ... ...

        try {
            ... ...
						
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo), app.getReportedProcState());
            ... ...

        }
        ... ...
    }
}
```

在这个方法中，app 对象的 `thread` 变量是一个 `ApplicationThread` `Binder` 对象，调用它的 `scheduleCreateService()` 方法之后，会进入客户端的 `ActivityThread` 中。

##### 1.1.3 主线程加载

和Activity的启动流程一样，这里通过binder的方式，又回到了ApplicationThread中。

```java
// frameworks/base/core/java/android/app/ActivityThread.java

public final class ActivityThread extends ClientTransactionHandler {

    // ApplicationThread 是一个 Binder
    private class ApplicationThread extends IApplicationThread.Stub {

        public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
    }   
}
```

又看到熟悉的sendMessage，没错这里后面也是通过哦mH，来切换到主线程的。

在对应的case内部调用了handleCreateService：

```java
// frameworks/base/core/java/android/app/ActivityThread.java

public final class ActivityThread extends ClientTransactionHandler {

    private void handleCreateService(CreateServiceData data) {
        ... ...

        LoadedApk packageInfo = getPackageInfoNoCheck(data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = packageInfo.getAppFactory().instantiateService(cl, data.info.name, data.intent);
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
						//为Service创建上下文
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
          //初始化service
            service.attach(context, this, data.info.name, data.token, app, ActivityManager.getService());
            service.onCreate();    // 进入 Service.onCreate() 方法
            mServices.put(data.token, service);
            try {
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }

}
```

首先通过 `ClassLoader` 类把 Service 加载进来，而参数 `data.info.name` 表示这个 `Service` 的名字，`instantiateService()` 方法是创建一个 Service 实例。接着，`创建一个 Context 对象`，作为上下文环境之用。

最后调用了service的onCreate的方法，对应的生命周期。



#### 1.2 Service的绑定流程

##### 1.2.1 从ContextImpl到AMS

bindService的方法来绑定Service，和startService方法一样，先在ContextWrapper中，然后到ContextImpl中，

```java
 						private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
1708            String instanceName, Handler handler, Executor executor, UserHandle user) {
1709        // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
1710        IServiceConnection sd;
1711        ...
1717        if (mPackageInfo != null) {
1718            if (executor != null) {
										//1 封装成IServiceConnection
1719                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
1720            } else {
1721                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
1722            }
1723        } else {
1724            throw new RuntimeException("Not supported in system context");
1725        }
1726        validateServiceIntent(service);
1727        try {
1728            ...
  							// 2
1735            int res = ActivityManager.getService().bindIsolatedService(
1736                mMainThread.getApplicationThread(), getActivityToken(), service,
1737                service.resolveTypeIfNeeded(getContentResolver()),
1738                sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
1739            if (res < 0) {
1740                throw new SecurityException(
1741                        "Not allowed to bind to service " + service);
1742            }
1743            return res != 0;
1744        } catch (RemoteException e) {
1745            throw e.rethrowFromSystemServer();
1746        }
1747    }
```

注释1处封装成一个IServiceConnection对象，在注释2处，通过Binder将对象传递到AMS中。

##### 1.2.2 Service的绑定过程

同样是AMS中的同名方法，然后调用ActiveServices的bindServiceLocked方法:

```java
int bindServiceLocked(...){
            //...
            if ((flags&Context.BIND_AUTO_CREATE) != 0) { //BIND_AUTO_CREATE
            	//...
                if (bringUpServiceLocked(...) { //1 用于拉起service
                    return 0;
                }
            }
            
            mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
                    callerApp.getCurProcState(), s.appInfo.uid, s.appInfo.longVersionCode,
                    s.instanceName, s.processName); //将Service放在独立的进程中
            
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp); //记录是否绑定的数据结构
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent,
                    callerApp.uid, callerApp.processName, callingPackage);//记录连接状态的数据结构，持有connection的binder对象
            
            if (s.app != null && b.intent.received) { //2 如果service已经被绑定过 或者Service进程存在
            	c.conn.connected(s.name, b.intent.binder, false);//通过binderProxy调用client端的`onServiceConnected回调`
            }else if (!b.intent.requested) { //AMS还没有请求过binder对象
                requestServiceBindingLocked(s, b.intent, callerFg, false); //3. 请求binder对象,rebind = false
            }
            
}            

```

注释1处，最后一系列操作后到的realStartServiceLocked方法，最后会由ActivityThread来调用Service的onCreate方法启动Service

注释2处和上面一样是判定当前的Service是否已经绑定了进程，但是需要注意的事 `b.intent.received`这里，其实是一个IntentBindRecord类型的对象，AMS会为每个绑定的Service的Intent分配一个IntentBindRecord（它内部管理者ServiceRecord对象）

注释3处，重新绑定Service，如果说rebind为false，那么就表示不需要重新绑定。

接下来继续往下走，requestServiceBindingLocked方法。然后会调用`r.app.thread.scheduleBindService()`远程调用到ActivityThead中的`scheduleBindService()`函数, Activity中又调用到了`handleBindService((BindServiceData)msg.obj)`来处理



```java 
private void handleBindService(BindServiceData data) {
		   if (!data.rebind) { //不是rebind时调用onBind
                        IBinder binder = s.onBind(data.intent); //调用onBind返回一个binder对象
                        ActivityManager.getService().publishService( //将binder对象发布到AMS
                                data.token, data.intent, binder);
                    } else {
                        s.onRebind(data.intent);//是rebind时调用onRebind
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
}

```

如果是rebind，则直接调用Service的`onRebind`生命周期回调

如果是第一次执行bind，则获取到`AMS`的binder对象调用`publishService()`将binder对象注册到AMS中，在`AMS`中又调用到了`ActiveServices`的`publishServiceLocked()`函数，这里的主要逻辑是根据intent找到ServiceRecord中所有匹配的ConnectionRecord，并将binder对象（service）分发给ConnectionRecord.

#### 1.3 小结

几个类说明的补充

- **ActiveServices**
 AMS中管理Service的具体类。

- **ServiceRecord**
   Service结构的具体描述。

- **ConnectionRecord**
 Client端与Service端绑定的抽象描述。 



**Q1：** start和bind两种启动方式的区别是什么？

**一、两种启动方式的定位不同：**

- start是一种纯后台服务，client端不需要/无法直接关注到Service的运行结果；
- bind提供的是一种client/server服务，client端可以通过Service返回的IBinder与Service进行交互。

**二、生命周期不同：**

- start是纯后台服务一直在后台运行，除了以下几种场景会被stop：
   1）client端调用stopService
   2）service内部主动stopSelf
   3）service端进程被kill
   如果onStartCommand返回`START_STICKY`，进程被kill（非force stop）时系统会主动再把Service拉起来。
- bind服务的生命周期与Client绑定，
   1）client端unbind`Context.BIND_AUTO_CREATE`类型连接
   2）client端绑定的Activity destroy
   3）client端进程被kill
   Service不会再被系统拉起。

> 注：bind服务并非需要全部unbind才会销毁Service，只要最后一个Context.BIND_AUTO_CREATE的连接unbind，Service就会被销毁掉。但当Service既被bind又被start时，需要调用stopService/stopSelf，且最后一个Context.BIND_AUTO_CREATE的连接unbind，Service才会被销毁掉。

**Q2：** Android 8.0+对后台Service究竟做了什么限制？对所有App一视同仁吗？

**一、主要对`target>=26`的App做了2点限制：**（*前台服务和bind服务 不受限制*）

1. 应用在后台不能startService；
2. 应用从前台切换到后台后，后台Service会在`60s`之后被kill

**二、3个例外场景：**

1. `persistent`应用不受限制；
2. 处于后台白名单中的`uid`不受限制
    uid为`system`的应用可以调用`AMS.backgroundWhitelistUid`将某一应用加入到此白名单中。
3. 处于耗电白名单的应用不受限制
    此白名单定义在`/data/system/deviceidle.xml`文件中

**Q3：** 前台服务为什么不受限制？前台服务可以不弹前台通知吗？

**一、从设计层面看**

Android 8.0加强了对后台服务的限制，主要是为了减少应用在后台运行不必要的Service，提高电池续航时间。前台服务需有前台通知，对用户来说是有感知的。

**二、从代码层面看**

前台服务的优先级更高，当应用处于后台，且：

- **有前台服务**
   进程状态定义为`非idle`，即不受限制；
- **只有后台服务**
   进程状态定义为`idle`，受限制。

从AOSP的代码看，其实是可以做到*前台服务不弹通知*的，因为AMS中判断是否为前台服务，是通过Service有没有调用`startForeground`，以及其通知id来判断的。即：

1. Service调用了`startForeground`，且通知id不为0；
2. 且Service未调用`stopForeground`移除前台通知。

只要满足以上2个条件，对`AMS`来说就是前台通知，`r.postNotification`是通过`NotificationManagerService`展示的，`AMS`与其的交互经过了消息处理，并不是同步的，`AMS`不能直接获取到通知的展示结果。

所以，只要想办法让`NMS`不展示通知，理论上是可以做到*前台服务不弹通知*的。在AOSP代码中，设置一个`无效的通知渠道`和在设置页面关闭`允许通知`的效果是一样的。即，给前台服务通知设置一个`无效的通知渠道`即可做到*前台服务不弹通知*。

但在一些国产手机中，这个方法无效，设置`无效的通知渠道`会直接导致应用crash，或者在应用切换到后台`60s`后Service被kill。这也许是因为利用此设计缺陷的开发者太多，国产ROM做了改进吧。



### 2. 广播的注册、 发送和接收过程

BroadcastReceiver分为两类：

- 静态广播接收者：通过AndroidManifest.xml的标签来申明的BroadcastReceiver。
- 动态广播接收者：通过AMS.registerReceiver()方式注册的BroadcastReceiver，动态注册更为灵活，可在不需要时通过unregisterReceiver()取消注册。

从广播发送方式可分为三类：

- 普通广播：通过Context.sendBroadcast()发送，可并行处理
- 有序广播：通过Context.sendOrderedBroadcast()发送，串行处理
- Sticky广播：通过Context.sendStickyBroadcast()发送

广播注册和Service注册的的流程差不多，通过ContextImpl，操作到AMS中mRegisteredReceivers的广播队列中

另外，当注册的是Sticky广播。

#### 2.1 广播的发送

广播的方发送可以分为，无序广播（普通广播），有序广播，粘性广播，这里来看一下无序广播。

ContextWrapper的sendBroadcast ----->Contextlmpl的sendBroadcast 最后到AMS的broadcastlntent

```java
public final int broadcastIntent(IApplicationThread caller, Intent intent, String resolvedType, IIntentReceiver resultTo, int resultCode, String resultData, Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle options, boolean serialized, boolean sticky, int userId) {
    enforceNotIsolatedCaller("broadcastIntent");
    synchronized(this) {
        //验证广播intent是否有效 即广播是否合法
        intent = verifyBroadcastLocked(intent);
        //获取调用者进程记录对象
        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        // 2.
        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                requiredPermissions, appOp, null, serialized, sticky,
                callingPid, callingUid, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
```



注释2处broadcastIntentLocked方法

```java
private final int broadcastIntentLocked(ProcessRecord callerApp, String callerPackage, Intent intent, String resolvedType, IIntentReceiver resultTo, int resultCode, String resultData, Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle options, boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {

    //step1: 设置flag
    //step2: 广播权限验证
    //step3: 处理系统相关广播
    //step4: 增加sticky广播
    //step5: 查询receivers和registeredReceivers
    //step6: 处理并行广播
    //step7: 合并registeredReceivers到receivers
    //step8: 处理串行广播

    return ActivityManager.BROADCAST_SUCCESS;
}
```

这里代码省略了，主要工作是将动态和静态注册的广播按照优先级高低存储在不同的列表中，再将两个列表合并到receivers，这样receivers就包括了所有的广播接受者，创建BroadcastRecord对象，并将receivers传进去。

#### 2.2 广播的接收

BroadcastQueue中的scheduleBroadcastsLocked 调用BroadcastHandler，最后调用processNextBroadcast方法：

```java
inal void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        //part1: 处理并行广播
        while (mParallelBroadcasts.size() > 0) {
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            final int N = r.receivers.size();
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                // 1. 分发广播给已注册的receiver
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
            }
            addBroadcastToHistoryLocked(r);//将广播添加历史统计
        }

        //part2: 处理当前有序广播
        do {
            if (mOrderedBroadcasts.size() == 0) {
                mService.scheduleAppGcsLocked(); //没有更多的广播等待处理
                if (looped) {
                    mService.updateOomAdjLocked();
                }
                return;
            }
            r = mOrderedBroadcasts.get(0); //获取串行广播的第一个广播
            boolean forceReceive = false;
            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
            if (mService.mProcessesReady && r.dispatchTime > 0) {
                long now = SystemClock.uptimeMillis();
                if ((numReceivers > 0) && (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                    broadcastTimeoutLocked(false); //当广播处理时间超时，则强制结束这条广播
                }
            }
            ...
            if (r.receivers == null || r.nextReceiver >= numReceivers
                    || r.resultAbort || forceReceive) {
                if (r.resultTo != null) {
                    //处理广播消息消息，调用到onReceive()
                    performReceiveLocked(r.callerApp, r.resultTo,
                        new Intent(r.intent), r.resultCode,
                        r.resultData, r.resultExtras, false, false, r.userId);
                }

                cancelBroadcastTimeoutLocked(); //取消BROADCAST_TIMEOUT_MSG消息
                addBroadcastToHistoryLocked(r);
                mOrderedBroadcasts.remove(0);
                continue;
            }
        } while (r == null);

        //part3: 获取下一个receiver
        r.receiverTime = SystemClock.uptimeMillis();
        if (recIdx == 0) {
            r.dispatchTime = r.receiverTime;
            r.dispatchClockTime = System.currentTimeMillis();
        }
        if (!mPendingBroadcastTimeoutMessage) {
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            setBroadcastTimeoutLocked(timeoutTime); //设置广播超时延时消息
        }

        //part4: 处理下条有序广播
        ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                info.activityInfo.applicationInfo.uid, false);
        if (app != null && app.thread != null) {
            app.addPackage(info.activityInfo.packageName,
                    info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
            processCurBroadcastLocked(r, app); //[处理串行广播]
            return;
            ...
        }

        //该receiver所对应的进程尚未启动，则创建该进程
        if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                        == null) {
            ...
            return;
        }
    }
}
```

然后看代码发现，通过deliverToRegisteredReceiverLocked方法将广播分发到自己的Receiver，代码不展示了主要是做了权限的检查，然后看内部调用的performReceiveLocked

```java
private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver, Intent intent, int resultCode, String data, Bundle extras, boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    //通过binder异步机制，向receiver发送intent
    if (app != null) {
        if (app.thread != null) {
            //调用ApplicationThreadProxy类对应的方法 
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                    data, extras, ordered, sticky, sendingUser, app.repProcState);
        } else {
            //应用进程死亡，则Recevier并不存在
            throw new RemoteException("app.thread must not be null");
        }
    } else {
        //调用者进程为空，则执行该分支
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
```

这里的代码有到了熟悉的app.thread,也就是回到了ApplicationThread，然后最后一样通过哦LoadedApk，performReceive方法中：

```java
public void performReceive(Intent intent, int resultCode, String data, Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    //通过handler消息机制发送args.
    if (!mActivityThread.post(args)) {
        //消息成功post到主线程，则不会走此处。
        if (mRegistered && ordered) {
            IActivityManager mgr = ActivityManagerNative.getDefault();
            args.sendFinished(mgr);
        }
    }
}
```

最后到BroadcastReceiver.PendingResult 

```java
public final class LoadedApk {
  static final class ReceiverDispatcher {
    final class Args extends BroadcastReceiver.PendingResult implements Runnable {
        public void run() {
            final BroadcastReceiver receiver = mReceiver;
            final boolean ordered = mOrdered;

            final IActivityManager mgr = ActivityManagerNative.getDefault();
            final Intent intent = mCurIntent;
            mCurIntent = null;

            if (receiver == null || mForgotten) {
                if (mRegistered && ordered) {
                    sendFinished(mgr);
                }
                return;
            }

            try {
                //获取mReceiver的类加载器
                ClassLoader cl =  mReceiver.getClass().getClassLoader();
                intent.setExtrasClassLoader(cl);
                setExtrasClassLoader(cl);
                receiver.setPendingResult(this);
                // 1. 回调广播onReceive方法
                receiver.onReceive(mContext, intent);
            } catch (Exception e) {
                ...
            }

            if (receiver.getPendingResult() != null) {
                finish(); 
            }
        }
      }
    }

```

上述的1处，就是BroadcastReceiver的onReceive方法。



#### 2.3 总结

1.BroadcastReceiver分为两类：

- 静态广播接收者：通过AndroidManifest.xml的标签来申明的BroadcastReceiver;
- 动态广播接收者：通过AMS.registerReceiver()方式注册的BroadcastReceiver, 不需要时记得调用unregisterReceiver();

2.广播发送方式可分为三类:

| 类型       | 方法                 | ordered | sticky |
| :--------- | :------------------- | :------ | :----- |
| 普通广播   | sendBroadcast        | false   | false  |
| 有序广播   | sendOrderedBroadcast | true    | false  |
| Sticky广播 | sendStickyBroadcast  | false   | true   |

3.广播注册registerReceiver():默认将当前进程的主线程设置为scheuler. 再向AMS注册该广播相应信息, 根据类型选择加入mParallelBroadcasts或mOrderedBroadcasts队列.

4.广播发送processNextBroadcast():根据不同情况调用不同的处理过程:

- 如果是动态广播接收者，则调用deliverToRegisteredReceiverLocked处理；
- 如果是静态广播接收者，且对应进程已经创建，则调用processCurBroadcastLocked处理；
- 如果是静态广播接收者，且对应进程尚未创建，则调用startProcessLocked创建进程。

![](http://gityuan.com/images/ams/send_broadcast.jpg)



### 3.  Content Provider 

Content Provider 作为四 大组件之一，即内容提供者，在通常情况下并没有其他的组件使用高频，直接看连接吧

[Android 这 13 道 ContentProvider 面试题，你都会了吗](https://www.jianshu.com/p/7a6b786ba728)





### 参考文章

1. [Android Broadcast广播机制分析](http://gityuan.com/2016/06/04/broadcast-receiver/)
2. Android进阶解密第四章
3. Android10 源码