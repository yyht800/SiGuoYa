# Android 进阶解密第三章——App的启动过程

## 1.概述

启动一个App的一般分为两种情况，一种是冷启动，另一种是热启动。区别在于AMS会检查该进程是否存在。

* 冷启动

  点击Launcher图标，如果当前进程不存在，申请fork一个进程加载Application，并启动对应Activity，这个过程称之为冷启动。

* 热启动

  应用的热启动比冷启动简单得多，开销也更低。在热启动中，因为**系统里已有该应用的进程**，所以系统的所有工作就是将您的 Activity 带到前台。

## 2. 启动流程

### 2.1 通知系统，启动的需求

点击图标，Launcher启动Activity，调用Activity.startActivityForResult方法，最终会转到mInstrumentation.execStartActivity方法。由于Launcher自己处在一个单独的进程，所以它需要跨进程告诉系统服务我要启动App的需求。

找到要通知的Service，名叫ActivityTaskManagerService，然后使用AIDL，通过Binder与他进行通信。

这里的简单说下ActivityTaskManagerService（简称ATMS）。原来这些通信工作都是属于ActivityManagerService，现在分了一部分工作给到ATMS，主要包括四大组件的调度工作。

```java
        //Instrumentation.java
    int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);


    //ActivityTaskManager.java
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };



    //ActivityTaskManagerService.java
    public class ActivityTaskManagerService extends IActivityTaskManager.Stub

    public static final class Lifecycle extends SystemService {
        private final ActivityTaskManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityTaskManagerService(context);
        }

        @Override
        public void onStart() {
            publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
            mService.start();
        }
    }
```

startActivity我们都很熟悉，平时启动Activity都会使用，启动应用也是从这个方法开始的，也会同样带上intent信息，表示要启动的是哪个Activity。

另外要注意的一点是，startActivity之后有个checkStartActivityResult方法，这个方法是用作检查启动Activity的结果。当启动Activity失败的时候，就会通过这个方法抛出异常，比如有我们常见的问题：未在AndroidManifest.xml注册。

```java
public static void checkStartActivityResult(int res, Object intent) {
        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                        + intent);
            case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
                throw new AndroidRuntimeException(
                        "FORWARD_RESULT_FLAG used while also requesting a result");
            case ActivityManager.START_NOT_ACTIVITY:
                throw new IllegalArgumentException(
                        "PendingIntent is not an activity");
            //...
        }
    }
```

### 2.2 通知Launcher 已经收到消息

ATMS收到要启动的消息后，就会通知上一个应用，也就是Launcher可以休息会了，进入Paused状态。

```java
        //ActivityStack.java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        //...
        ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
        //...
        boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        //...

        if (next.attachedToProcess()) {
            //应用已经启动
            try {
                //...
                transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                getDisplay().mDisplayContent.isNextTransitionForward()));
                mService.getLifecycleManager().scheduleTransaction(transaction);
                //...
            } catch (Exception e) {
                //...
                mStackSupervisor.startSpecificActivityLocked(next, true, false);
                return true;
            }
            //...
            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.completeResumeLocked();
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                return true;
            }
        } else {
            //冷启动流程
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
    }
```

这里有两个类没有见过：

* ActivityStack，是Activity的栈管理，相当于我们平时项目里面自己写的Activity管理类，用于管理Activity的状态啊，如栈出栈顺序等等。
* ActivityRecord，代表具体的某一个Activity，存放了该Activity的各种信息。

PS:关于这两个类还有几个补充：

1. ActivityRecord是Activity管理的最小单位，它对应着应用进程的一个Activity;
2. TaskRecord也是一个栈式管理结构，每一个TaskRecord都可能存在一个或多个ActivityRecord，栈顶的ActivityRecord表示当前可见的界面;
3. ActivityStack是一个栈式管理结构，每一个ActivityStack都可能存在一个或多个TaskRecord，栈顶的TaskRecord表示当前可见的任务;
4. ActivityStackSupervisor管理着多个ActivityStack，但当前只会有一个获取焦点\(Focused\)的ActivityStack;
5. ProcessRecord记录着属于一个进程的所有ActivityRecord，运行在不同TaskRecord中的ActivityRecord可能是属于同一个 ProcessRecord。

