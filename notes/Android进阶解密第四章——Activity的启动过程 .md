## Android进阶解密第四章——Activity的启动过程

### 1.概述

Activity的启动大致分为两种，一种是目标Activity的进程存在，一种是进程不存在，需要创建进程。后者包含前者，

大致和前文的第二章介绍的差不多，启动的时候会去AMS查询进程是否存在，如果不存在则请求Zygote，通过fork的方式创建进程，启动进程后通知AMS并启动目标Activity。

### 2. Activity的启动过程

上图中使用的事Launcher其实不准确，应该是描述为当前的正在运行的进程。Android中Activity的启动流程大致如上，但是在Android10中进行了一些优化，新增了*ActivityTaskManagerService（以下简称为ATMS）*以及事务管理等。关于事务可以参考这个文章[什么是事务？事务的四个特性以及事务的隔离级别](https://cloud.tencent.com/developer/article/1133074)。流程图细节如下所示：

![](https://user-gold-cdn.xitu.io/2020/7/12/17343b491419f3e9)

#### 2.1 原Activity发起启动TargetActivity的请求

从startActivity 到startActivityForResult通过mInstrumentation.execStartActivity 到 SystemServer中：

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        ...

        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
          //将数据传递到AMS中 Android 10以后将部分功能转移到ATMS中
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

### 2.2 AMS收到对应的请求

在2.1中逻辑执行到这里进入到AMS中，Android 10中将部分功能迁移到了ATMS。下面继续看执行逻辑：

在ATMS的startActivity >startActivityAsUser等一系列的调用后到ActivityStarter中execute方法开始启动activity。分了两种情况，不过 不论startActivityMayWait还是startActivity最终都是走到下面这个startActivity方法：

```java
    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity, boolean restrictedBgActivity) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity, restrictedBgActivity);
        } finally {
            final ActivityStack currentStack = r.getActivityStack();
            startedActivityStack = currentStack != null ? currentStack : mTargetStack;

           ...
        }

        postStartActivityProcessing(r, result, startedActivityStack);
        return result;
    }
```

看到代码中有调用startActivityUnchecked，这里经过一系列的调用最终到ActivityStack中resumeTopActivityUncheckedLocked的方法，代码如下所示：

```java
//ActivityStack
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }
        boolean result = false;
        try {
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }

```

接着调用了resumeTopActivityInnerLocked

```java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        boolean pausing = getDisplay().pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
             // 暂停上一个Activity
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        ...
        //这里next.attachedToProcess()，只有启动了的Activity才会返回true
        if (next.attachedToProcess()) {
            ...
            
            try {
                final ClientTransaction transaction =
                        ClientTransaction.obtain(next.app.getThread(), next.appToken);
                ...
                //启动了的Activity就发送ResumeActivityItem事务给客户端了，后面会讲到
                transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                getDisplay().mDisplayContent.isNextTransitionForward()));
                mService.getLifecycleManager().scheduleTransaction(transaction);
               ....
            } catch (Exception e) {
                ....
                mStackSupervisor.startSpecificActivityLocked(next, true, false);
                return true;
            }
            ....
        } else {
            ....
            if (SHOW_APP_STARTING_PREVIEW) {
            	    //这里就是 冷启动时 出现白屏 的原因了：取根activity的主题背景 展示StartingWindow
                    next.showStartingWindow(null , false ,false);
                }
            // 继续当前Activity，普通activity的正常启动 关注这里即可
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
        return true;
    }

```

先对上一个Activity执行pause操作，再执行当前创建操作，代码最终进入到了ActivityStackSupervisor.startSpecificActivityLocked方法中。这里有个点注意下，启动activity前调用了next.showStartingWindow方法来展示一个window，这就是 **冷启动时 出现白屏 的原因**了。我们继续看ActivityStackSupervisor.startSpecificActivityLocked方法。

在ActivityStackSupervisor.startSpecificActivityLocked的方法中，对Target所在的进程进行了判断，是否存在对应的进程，先看进程已经启动的情况，在启动的情况下会调用realStartActivityLocked的方法：

```java
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
	
			...

                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);

                final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.icicle, r.persistentState, results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                                r.assistToken));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

                ...

        return true;
    }

```

中间有段代码如上，通过 ClientTransaction.obtain( proc.getThread(), r.appToken)获取了clientTransaction，其中参数proc.getThread()是IApplicationThread，就是前面提到的ApplicationThread在系统进程的代理，通常在一些资料中简称为ATP。

**ClientTransaction**是包含一系列的待客户端处理的事务的容器，客户端接收后取出事务并执行。

可以看下，clientTransaction.addCallback添加了LaunchActivityItem实例：

```java
//都是用来发送到客户端的
private List<ClientTransactionItem> mActivityCallbacks;

public void addCallback(ClientTransactionItem activityCallback) {
    if (mActivityCallbacks == null) {
        mActivityCallbacks = new ArrayList<>();
    }
    mActivityCallbacks.add(activityCallback);
}
```
接着看realStartActivityLocked方法，他最后调用了 mService.getLifecycleManager().scheduleTransaction(clientTransaction),这里其实获取了ClientLifecycleManager的实例，然后看下scheduleTransaction的代码，

```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }

    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }

```

两个方法的调用，其实最红就是通过ATP（ApplicationThread的代理），将Activity的启动交回到应用程序的进程中，也就是TargetActivity所在的进程中。

### 2.3 回到TargetActivity所在的进程中——线程切换以及消息处理

接着上面的分析，我们找到ApplicationThread的scheduleTransaction方法：

```java
      	//ApplicationThre
				@Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
				
				//ActivityThread
				void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        //最后到mH中，其实就是一个handler
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    		}
```

执行到ActivityThread，最后调用到mH.sendMessage(msg)。

mH是在创建ActivityThread实例时赋值的，是自定义[Handler](https://blog.csdn.net/hfy8971613/article/details/103881609)子类H的实例，也就是在ActivityThread的main方法中，并且初始化是已经主线程已经有了mainLooper，所以，**使用这个mH来sendMessage就把消息发送到了主线程**。

那么是从哪个线程发送的呢？那就要看看ApplicationThread的scheduleTransaction方法是执行在哪个线程了。根据[IPC](https://blog.csdn.net/hfy8971613/article/details/79688664)知识，我们知道，服务器的Binder方法运行在Binder的线程池中，也就是说系统进行跨进程调用ApplicationThread的scheduleTransaction就是执行在Binder的线程池中的了。

到这里，消息就在主线程处理了，那么是怎么处理Activity的启动的呢？接着看。我们找到ActivityThread.H.EXECUTE_TRANSACTION这个消息的处理，就在handleMessage方法的倒数第三个case。

取出ClientTransaction实例，调用TransactionExecutor的execute方法，在方法中继续调用executeCallbacks，executeCallbacks的方法中主要是遍历callbacks，并调用ClientTransactionItem的execute方法，而我们这里要关注的是ClientTransactionItem的子类LaunchActivityItem，看下它的execute方法：

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```
其中可以看到client.handleLaunchActivity，这里最终调用到ActivityThread的handleLaunchActivity方法。

看到这里小结一下，**ApplicationThread把启动Activity的操作，通过mH切到了主线程，走到了ActivityThread的handleLaunchActivity方法**。

#### 2.4 Activity的初始化以及生命周期

#####  2.4.1 Activity的onCreate方法 

这里就是比较熟悉的流程了可以直接看核心代码**performLaunchActivity**

```java
    /**  activity 启动的核心实现. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	//1、从ActivityClientRecord获取待启动的Activity的组件信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
				//创建ContextImpl对象
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
        	//2、创建activity实例
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            ..
        }
        try {
        	//3、创建Application对象（如果没有的话）
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            ...
            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
              
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                
                //4、attach方法为activity关联上下文环境
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                
                //5、调用生命周期onCreate
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);
            
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

        } 
        ...

        return activity;
    }
```

performLaunchActivity主要完成以下事情：

1. 从ActivityClientRecord获取待启动的Activity的组件信息
2. 通过mInstrumentation.newActivity方法使用类加载器创建activity实例
3. 通过LoadedApk的makeApplication方法创建Application对象，内部也是通过mInstrumentation使用类加载器，创建后就调用了instrumentation.callApplicationOnCreate方法，也就是Application的onCreate方法。
4. 创建ContextImpl对象并通过activity.attach方法对重要数据初始化，关联了Context的具体实现ContextImpl，attach方法内部还完成了window创建，这样Window接收到外部事件后就能传递给Activity了。
5. 调用Activity的onCreate方法，是通过 mInstrumentation.callActivityOnCreate方法完成。

**到这里Activity的onCreate方法执行完**，后面就是类似的onStart 和onResume流程了。

##### 2.4.2 onStart和onResume

这里直接看handleResumeActivity，他主要做了两件事：

1. 调用生命周期：通过performResumeActivity方法，内部调用生命周期onStart、onResume方法
2. 设置视图可见：通过activity.makeVisible方法，添加[window](https://blog.csdn.net/hfy8971613/article/details/103241153)、设置可见。（所以视图的真正可见是在onResume方法之后）

先看第一点，handleResumeActivity 调用到Activity.performResume:

```java
    final void performResume(boolean followedByPause, String reason) {
        dispatchActivityPreResumed();
        //内部会走onStart
        performRestart(true /* start */, reason);
        ...
        // 走onResume
        mInstrumentation.callActivityOnResume(this);
        ...
		//这里是走fragment的onResume
        mFragments.dispatchResume();
        mFragments.execPendingActions();

        ...
    }
```

先调用了performRestart()，performRestart()又会调用performStart()，performStart()内部调用了mInstrumentation.callActivityOnStart(this)，也就是**Activity的onStart()**方法了。

然后是mInstrumentation.callActivityOnResume，也就是**Activity的onResume()**方法了。到这里启动后的生命周期走完了。

然后看第二点，activity.makeVisible的方法，

```java
//Activity
    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }

```

这里把activity的顶级布局mDecor通过windowManager.addView()方法，把视图添加到[window](https://blog.csdn.net/hfy8971613/article/details/103241153)，并设置mDecor可见。到这里视图是真正可见了。值得注意的是，视图的真正可见是在onResume方法之后的。

另外一点，Activity视图渲染到Window后，会设置window焦点变化，先走到DecorView的onWindowFocusChanged方法，最后是到Activity的onWindowFocusChanged方法，表示首帧绘制完成，此时Activity可交互。

至此Activity的启动（不需要创建进程的情况下），就此结束。

#### 2.5 对上文中提到的Activity启动中的提到的一些类

类似LaunchActivityItem的XXXActivityItem如下：

- LaunchActivityItem 远程App端的onCreate生命周期事务
- ResumeActivityItem 远程App端的onResume生命周期事务
- PauseActivityItem 远程App端的onPause生命周期事务
- StopActivityItem 远程App端的onStop生命周期事务
- DestroyActivityItem 远程App端onDestroy生命周期事务

另外梳理过程中涉及的几个类：

- ClientTransaction 客户端事务控制者
- ClientLifecycleManager 客户端的生命周期事务控制者
- TransactionExecutor 远程通信事务执行者

分别会走到ActivityThread中handlexxxActivity对应的方法中。

| 类名                                   | 作用                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| ActivityThread                         | 应用的入口类，系统通过调用main函数，开启消息循环队列。ActivityThread所在线程被称为应用的主线程（UI线程） |
| ApplicationThread                      | 是ActivityThread的内部类，继承IApplicationThread.Stub，是一个IBinder，是ActiivtyThread和AMS通信的桥梁，AMS则通过代理调用此App进程的本地方法，运行在Binder线程池 |
| H                                      | 继承Handler，在ActivityThread中初始化，即主线程Handler，用于主线程所有消息的处理。本片中主要用于把消息从Binder线程池切换到主线程 |
| Intrumentation                         | 具有跟踪application及activity生命周期的功能，用于监控app和系统的交互 |
| ActivityManagerService                 | Android中最核心的服务之一，负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要，它本身也是一个Binder的实现类。 |
| ActivityTaskManagerService             | 管理activity及其容器（task, stacks, displays）的系统服务（Android10中新增，分担了AMS的部分职责） |
| ActivityStarter                        | 用于解释如何启动活动。该类收集所有逻辑，用于确定Intent和flag应如何转换为活动以及相关的任务和堆栈 |
| ActivityStack                          | 用来管理系统所有的Activity，内部维护了Activity的所有状态和Activity相关的列表等数据 |
| ActivityStackSupervisor                | 负责所有Activity栈的管理。AMS的stack管理主要有三个类，ActivityStackSupervisor，ActivityStack和TaskRecord |
| ClientLifecycleManager                 | 客户端生命周期执行请求管理                                   |
| ClientTransaction                      | 是包含一系列的 待客户端处理的事务 的容器，客户端接收后取出事务并执行 |
| LaunchActivityItem、ResumeActivityItem | 继承ClientTransactionItem，客户端要执行的事务信息，启动activity |

###  3. 根Activity的启动（需要创建进程）

#### 3.1 请求创建新的进程

在上文中提到了一个ActivityStackSupervisor的startSpecificActivityLocked方法中会判断是否需要创建一个进程。这里讲一下需要创建进程的情况。

比如说点击桌面上的图标启动对应的App，前面过程都一样直接看到startSpecificActivityLocked的方法中：

```java
    void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
            	//有应用进程就启动activity
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            knownToBeDead = true;
        }

        ...
        
        try {
            if (Trace.isTagEnabled(TRACE_TAG_ACTIVITY_MANAGER)) {
                Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "dispatchingStartProcess:"
                        + r.processName);
            }
            // Post message to start process to avoid possible deadlock of calling into AMS with the
            // ATMS lock held.
            // 上面的wpc != null && wpc.hasThread()不满足的话，说明没有进程，就会去创建进程
            final Message msg = PooledLambda.obtainMessage(
                    ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                    r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
            mService.mH.sendMessage(msg);
        } finally {
            Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
        }
    }

