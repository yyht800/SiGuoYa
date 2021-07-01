# Android进阶解密第五章——理解Context的上下文

## 1. 概述

![](https://upload-images.jianshu.io/upload_images/10107787-ab22cce86fa4f1da.png)

一图胜千言，**Context 使用了装饰模式，除了 ContextImpl 外，其他 Context 都是 ContextWrapper 的子类。**需要主题的的Activity继承自ContextThemeWrapper，而Application和Service都是ContextWrapper的子类。

## 2. Application的Context的创建

### 2.1 performLaunchActivity需求Application

在Activity的启动章节有提到过，Application的创建在ActivityThread的performLaunchActivity方法中：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
       ...

        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ...
        return activity;
    }
```

关注重点就是`packageInfo.makeApplication`,这里是ActivityClientRecord的成员变量packageInfo是LoadedApk的类型的，这里直接看到LoadedApk的makeApplication方法中：

```java
        @UnsupportedAppUsage
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        //-----1----- 判断Application是否已经完成创建了，已经有则直接返回，否则下一步
        if (mApplication != null) {
            return mApplication;
        }
          ...
        try {
            java.lang.ClassLoader cl = getClassLoader();
            ...
            //-----2----- 创建ContextImpl
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            //-----3----- 创建Application
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            //-----4----- 
            appContext.setOuterContext(app);
        } catch (Exception e) {
            ...
        }
        mActivityThread.mAllApplications.add(app);
        //-----5-----
        mApplication = app;

           ...
    }
```

重点看下注释4：Application贼值给 Contextlmpl的 Context类型的成员变量 mOuterContext，这 样 Contextlmpl 中也包含了 Application 的引用 。

注释5：将Application赋值给mApplication，这样下次直接可以在注释1处返回了。

接着看下注释3，如何创建Application。

### 2.2 如何创建Application

在Instrumentation的newApplication方法中,最后到Application的attach方法

```java
@UnsupportedAppUsage
    /* package */ final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}

//ContextWrapper
protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
}
```

整理的mBase就是一路传进来的ContextImpl。将 Contextimpl 赋值给 ContextWrapper 的 Context 类型的成员变 量 mBase，就建立了关联关系。

### 2.3 如何获取Application的Context

从2.2中可以知道最后ContextImpl和Application即ContextWrapper建立了关联关系，这里看下getApplicationContext的调用顺序：

Step1：ContextWrapper.getApplicationContext

Step2: ContextImpl.getApplicationContext

Step3: LoadedApk.getApplication

```java
@Override
public Context getApplicationContext() {
      //从上文中可以看到 mBase实际就是ContextImpl
    return mBase.getApplicationContext();
}


@Override
public Context getApplicationContext() {
    return (mPackageInfo != null) ?
            mPackageInfo.getApplication() : mMainThread.getApplication();
}

Application getApplication() {
    return mApplication;
}
```

## 3. Activity的Context

直接来看ActivityThread的perfonnLaunchActivity：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        //-----1-----
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //-----2-----
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
           ...

        }

        try {
            ...
            if (activity != null) {
                ....
                //-----3-----
                appContext.setOuterContext(activity);
                //-----4-----
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                ...
                activity.mCalled = false;
                //-----5-----
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ....
            }
            ...

        } 
        ...
        return activity;
    }
```

注释1：创建了一个Activity的ContextImpl

注释2：通过`mInstrumentation.newActivity`创建了`Activity`实例

注释3：将activity赋值给ContextImpl的`mOuterContext`, 这样我们就可以通过ContextImpl来调用Activity中的方法和变量

注释4：`activity.attach`是一个很重要的方法，此处将`appContext`传入到Activity中，`attach`方法如下

注释5：调用Activity的onCreate的方法。

```java
    @UnsupportedAppUsage
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
        //-----1-----
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
        //-----2-----
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        //-----3-----
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        ...
        //-----4-----
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        //-----5-----
        mWindowManager = mWindow.getWindowManager();
        ...
    }
```

注释1：调用了`activity.attachBaseContext`,内部代码如下, 而`super.attachBaseContext(newBase)` 会根据继承关系一直调用到`ContextWrapper`, 然后将`ContextImpl`类型的Context传进去，赋值给`mBase`

注释2：对Window的初始化，我们发现它的实现类有且仅有PhoneWindow。这是我们应用程序的窗口

注释3：mWindow.setCallback\(this\), 对window设置了一个回调，这个回调中有很多我们经常使用到的方法，比如onAttachedToWindow,onDetachedFromWindow,dispatchTouchEvent等。

注释4：将给Window中设置了一个WindowManager。

注释5：将Window中刚才设置的WindowManager获取出来赋值给Activity中的成员变量mWindowManager。而Activity中通过getWindowManager\(\)方法获取的WindowManager也就是PhoneWindow中的WindowManager。

## 4. Service的Context

Service的Context和Activity的基本差不多可以直接从`ActivityThread.handleCreateService`开始分析：

```java
@UnsupportedAppUsage
    private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            //-----1-----
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            //-----2-----
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);

            //-----3-----
            context.setOuterContext(service);
            //-----4-----
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //-----5-----
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            //-----6-----
            service.onCreate();
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
```

注释1:创建一个service对象。

注释2:创建一个Service的ContextImpl对象。

注释3:setOuterContext将service对象赋值给ContextImpl中的mOuterContext, 这样ContextImpl就可以访问service中的方法和变量了。

注释4:创建一个Application，在前面Android深入理解Context讲过。

注释5:调用service.attach，将创建的context传递进service中。

注释6:调用service.onCreate\(\)。

我们主要看下注释4的代码部分，进入service.attach：

```java
public abstract class Service extends ContextWrapper {
    @UnsupportedAppUsage
    public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {
        attachBaseContext(context);
        mThread = thread;           // NOTE:  unused - remove?
        mClassName = className;
        mToken = token;
        mApplication = application;
        mActivityManager = (IActivityManager)activityManager;
        mStartCompatibility = getApplicationInfo().targetSdkVersion
                < Build.VERSION_CODES.ECLAIR;
    }
}

public class ContextWrapper extends Context {
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
}
```

最终还是将Service类型的ContextImpl赋值给了ContextWrapper中的`mBase`变量，这样我们调用ContextWrapper中的方法实际就是调用`mBase`这个真正的ContextImpl中的方法。

注释4：Application贼值给 Contextlmpl的 Context类型的成员变量 mOuterContext，这 样 Contextlmpl 中也包含了 Application 的引用

## 参考文章

1. [理解Android Context](http://gityuan.com/2017/04/09/android_context/)
2. Android 进阶解密第五章