接上文，startPausingLocked方法就是让上一个应用，这里也就是Launcher进入Paused状态。然后就会判断应用是否启动，如果已经启动了，就会走ResumeActivityItem的方法，看这个名字，结合应用已经启动的前提，是不是已经猜到了它是干吗的？没错，这个就是用来控制Activity的onResume生命周期方法的，不仅是onResume还有onStart方法，具体可见ActivityThread的handleResumeActivity方法源码。

如果应用没启动就会接着走到startSpecificActivityLocked方法，接着看。

### 2.3 判断是否启动进程，没有启动则重新创建

Launcher进入Paused之后，ActivityTaskManagerService就会判断要打开的这个应用进程是否已经启动，如果已经启动，则直接启动Activity即可，这也就是应用内的启动Activity流程。

如果进程没有启动，则需要创建进程。

这里有两个问题：

**1. 怎么判断应用进程是否存在呢？**

如果一个应用已经启动了，会在ATMS里面保存一个WindowProcessController信息，这个信息包括processName和uid，uid则是应用程序的id，可以通过applicationInfo.uid获取。processName则是进程名，一般为程序包名。

所以判断是否存在应用进程，则是根据processName和uid去判断是否有对应的WindowProcessController，并且WindowProcessController里面的线程不为空。

代码如下：

```java
//ActivityStackSupervisor.java
    void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            //应用进程存在
            try {
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            }
        }
    }

    //WindowProcessController.java
    IApplicationThread getThread() {
        return mThread;
    }

    boolean hasThread() {
        return mThread != null;
    }
```

**2. 还有个问题就是怎么创建进程？**

之前说了Zygote进程是所有进程的父进程，所以就要通知Zygote去fork一个新的进程，服务于这个应用。

```java
        //ZygoteProcess.java
    private Process.ProcessStartResult attemptUsapSendArgsAndGetResult(
            ZygoteState zygoteState, String msgStr)
            throws ZygoteStartFailedEx, IOException {
        try (LocalSocket usapSessionSocket = zygoteState.getUsapSessionSocket()) {
            final BufferedWriter usapWriter =
                    new BufferedWriter(
                            new OutputStreamWriter(usapSessionSocket.getOutputStream()),
                            Zygote.SOCKET_BUFFER_SIZE);
            final DataInputStream usapReader =
                    new DataInputStream(usapSessionSocket.getInputStream());

            usapWriter.write(msgStr);
            usapWriter.flush();

            Process.ProcessStartResult result = new Process.ProcessStartResult();
            result.pid = usapReader.readInt();
            // USAPs can't be used to spawn processes that need wrappers.
            result.usingWrapper = false;

            if (result.pid >= 0) {
                return result;
            } else {
                throw new ZygoteStartFailedEx("USAP specialization failed");
            }
        }
    }
```

可以看到，这里其实是通过socket和Zygote进行通信，BufferedWriter用于读取和接收消息。这里将要新建进程的消息传递给Zygote，由Zygote进行fork进程，并返回新进程的pid。

这里的Socket 是registerZygoteSocket方法创建的一个Server的Socket，接收AMS请求，用来创建App进程。

**可能又会有人问了？fork是啥？为啥这里又变成socket进行IPC通信，而不是Bindler了？**

* 首先，fork\(\)是一个方法，是类Unix操作系统上创建进程的主要方法。用于创建子进程\(等同于当前进程的副本\)。
* 那为什么fork的时候不用Binder而用socket了呢？主要是因为fork不允许存在多线程，Binder通讯偏偏就是多线程。

**问题总是在不断产生，总有好奇的朋友会接着问，为什么fork不允许存在多线程？**