```

这里看下wpc.hasThread()，看下WindowProcessController的hasThread方法：

```java
    // The actual proc...  may be null only if 'persistent' is true (in which case we are in the
    // process of launching the app)
    private IApplicationThread mThread;
    
    boolean hasThread() {
        return mThread != null;
    }
```

前面已有说明，IApplicationThread是ApplicationThread在客户端（app）在服务端（系统进程）的代理，这里判断 **IApplicationThread不为空 就代表进程已存在**，为啥这么判断呢？这里先猜测，进程创建后，一定会有给IApplicationThread赋值的操作，这样就符合这个逻辑了。我们继续看，瞅瞅进程是如何创建的，以及创建后是否有给IApplicationThread赋值的操作。

通过ActivityTaskManagerService中的mH的sendMessage方法，最后调用到AMS中**startProcessLocked**,

```java
    public static ProcessStartResult start(@NonNull final String processClass,
                                           @Nullable final String niceName,
                                           int uid, int gid, @Nullable int[] gids,
                                           int runtimeFlags,
                                           int mountExternal,
                                           int targetSdkVersion,
                                           @Nullable String seInfo,
                                           @NonNull String abi,
                                           @Nullable String instructionSet,
                                           @Nullable String appDataDir,
                                           @Nullable String invokeWith,
                                           @Nullable String packageName,
                                           @Nullable String[] zygoteArgs) {
        return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, packageName,
                    /*useUsapPool=*/ true, zygoteArgs);
    }

```

ZYGOTE_PROCESS是用于保持与Zygote进程的通信状态，发送socket请求与Zygote进程通信。**Zygote进程**是**进程孵化器**，用于创建进程。

- Zygote通过fork创建了一个进程
- 在新建的进程中创建Binder线程池（此进程就支持了Binder IPC）
- 最终是通过反射获取到了ActivityThread类并执行了main方法

#### 3.2 启动根Activity

先看下上文提到的ActivityThread中的main方法，

```java
    final H mH = new H();
    
    public static void main(String[] args) {
        ...
        //1、准备主线程的Looper
        Looper.prepareMainLooper();

        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        //这里实例化ActivityThread，也就实例化了上面的mH，就是handler。
        ActivityThread thread = new ActivityThread();
      	//关联ApplicationThread到AMS
        thread.attach(false, startSeq);

		//获取handler
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        ...
        //主线程looper开启
        Looper.loop();
		//因为主线程的Looper是不能退出的，退出就无法接受事件了。一旦意外退出，会抛出异常
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }


```

开启主线程消息循环，并创建ActivityThread的实例，内部包含ApplicationThread的创建。thread.attach方法：

```java
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
            	//把ApplicationThread实例关联到AMS中
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
        } 
        ...
    }
```

经过一系列调用最后到AMS中attachApplicationLocked中：

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {

			...
				//1、IPC操作，创建绑定Application
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions);
			...
            // 2、赋值IApplicationThread
            app.makeActive(thread, mProcessStats);
			...
			
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
            	//3、通过ATMS启动 根activity
                didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
		...
}
```