* 防止死锁。其实你想想，多线程+多进程，听起就不咋靠谱是不。假设多线程里面线程A对某个锁lock，另外一个线程B调用fork创建了子进程，但是子进程却没有了线程A，但是锁本身却被fork了出来，那么这个锁没人可以打开了。一旦子进程中另外的线程又对这个锁进行lock，就死锁了。

### 2.4 ActivityThread闪亮登场

上文说到由Zygote进行fork进程，并返回新进程的pid。其实这过程中也实例化ActivityThread对象。一起看看是怎么实现的：

```java
    //RuntimeInit.java
   protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }
        //...
        return new MethodAndArgsCaller(m, argv);
    }
```

原来是反射！通过反射调用了ActivityThread 的 main 方法。ActivityThread大家应该都很熟悉了，代表了Android的主线程，而main方法也是app的主入口。

这不对上了！新建进程的时候就调用了，可不是主入口嘛。来看看这个主入口。

```java
public static void main(String[] args) {
        //...
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        //...

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }
        //...
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

main方法主要创建了ActivityThread，创建了主线程的Looper对象，并开始loop循环。除了这些，还要告诉AMS，我醒啦，进程创建好了！

也就是上述代码中的attach方法，最后会转到AMS的attachApplicationLocked方法，一起看看这个方法干了啥：

```java
//ActivitymanagerService.java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        //...
        ProcessRecord app;
        //...
        thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions);
        //...
        app.makeActive(thread, mProcessStats);

        //...
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        //...
    }

    //ProcessRecord.java
    public void makeActive(IApplicationThread _thread, ProcessStatsService tracker) {
        //...
        thread = _thread;
        mWindowProcessController.setThread(thread);
    }
```

这里主要做了三件事：

* bindApplication方法，主要用来启动Application。
* makeActive方法，设定WindowProcessController里面的线程，也就是上文中说过判断进程是否存在所用到的。
* attachApplication方法，启动根Activity。

### 2.5 创建Application

接着上面看，按照我们所熟知的，应用启动后，应该就是启动Applicaiton，也就是上文提到的bindApplication的方法，启动Activity。看看是不是怎么回事：

```java
        //ActivityThread#ApplicationThread
    public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, AutofillOptions autofillOptions,
                ContentCaptureOptions contentCaptureOptions) {
            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            data.autofillOptions = autofillOptions;
            data.contentCaptureOptions = contentCaptureOptions;
            sendMessage(H.BIND_APPLICATION, data);
        }

        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                }
            }
```

可以看到这里有个H，H是主线程的一个Handler类，用于处理需要主线程处理的各类消息，包括BIND\_SERVICE，LOW\_MEMORY，DUMP\_HEAP等等。接着看handleBindApplication：

```java
private void handleBindApplication(AppBindData data) {
        //...
        try {
            final ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        }
        //...
        Application app;
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        final StrictMode.ThreadPolicy writesAllowedPolicy = StrictMode.getThreadPolicy();
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;
            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers);
                }
            }

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            //...
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                            "Unable to create application " + app.getClass().getName()
                                    + ": " + e.toString(), e);
                }
            }
        }
            //...
    }
```

这里信息量就多了，一点点的看：

* 首先，创建了Instrumentation，也就是上文一开始startActivity的第一步。每个应用程序都有一个Instrumentation，用于管理这个进程，比如要创建Activity的时候，首先就会执行到这个类里面。
* makeApplication方法，创建了Application，终于到这一步了。最终会走到newApplication方法，执行Application的attach方法。

```java
public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        app.attach(context);
        return app;
    }
```

attach方法有了，onCreate方法又是何时调用的呢？马上来了：

```java
instrumentation.callApplicationOnCreate(app);