看注释：

AMS的attachApplicationLocked方法主要三件事：

- 调用IApplicationThread的bindApplication方法，IPC操作，创建绑定Application；
- 通过makeActive方法赋值IApplicationThread，即验证了上面的猜测（创建进程后赋值）；
- 通过ATMS启动 **根activity**

**Step1 绑定Application **

IApplicationThread的bindApplication方法实现是客户端的ApplicationThread的bindApplication方法，它又使用H转移到了ActivityThread的handleBindApplication方法（从Binder线程池转移到主线程），handleBindApplication

```java
private void handleBindApplication(AppBindData data) {
	...
	            final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);

          
            final ContextImpl instrContext = ContextImpl.createAppContext(this, pi,
                    appContext.getOpPackageName());

            try {
            	//创建Instrumentation
                final ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } 
            ...
            final ComponentName component = new ComponentName(ii.packageName, ii.name);
            mInstrumentation.init(this, instrContext, appContext, component,
                    data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
	...
			//创建Application
            app = data.info.makeApplication(data.restrictedBackupMode, null);

	...
            mInitialApplication = app;
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
    ...
            try {
            	//内部调用Application的onCreate方法
                mInstrumentation.callApplicationOnCreate(app);
            }
	...
}

```

主要就是创建Application，并且调用生命周期onCreate方法。你会发现在前面介绍的ActivityThread的performLaunchActivity方法中，也有同样的操作，只不过会先判断Application是否已存在。也就是说，**正常情况下Application的初始化是在handleBindApplication方法中的，并且是创建进程后调用的。performLaunchActivity中只是做了一个检测，异常情况Application不存在时才会创建。**