public void callApplicationOnCreate(Application app) {
    app.onCreate();
}
```

也就是创建Application-&gt;attach-&gt;onCreate调用顺序。

等等，在onCreate之前还有一句重要的代码：

```java
installContentProviders
```

这里就是启动Provider的相关代码了，具体逻辑就不分析了。

### 2.6 启动Activity

完bindApplication，该说说后续了，上文2.5说到，bindApplication方法之后执行的是attachApplication方法，最终会执行到ActivityThread的handleLaunchActivity方法：

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
                                         PendingTransactionActions pendingActions, Intent customIntent) {
        //...
        WindowManagerGlobal.initialize();
        //...
        final Activity a = performLaunchActivity(r, customIntent);
        //...
        return a;
    }
```

首先，初始化了WindowManagerGlobal，这是个啥呢？没错，就是WindowManagerService了，也为后续窗口显示等作了准备。

继续看performLaunchActivity：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //创建ContextImpl
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //创建Activity
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
        }

        try {
            if (activity != null) {
                //完成activity的一些重要数据的初始化
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }

                //设置activity的主题
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                //调用activity的onCreate方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            }
        }

        return activity;
    }
```

哇，终于看到onCreate方法了。稳住，还是一步步看看这段代码。

首先，**创建了ContextImpl对象**，ContextImpl可能有的朋友不知道是啥，ContextImpl继承自Context,其实就是我们平时用的上下文。有的同学可能表示，这不对啊，获取上下文明明获取的是Context对象。来一起跟随源码看看。

```java
//Activity.java
Context mBase;

@Override
public Executor getMainExecutor() {
    return mBase.getMainExecutor();
}

@Override
public Context getApplicationContext() {
    return mBase.getApplicationContext();
}
```

这里可以看到，我们平时用的上下文就是这个mBase，那么找到这个mBase是啥就行了：

```java
protected void attachBaseContext(Context base) {
    if (mBase != null) {
        throw new IllegalStateException("Base context already set");
    }
    mBase = base;
}

//一层层往上找

final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor) {

    attachBaseContext(context);

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }


}
```

这不就是，，，刚才一开始performLaunchActivity方法里面的attach吗？太巧了，所以这个ContextImpl就是我们平时所用的上下文。

顺便看看attach还干了啥？新建了PhoneWindow，建立自己和Window的关联，并设置了setSoftInputMode等等。

ContextImpl创建完之后，会通过类加载器创建Activity的对象，然后设置好activity的主题，最后调用了activity的onCreate方法。

## 3.App启动小结

* Launcher被调用点击事件，转到Instrumentation类的startActivity方法。（发起申请）
* Instrumentation通过跨进程通信告诉AMS要启动应用的需求。（收到请求）
* AMS反馈Launcher，让Launcher进入Paused状态。（反回确认消息）
* Launcher进入Paused状态，AMS转到ZygoteProcess类，并通过socket与Zygote通信，告知Zygote需要新建进程。（fork并创建ActivityThread）
* Zygote fork进程，并调用ActivityThread的main方法，也就是app的入口。
* ActivityThread的main方法新建了ActivityThread实例，并新建了Looper实例，开始loop循环。
* 同时ActivityThread也告知AMS，进程创建完毕，开始创建Application，Provider，并调用Applicaiton的attach，onCreate方法。（创建并启动Application）
* 最后就是创建上下文，通过类加载器加载Activity，调用Activity的onCreate方法。（启动Activity）

PS：分析启动过程，其实可以**优化启动速度**的地方有三个地方：

* Application的attach方法，MultiDexApplication会在方法里面会去执行MultiDex逻辑。所以这里可以进行MultiDex优化，比如今日头条方案就是单独启动一个进程的activity去加载MultiDex。
* Application的onCreate方法，大量三方库的初始化都在这里进行，所以我们可以开启线程池，懒加载等等。把每个启动任务进行区分，哪些可以子线程运行，哪些有先后顺序。
* Activity的onCreate方法，同样进行线程处理，懒加载。或者预创建Activity，提前类加载等等。

## 参考文章

1. 《Android进阶解密》；
2. [女儿拿着小天才电话手表问我App启动流程](https://mp.weixin.qq.com/s/1bfi-BVe23_96A2AlhaPKg)