这里注意一点，创建Application后，内部会attach方法，attach内部会调用attachBaseContext方法，**attachBaseContext**方法是我们能接触到的一个方法，接着才是onCreate方法.

**Step2 关联**

**Step2. 关联ApplicationThread到AMS**

先看下makeActive方法：

```java
public void makeActive(IApplicationThread _thread, ProcessStatsService tracker) {
    ...
    thread = _thread;
    mWindowProcessController.setThread(thread);
}	
```
看到使用mWindowProcessController.setThread(thread)确实完成了IApplicationThread的赋值。这样就可以依据IApplicationThread是否为空判断进程是否存在了。

**Step3 启动根Activity**

这里又回到mStackSupervisor.realStartActivityLocked方法，和上面的流程一样。



### 4.总结

看一下大厂面试官的图：

<img src="https://github.com/yyht800/SiGuoYa/blob/master/res/drawable/Activity%E5%90%AF%E5%8A%A8%E5%BD%A9%E5%9B%BE.jpg?raw=true" style="zoom:80%;" />

简单的流程可以参考上面的图，稍加修饰，带上上面总结如下：

1. 请求进程发起请求
2. 通过IPC到达SystemServer进程，ATMS收到消息，然后将参数给ActivityStarter
3. ActivityStarter将信息解析出来（intent和flag之类的）进入到ActivityStack中
4. ActivityStack是管理Activity的状态的（一个Activity对应一个ActivityRecord，里面包含Activity的相关信息）在这里做了两件事，通知上一个Activity onPause，（这里还存在一个白屏的原因，当Activity没有背景的时候，会默认给一个）。
5. 然后会进入到ActivityStackSupervisor（内部持有几个ActivityStack，前台和创建中的stack，看官方文档，有个TODO，描述是这里东西太冗余，会将部分功能迁移出去）内部需要注意的几点：
   1. startSpecificActivityLocked中判断进程是否存在，不存在则创建进程，存在则下一步
   2. realStartActivityLocked中通过Binder通信到目标进程中ApplicationThread
6. 通过mH切换线程到主线程中，ActivityThread中。
7. 然后就是执行LaunchActivity之类的操作了



创建进程的流程：

1. 通知Zygote需要创建一个进程
2. Zygote通过fork创建了一个进程
3. 在新建的进程中创建Binder线程池（此进程就支持了Binder IPC）
4. 最终是通过反射获取到了ActivityThread类并执行了main方法
5. 在ActivityThread的main方法中，
   1. 创建ActivitThread的实例
   2. 创建消息循环机制
6. 调用attach方法，最后到AMS中
   1. 创建并绑定Application
   2. ApplicationThread关联到AMS
7. 通过ATMS启动根Activity

### 参考文章

1. [Activity的启动过程详解（基于Android10.0）](https://juejin.cn/post/6847902222294990862#heading-2)
2. Android进阶解密第四章
3. Top大厂面试官——Activity的启动流程





