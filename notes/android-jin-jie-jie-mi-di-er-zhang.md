# Android进阶解密——第二章 Android系统的启动

### 概述

系统启动相关的东西比较多，下面简单介绍下系统开机的过程，可以先看下

![](http://gityuan.com/images/android-arch/android-boot.jpg)

### 1. 启动电源以及系统启动

当电源按下时引导芯片代码会从预定义的地方（固化在ROM）开始执行，加载引导程序BootLoader到RAM，然后执行。

### 2. 引导程序BootLoader

它是Android操作系统开始运行前的一个小程序，主要将操作系统OS拉起来并进行

### 3. Linux内核启动

当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。此外，还启动了Kernel的swapper进程（pid = 0）和kthreadd进程（pid = 2）。下面分别介绍下它们：

* swapper进程：又称为idle进程，系统初始化过程Kernel由无到有开创的第一个进程, 用于初始化进程管理、内存管理，加载Binder Driver、Display、Camera Driver等相关工作。
* kthreadd进程：Linux系统的内核进程，是所有内核进程的鼻祖，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。

当内核完成系统设置时，它首先在系统文件中寻找init.rc文件，并启动init进程。

### 4. init进程启动

#### 4.1 创建和挂载启动所需的文件目录

其中挂载了tmpsf、devpts、proc、sysfs和selinuxfs共5种文件系统（它们均是系统运行时目录）

#### 4.2 对属性服务进行初始化

```text
property_init()
```

对属性服务进行初始化，

#### 4.3 设置子进程信号处理函数，如果子进程（zygote进程）异常退出，init进程会调用该函数中设定的信号处理函数来进行处理

```text
sigchld_handler_init();
```

为了防止僵尸进程的出现，系统会在子进程暂停和终止的时候发出SIGCJHLD信号，该函数就是用来接收SIGCHLD信号的，注意它仅处理进程终止的SIGCHLD信号

* **如果是Zygote进程终止了，则会如何？**

  sigchld\_handler\_init\(\)函数内部会找到Zygote进程并移除所有的Zygote进程的信息，在重启Zygote服务的启动脚本（如init.zygote64.rc）中带有onrestart选项的服务。

#### 4.4 启动属性服务（其中会启动servicemanager\(binder服务大管家\)、bootanim\(开机动画\)等重要服务）

* 1、首先，创建非阻塞式的Socket，并返回property\_set\_fd文件描述符。
* 2、使用listen\(\)函数去监听property\_set\_fd，此时Socket即成为属性服务端，并且它最多同时可为8个试图设置属性的用户提供服务。
* 3、使用epoll\(\)来监听property\_set\_fd：当property\_set\_fd中有数据到来时，init进程将调用handle\_property\_set\_fd\(\)函数进行处理。在Andorid 8.0的源码中则在handle\_property\_set\_fd\(\)函数中添加了handle\_property\_set函数做进一步封装处理。
* 4、系统属性分为两种属性，即普通属性和控制属性。控制属性用来执行一些命令，比如开机的动画就使用了这种属性。在handle\_property\_set\_fd\(\)函数中会先判断如果属性名是以”ctl.”开头的，就说明是控制属性，如果客户端权限满足，则会调用handle\_control\_message\(\)函数来修改控制属性。如果是普通属性，则会在客户端全面满足的条件下调用property\_set函数来修改普通属性。
* 5、在property\_set中会先从属性存储空间中查找该属性，如果有，则更新，否则添加该属性。此外，如果名称是以”ro”开头（表示只读，不能修改），直接返回，如果名称是以”persist.”开头，则写入持久化属性。

#### 4.5 解析init.rc的配置文件

```text
parser.ParseConfig("/init.rc");
```

* **init.rc是什么？**

  它是由Android初始化语言编写的一个非常重要的配置脚本文件。Android初始化语言主要包含5种类型的语句：

  * Action（常用）
  * Service（常用）
  * Command
  * Option
  * Import

  这里了解下Action和Service的格式：

  \`\`\` on  \[&& \]\* //设置触发器  
  
   //动作触发之后要执行的命令 ...

service   \[  \]\* //&lt;执行程序路径&gt;&lt;传递参数&gt;  
 //option是service的修饰词，影响什么时候、如何启动services  
  
...

```text
注意：Android8.0对init.rc文件进行了拆分，每个服务对应一个rc文件。

- **init.rc中的Action、Service语句都有相应的XXXParser类来解析，即ActionParser、ServiceParser。那么ServiceParser是如何解析Service语句的？**
  - 1.先使用ParseSection()方法根据参数创建出一个Service对象。
  - 2.再使用ParseLineSection()方法解析Service语句中的每一个子项，将其中的内容添加到Service对象中。
  - 3.然后，在解析完所有数据后，会调用EndSection函数，内部会执行ServiceManager的AddService函数，最终将Service对象加入vector类型的Service链表中。

- **init.rc启动Zygote**
```

... on nonencrypted  
exec - root -- /system/bin/update\_verifier nonencrypted  
class\_start main //1  
class\_start late\_start ...

```text
  1. 注释1：使用class_start这个COMMAND去启动名字为main的service（即zygote）。其中class_start对应do_class_start()函数。
```

static Result do\_class\_start\(const BuiltinArguments& args\) { // Starting a class does not start services which are explicitly disabled. // They must be started individually. for \(const auto& service : ServiceList::GetInstance\(\)\) { if \(service-&gt;classnames\(\).count\(args\[1\]\)\) { // 2 if \(auto result = service-&gt;StartIfNotDisabled\(\); !result\) { LOG\(ERROR\) &lt;&lt; "Could not start service'" &lt;&lt; service-&gt;name\(\) &lt;&lt; "' as part of class '" &lt;&lt; args\[1\] &lt;&lt; "': " &lt;&lt; result.error\(\); } } } return Success\(\); }

```text
    2. 注释2：在system/core/init/builtins.cpp的do_class_start()函数中会遍历前面的Vector类型的Service链表，找到classname为main的Zygote，并调用system/core/init/service.cpp中的startIfNotDisabled()函数。
```

bool Service::StartIfNotDisabled\(\) { if \(!\(flags _& SVC\_DISABLED\)\) { return Start\(\); } else { flags_ \|= SVC\_DISABLED\_START; } return Success\(\); }

```text
  3. 如果Service没有再其对应的rc文件中设置disabled选项，则会调用Start()启动该Service。
```

bool Service::Start\(\) {

```text
  ...


  if (flags_ & SVC_RUNNING) {
      if ((flags_ & SVC_ONESHOT) && disabled) {
          flags_ |= SVC_RESTART;
      }
      // 如果不是一个错误，尝试去启动一个已经启动的服务
      return Success();
  }

  ...

  // 判断需要启动的Service的对应的执行文件是否存在，不存在则不启动该Service
  struct stat sb;
  if (stat(args_[0].c_str(), &sb) == -1) {
      flags_ |= SVC_DISABLED;
      return ErrnoError() << "Cannot find '" << args_[0] << "'";
  }

  ...

  // fork函数创建子进程
  pid_t pid = fork();
  // 运行在子进程中
  if (pid == 0) {
      umask(077);
      ...

      // 在ExpandArgsAndExecv函数里进行了参数装配并使用了execve()执行程序
      if (!ExpandArgsAndExecv(args_)) {
          PLOG(ERROR) << "cannot execve('" << args_[0] << "')";
      }

       _exit(127);
  }
  ...
  return true;
```

}

static bool ExpandArgsAndExecv\(cons std::vector& args\) { std::vector expanded\_args; std::vector c\_strings;

```text
  expanded_args.resize(args.size());
  c_strings.push_back(const_cast<char*>(args[0].data()));
  for (std::size_t i = 1; i < args.size(); ++i) {
      if (!expand_props(args[i], &expanded_args[i])) {
          LOG(FATAL) << args[0] << ": cannot expand '" << args[i] << "'";
      }
      c_strings.push_back(expanded_args[i].data());
  }
  c_strings.push_back(nullptr);

  // 最终通过execve执行程序
  return execv(c_strings[0], c_strings.data()) == 0;
```

}

```text
  4. 在Start()函数中，如果Service已经运行，则不再启动。如果没有，则使用fork()函数创建子进程，并返回pid值。当pid为0时，则说明当前代码逻辑在子进程中运行，最然后会调用execve()函数去启动子进程，并进入该Service的main函数中，如果该Service是Zygote，则会执行Zygote的main函数。（对应frameworks/base/cmds/app_process/app_main.cpp中的main()函数）
```

int main\(int argc, char\* const argv\[\]\) { ... if \(zygote\) { runtime.start\("com.android.internal.os.ZygoteInit", args, zygote\); } else if \(className\) { runtime.start\("com.android.internal.os.RuntimeInit", args, zygote\); } else { fprintf\(stderr, "Error: no class name or --zygote supplied.\n"\); app\_usage\(\); LOG\_ALWAYS\_FATAL\("app\_process: no class name or --zygote supplied."\); } }

```text
  5. 最后，调用runtime的start函数启动Zygote。

#### 4.5 init 启动进程的总结

经过以上的分析，init进程的启动过程主要分为以下三部：

- 1、创建和挂载启动所需的文件目录。
- 2、初始化和启动属性服务。
- 3、解析init.rc配置文件并启动Zygote进程。



### 5. Zygote进程的启动

#### 5.1 Zygote概述

Zygote进程在启动的时候会创建DVM或者ART，因此通过fork而创建的应用程序进程和SystemServer进程可以在内部获取一个DVM或者ART的实例副本。由于应用程序和SystemServer都是从它fork出来的，我们也称之为孵化器。

#### 5.2 Zygote启动脚本

由上文可知，在init进程中启动了Zygote进程，在init.rc文件中采用了如下所示的Import类型语句来引入Zygote启动脚本：
```

import /init.${ro.zygote}.rc

```text
这里根据属性ro.zygote的内容来引入不同的Zygote启动脚本。从Android 5.0开始，Android开始支持64位程序，Zygote有了32/64位之别，ro.zygote属性的取值有4种：

- init.zygote32.rc （对应32位的系统）
- init.zygote32_64.rc （兼容32位和64位的系统，主模式是32位，这里会启动两个进程一个是名字为zygote，另一个是zygote_secondary）
- init.zygote64.rc （对应64位系统）
- init.zygote64_32.rc 

注意：上面的Zygote的启动脚本都存放在system/core/rootdir目录中。

#### 5.3 Zygote的启动流程

- **AppRuntime.start()**

在init启动Zygote时主要是调用app_main.cpp的main函数中的AppRuntime.start()方法来启动Zygote进程的，我们先看到app_main.cpp的main函数：
```

int main\(int argc, char _const argv\[\]\) { ... while \(i &lt; argc\) { const char_ arg = argv\[i++\];

```text
    if (strcmp(arg, "--zygote") == 0) {
    // 1 判断是否运行在zygote进程中
        zygote = true;
        niceName = ZYGOTE_NICE_NAME;
    } else if (strcmp(arg, "--start-system-server") == 0) {
        // 2 判断是否运行在startSystemServer进程中
        startSystemServer = true;
    } else if (strcmp(arg, "--application") == 0) {
        // 3 判断是否运行在application中
        application = true;
    } else if (strncmp(arg, "--nice-name=", 12) == 0) {
        niceName.setTo(arg + 12);
    } else if (strncmp(arg, "--", 2) != 0) {
        className.setTo(arg);
        break;
    } else {
        --i;
        break;
    }
}

...

// 4 如果运行在 Zygote 进程中
if (zygote) {
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
} else if (className) {
    runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
} else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
}
```

}

```text
由于Zygote进程都是通过fork自身来创建子进程的，这样Zygote进程和由它fork出来的子进程都会进入app_main.cpp的main函数中，所以在mian函数中，首先会判断当前运行在哪个进程，

1. 在注释1处，会判断参数arg中释放包含了”–zygote”，如果包含了，则说明main函数是运行在Zygote进程中的并会将zygote标记置为true。
2. 在注释2处会判断参数arg中是否包含了”–start-system-server”，如果包含了则表示当前是处在SystemServer进程中并将startSystemServer设置为true。
3. 同理在注释3处会判断参数arg是否包含”–application”，如果包含了说明当前处在应用程序进程中并将application标记置为true。
4. 最后在注释4处，当zygote标志是true的时候，也就是当前正处在Zygote进程中时，则使用AppRuntime.start()函数启动Zygote进程。

- **AndroidRuntime.start()**
```

void AndroidRuntime::start\(const char\* className, const Vector& options, bool zygote\) { ...

```text
/* start the virtual machine */
JniInvocation jni_invocation;
jni_invocation.Init(NULL);
JNIEnv* env;
// 1 启动java虚拟机
if (startVm(&mJavaVM, &env, zygote) != 0) {
    return;
}
onVmCreated(env);

/*
 * 2、为java虚拟机注册jni方法
 */
if (startReg(env) < 0) {
    ALOGE("Unable to register all android natives\n");
    return;
}

...
// 3、从上一步可以得知 com.android.internal.os.ZygoteInit
classNameStr = env->NewStringUTF(className);
assert(classNameStr != NULL);
env->SetObjectArrayElement(strArray, 0, classNameStr);

for (size_t i = 0; i < options.size(); ++i) {
    jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
    assert(optionsStr != NULL);
    env->SetObjectArrayElement(strArray, i + 1, optionsStr);
}

/*
 * Start VM.  This thread becomes the main thread of the VM, and will
 * not return until the VM exits.
 */
// 4、将className 的”.“替换为"/"
char* slashClassName = toSlashClassName(className != NULL ? className : "");
jclass startClass = env->FindClass(slashClassName);
if (startClass == NULL) {
    ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
    /* keep going */
} else {
    // 6 找到zygote的main方法
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
        "([Ljava/lang/String;)V");
    if (startMeth == NULL) {
        ALOGE("JavaVM unable to find main() in '%s'\n", className);
        /* keep going */
    } else {
        // 7、通过jni调用zygote的main方法
        env->CallStaticVoidMethod(startClass, startMeth, strArray);
```

## if 0

```text
        if (env->ExceptionCheck())
            threadExitUncaughtException(env);
```

## endif

```text
    }
}
free(slashClassName);

...
```

}

```text
1. 使用startVm函数来启动弄Java虚拟机，
2. 使用startReg函数为Java虚拟机注册JNI方法。
3. 在注释3处的classNameStr是传入的参数，值为com.android.internall.os.ZygoteInit。
4. 然后在注释4处使用toSlashClassName函数将className的”.”替换为”/“，替换后的值为com/android/internal/os/ZygoteInit。
5. 接着根据第4步找到ZygoteInit并在注释5处找到ZygoteInit的main函数，
6. 在注释6处使用JNI调用ZygoteInit的main函数，之所以这里要使用JNI，是因为ZygoteInit是java代码。最终，Zygote就从Native层进入了Java FrameWork层。在此之前，并没有任何代码进入Java FrameWork层面，因此可以认为，Zygote开创了java FrameWork层。

- **Zygoteinit.java中的main方法**

上一步是从Native进入到FrameWork层，下面看下FrameWork层中Zygoteinit.java的代码

```java
public static void main(String argv[]) {

    ...

    try {
        ...

        // 1 创建了一个名为”zygote”的Server端的Socket，它用来等待AMS请求Zygote来创建新的应用程序进程。
        zygoteServer.registerServerSocketFromEnv(socketName);
        // In some configurations, we avoid preloading resources and classes eagerly.
        // In such cases, we will preload things prior to our first fork.
        if (!enableLazyPreload) {
            bootTimingsTraceLog.traceBegin("ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());

            // 2 预加载类和资源
            preload(bootTimingsTraceLog);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());
            bootTimingsTraceLog.traceEnd(); // ZygotePreload
        } else {
            Zygote.resetNicePriority();
        }

        ...

        if (startSystemServer) {
            // 3 启动systemServer进程
            Runnable r = forkSystemServer(abiList, socketName, zygoteServer);

            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }

        Log.i(TAG, "Accepting command socket connections");

        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        // 4 等待AMS请求
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with exception", ex);
        throw ex;
    } finally {
        zygoteServer.closeServerSocket();
    }

    // We're in the child process and have exited the select loop. Proceed to execute the
    // command.
    if (caller != null) {
        caller.run();
    }
}
```

这里ZygoteInit的main方法主要做了4件事：

1. 创建一个Server端的Socket
2. 预加载类和资源
3. 启动SystemServer进程
4. 等待AMS请求创建新的应用进程

#### 5.4Zygote启动总结

从以上的分析可以得知，Zygote进程启动中承担的主要职责如下：

* 1、创建AppRuntime，执行其start方法，启动Zygote进程。。
* 2、创建JVM并为JVM注册JNI方法。
* 3、使用JNI调用ZygoteInit的main函数进入Zygote的Java FrameWork层。
* 4、使用registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等等AMS的请求去创建新的应用进程。
* 5、启动SystemServer进程。

### 6 SystemServer进程启动

#### 6.1 SystemServer概述

SystemServer进程主要是用于创建系统服务的，例如AMS、WMS、PMS。由上文中可知，SystemServer是由ZygoteInit.java创建的

```java
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {

    ...

    try {
        parsedArgs = new ZygoteConnection.Arguments(args);
        ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
        ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

        boolean profileSystemServer = SystemProperties.getBoolean(
                "dalvik.vm.profilesystemserver", false);
        if (profileSystemServer) {
            parsedArgs.runtimeFlags |= Zygote.PROFILE_SYSTEM_SERVER;
        }

        /* Request to fork the system server process */
        // 1
        pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.runtimeFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    // 2 当前运行在 SystemServer进程中
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }
        // 3 关闭Zygote 进程创建的Socket
        zygoteServer.closeServerSocket();
        // 4
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

1. 在注释1处，调用了Zygote的forkSystemServer\(\)方法创建了SystemServer进程，并返回了当前进程的pid。
2. 在注释2处，如果pid==0则说明Zygote进程创建SystemServer进程成功，当前运行在SystemServer进程中。
3. 在注释3处，由于SystemServer进程fork了Zygote进程的地址空间，所以会得到Zygote进程创建的Socket，这个Socket对于SystemServer进程是无用的，因此，在此处关闭了该Socket。
4. 在注释4处，调用了handleSystemServerprocess\(\)方法来启动SystemServer进程。handleSystemServerProcess\(\)方法如下所示： 

在注释1处，调用了Zygote的forkSystemServer\(\)方法创建了SystemServer进程，并返回了当前进程的pid。在注释2处，如果pid==0则说明Zygote进程创建SystemServer进程成功，当前运行在SystemServer进程中。接着，在注释3处，由于SystemServer进程fork了Zygote进程的地址空间，所以会得到Zygote进程创建的Socket，这个Socket对于SystemServer进程是无用的，因此，在此处关闭了该Socket。最后，在注释4处，调用了handleSystemServerprocess\(\)方法来启动SystemServer进程。handleSystemServerProcess\(\)方法如下所示：

```java
/**
 * Finish remaining work for the newly forked system server process.
 */
private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {

    ...

    if (parsedArgs.invokeWith != null) {

        ...
    } else {
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            // 1 创建了PathClassLoader
            cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);

            Thread.currentThread().setContextClassLoader(cl);
        }

        /*
         * Pass the remaining arguments to SystemServer.
         */
        // 2 ZygoteInit的init方法
        return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
}
```

注释1中创建了PathClassLoader，然后启动了ZygoteInit的zygoteInit方法

```java
public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
    if (RuntimeInit.DEBUG) {
        Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();

    RuntimeInit.commonInit();
    // 1 启动binder线程池
    ZygoteInit.nativeZygoteInit();
    // 2 进入SystemServer的main方法
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}
```

在zygoteInit\(\)方法中，首先在注释1处执行了nativeZygoteInit\(\)方法，这里看到方法前缀为native可知是一个本地函数，因此，我们先了解它对应的JNI文件，在AndroidRuntime.cpp类中可以查看到nativeZygoteInit\(\)方法对应的native函数，如下所示：

```text
/*
* JNI registration.
*/
int register_com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env)
{
    const JNINativeMethod methods[] = {
        { "nativeZygoteInit", "()V",
            (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
        methods, NELEM(methods));
}
```

这里使用了JNI动态注册的方式，将nativeZygoteInit\(\)方法和native函数com\_android\_internal\_os\_ZygoteInit\_nativeZygoteInit\(\)建立了映射关系，我们看到这个native方法的代码：

```text
static AndroidRuntime* gCurRuntime = NULL;

static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

可以看到，gCurRuntime是AndroidRuntime类型的指针，具体指向的是其子类AppRuntime，它在app\_main.cpp中定义，代码如下所示：

```text
class AppRuntime : public AndroidRuntime
{
    ...

    virtual void onZygoteInit()
    {
        // 1 创建了一个ProcessState实例
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        // 2 启动了一个Binder线程池
        proc->startThreadPool();
    }

    ...
}
```

1. 在注释1处，创建了一个ProcessState实例， 在Android中ProcessState是客户端和服务端公共的部分，作为Binder通信的基础，ProcessState是一个singleton类，每个进程只有一个对象，这个对象负责打开Binder驱动，建立线程池，让其进程里面的所有线程都能通过Binder通信。
2. 在注释2处，调用了ProcessState实例的startThreadPool\(\)函数启动了一个Binder线程池，其实里面最终会调用到IPCThreadState实例的joinThreadPool\(\)函数进程Binder线程池相关的处理。现在，我们再回到zygoteInit\(\)方法的注释2处，这里调用了RuntimeInit的applicationInit\(\)方法
3. **SystemServer的main方法**

```text
/**
* The main entry point from zygote.
*/
public static void main(String[] args) {
    new SystemServer().run();
}
```

他调用了SystemServer 的run方法

```java
rivate void run() {
    try {
        ...

        // 1 创建looper
        Looper.prepareMainLooper();
        ...

        // Initialize native services.
        // 2 加载动态库libandroid_servers.so
        System.loadLibrary("android_servers");

        // Check whether we failed to shut down last time we tried.
        // This call may not return.
        performPendingShutdown();

         // 创建系统的cotext
        createSystemContext();

        // Create the system service manager.
        // 3 创建SystemServiceManager
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        mSystemServiceManager.setStartInfo(mRuntimeRestart,
                mRuntimeStartElapsedTime, mRuntimeStartUptime);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        // Prepare the thread pool for init tasks that can be parallelized
        SystemServerInitThreadPool.get();
    } finally {
        traceEnd();  // InitBeforeStartServices
    }

    // Start services.
    try {
        traceBeginAndSlog("StartServices");
        // 4 启动引导服务
        startBootstrapServices();
        // 5 启动核心服务
        startCoreServices();
        // 6 启动其他服务
        startOtherServices();
        SystemServerInitThreadPool.shutdown();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        traceEnd();
    }

    ...

    // Loop forever.
    // 7 
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

可以看到，上面把系统服务分成了三种类型：引导服务、核心服务、其它服务。这些系统服务共有100多个，其中对于我们来说比较关键的有：

* 引导服务：ActivityManagerService，负责四大组件的启动、切换、调度。
* 引导服务：PackageManagerService，负责对APK进行安装、解析、删除、卸载等操作。
* 引导服务：PowerManagerService，负责计算系统中与Power相关的计算，然后决定系统该如何反应。
* 核心服务：BatteryService，管理电池相关的服务。
* 其它服务：WindowManagerService，窗口管理服务。
* 其它服务：InputManagerService，管理输入事件。

很多系统服务的启动逻辑都是类似的，这里我以启动ActivityManagerService服务来进行举例，代码如下所示：

```text
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
```

SystemServiceManager的startService\(\)方法启动了ActivityManagerService，该启动方法如下所示：

```java
@SuppressWarnings("unchecked")
public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();

        ...

        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            // 1
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {

        ...

        // 2
        startService(service);
        return service;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
}
```

在注释1处使用反射创建了ActivityManagerService实例，并在注释2处调用了另一个startService\(\)重载方法，如下所示：

```java
public void startService(@NonNull final SystemService service) {
    // Register it.
    // 1
    mServices.add(service);
    // Start it.
    long time = SystemClock.elapsedRealtime();
    try {
        // 2
        service.onStart();
    } catch (RuntimeException ex) {
        throw new RuntimeException("Failed to start service " + service.getClass().getName()
                + ": onStart threw an exception", ex);
    }
    warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
}
```

在注释1处，首先会将ActivityManagerService添加在mServices中，它是一个存储SystemService类型的ArrayList，这样就完成了ActivityManagerService的注册。在注释2处，调用了ActivityManagerService的onStart\(\)方法完成了启动ActivityManagerService服务。

除了使用SystemServiceManager的startService\(\)方法来启动系统服务外，也可以直接调用服务的main\(\)方法来启动系统服务，如PackageManagerService：

```text
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
```

这里直接调用了PackageManagerService的main\(\)方法：

```text
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();

    // 1
    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    m.enableSystemUserPackages();
    // 2
    ServiceManager.addService("package", m);
    // 3
    final PackageManagerNative pmn = m.new PackageManagerNative();
    ServiceManager.addService("package_native", pmn);
    return m;
}
```

在注释1处，直接新建了一个PackageManagerService实例，并在注释2处将PackageManagerService注册到服务大管家ServiceManager中，ServiceManager用于管理系统中的各种Service，用于系统C/S架构中的Binder进程间通信，即如果Client端需要使用某个Servcie，首先应该到ServiceManager查询Service的相关信息，然后使用这些信息和该Service所在的Server进程建立通信通道，这样Client端就可以服务端进程的Service进行通信了。

#### 6.2 总结

SystemS ervice的启动流程分析至此已经完结，经过以上的分析可知，SystemService进程被创建后，主要的处理如下：

* 1、启动Binder线程池，这样就可以与其他进程进行Binder跨进程通信。
* 2、创建SystemServiceManager，它用来对系统服务进行创建、启动和生命周期管理。
* 3、启动各种系统服务：引导服务、核心服务、其他服务，共100多种。应用开发主要关注引导服务ActivityManagerService、PackageManagerService和其他服务WindowManagerService、InputManagerService即可。

### 7 Launcher进程启动

#### 7.1 Launcher概述

Android系统启动的最后一步就是启动了一个Launcher应用程序来显示系统中已经安装的应用程序。**Launcher在启动的过程中会请求请求PMS返回系统中已安装的应用程序的信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上，从而使得用户可以点击这些快捷图片来启动相应的应用程序。**

Launcher作为Android系统的桌面，它的作用有两点：

* 1、作为Android系统的启动器，用于启动应用程序。
* 2、作为Android系统的桌面，用于显示和管理应用程序的快捷图标或者其它桌面组件。

#### 7.2 Launcher启动过程分析

SystemServer进程在启动的过程中会启动PMS，PMS启动后会将系统中的应用程序安装完成，先前已经启动的AMS会将Launcher启动起来。在SystemServer的startOtherServices\(\)方法中，调用了AMS的systemReady\(\)方法，此即为Launcher的入口，如下所示：

```text
private void startOtherServices() {
    ...

    mActivityManagerService.systemReady(() -> {
        Slog.i(TAG, "Making services ready");
        traceBeginAndSlog("StartActivityManagerReadyPhase");
        mSystemServiceManager.startBootPhase(
                SystemService.PHASE_ACTIVITY_MANAGER_READY);

        ...
        }
    ...
}
```

在Android 8.0及以上的部分源码中，都引入了Java Lambda表达式，可见其重要性的上升。下面继续分析AMS的systemReady\(\)方法：

```text
public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
    ...

    synchronized (this) {
        ...
        mStackSupervisor.resumeFocusedStackTopActivityLocked();
        mUserController.sendUserSwitchBroadcasts(-1, currentUserId);

        ...
    }
}
```

在systemReady\(\)方法中继续调用了ActivityStackSupervisor的resumeFocusedStackTopActivityLocked\(\)方法，如下所示：

```text
boolean resumeFocusedStackTopActivityLocked() {
    return resumeFocusedStackTopActivityLocked(null, null, null);
}

boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    ...

    if (targetStack != null && isFocusedStack(targetStack)) {
        // 1
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    ...

    return false;
}
```

最终，调用了注释1处ActivityStack（描述Acitivity堆栈）的resumeTopActivityUncheckedLocked\(\)方法，如下所示：

```text
@GuardedBy("mService")
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mStackSupervisor.inResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        mStackSupervisor.inResumeTopActivity = true;
        // 1
        result = resumeTopActivityInnerLocked(prev, options);

        // When resuming the top activity, it may be necessary to pause the top activity (for
        // example, returning to the lock screen. We suppress the normal pause logic in
        // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
        // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
        // to ensure any necessary pause logic occurs. In the case where the Activity will be
        // shown regardless of the lock screen, the call to
        // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            checkReadyForSleep();
        }
    } finally {
        mStackSupervisor.inResumeTopActivity = false;
    }

     return result;
}
```

在注释1处调用了resumeTopActivityInnerLocked\(\)方法，如下所示：

```text
@GuardedBy("mService")
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...

     if (!hasRunningActivity) {
        // There are no activities left in the stack, let's look somewhere else.
        return resumeTopActivityInNextFocusableStack(prev, options, "noMoreActivities");
    }

    ...
}
```

resumeTopActivityInnerLocked\(\)方法非常长，大概有好几百行代码，但是对于主要流程来说最关键的就是在其中调用了resumeTopActivityInNextFocusableStack\(\)方法，如下所示：

```text
private boolean resumeTopActivityInNextFocusableStack(ActivityRecord prev,
        ActivityOptions options, String reason) {
    if (adjustFocusToNextFocusableStack(reason)) {
        // Try to move focus to the next visible stack with a running activity if this
        // stack is not covering the entire screen or is on a secondary display (with no home
        // stack).
        return mStackSupervisor.resumeFocusedStackTopActivityLocked(
                mStackSupervisor.getFocusedStack(), prev, null);
    }

    // Let's just start up the Launcher...
    ActivityOptions.abort(options);
    if (DEBUG_STATES) Slog.d(TAG_STATES,
            "resumeTopActivityInNextFocusableStack: " + reason + ", go home");
    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
    // Only resume home if on home display
    // 1
    return isOnHomeDisplay() &&
            mStackSupervisor.resumeHomeStackTask(prev, reason);
}
```

在注释1处，调用了ActivityStackSupervisor的resumeHomeStackTask\(\)方法，如下所示：

```text
boolean resumeHomeStackTask(ActivityRecord prev, String reason) {
    ...

    // Only resume home activity if isn't finishing.
    if (r != null && !r.finishing) {
        moveFocusableActivityStackToFrontLocked(r, myReason);
        return resumeFocusedStackTopActivityLocked(mHomeStack, prev, null);
    }
    // 1
    return mService.startHomeActivityLocked(mCurrentUser, myReason);
}
```

注释1处，调用了AMS的startHomeActivityLocked\(\)方法，如下所示：

```text
boolean startHomeActivityLocked(int userId, String reason) {
    // 1
    if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
            && mTopAction == null) {
        // We are running in factory test mode, but unable to find
        // the factory test app, so just sit around displaying the
        // error message and don't try to start anything.
        return false;
    }

    // 2
    Intent intent = getHomeIntent();
    ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
    if (aInfo != null) {
        intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        // Don't do this if the home app is currently being
        // instrumented.
        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
        ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                aInfo.applicationInfo.uid, true);

        // 3
        if (app == null || app.instr == null) {
            intent.setFlags(intent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
            final int resolvedUserId = UserHandle.getUserId(aInfo.applicationInfo.uid);
            // For ANR debugging to verify if the user activity is the one that actually
            // launched.
            final String myReason = reason + ":" + userId + ":" + resolvedUserId;

            // 4
            mActivityStartController.startHomeActivity(intent, aInfo, myReason);
        }
    } else {
        Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
    }

    return true;
}
```

首先，会在注释1处判断工厂模式和mTopAction的值，这里的工厂模式mFactoryTest代表的了系统的运行模式，它分为三种：

* 1、非工厂模式
* 2、低级工厂模式
* 3、高级工厂模式

而mTopAction是来描述第一个被启动Activity组件的Action，默认值为Intent.ACTION\_MAIN。所以，此时可知当mFactoryTest为低级工厂模式并且mTopAction为空时，则返回false。接着，在注释2处，调用了getHomeintent\(\)方法，如下所示：

```text
Intent getHomeIntent() {
    // 1
    Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
    intent.setComponent(mTopComponent);
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        // 2
        intent.addCategory(Intent.CATEGORY_HOME);
    }
    return intent;
}
```

在getHomeIntent\(\)方法的注释1处，根据mTopAction和mTopData创建了Intent。注释2处，会判断如果系统运行模式不是低级工厂模式，则会将Category设置为Intent.CATEGORY\_HOME，最后返回该Intent。

我们再回到AMS的startHomeActivityLocked\(\)方法的注释3处，这里会判断符合上述Intent的应用程序是否已经启动，如果没有启动，则会在注释4处调用ActivityStartController的startHomeActivity\(\)方法启动该应用程序，即Launcher。下面我们继续看看startHomeActivity\(\)方法，如下所示：

```text
void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason) {
    // 1
    mSupervisor.moveHomeStackTaskToTop(reason);

    // 2
    mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
            .setOutActivity(tmpOutRecord)
            .setCallingUid(0)
            .setActivityInfo(aInfo)
            .execute();
    mLastHomeActivityStartRecord = tmpOutRecord[0];
    if (mSupervisor.inResumeTopActivity) {
        // If we are in resume section already, home activity will be initialized, but not
        // resumed (to avoid recursive resume) and will stay that way until something pokes it
        // again. We need to schedule another resume.
        mSupervisor.scheduleResumeTopActivities();
    }
}
```

注释1处，会将Launcher放入HomeStack中，它是ActivityStackSupervisor中用于存储Launcher的变量。然后，在注释2处调用了obtainStarter\(\)方法，如下所示：

```text
**
 * @return A starter to configure and execute starting an activity. It is valid until after
 *         {@link ActivityStarter#execute} is invoked. At that point, the starter should be
 *         considered invalid and no longer modified or used.
 */
ActivityStarter obtainStarter(Intent intent, String reason) {
    return mFactory.obtain().setIntent(intent).setReason(reason);
}
```

可知这里最终会返回一个配置好指定intent和reason和ActivityStarter，当它调用execute\(\)方法时，则会启动Launcher，如下所示：

```text
int execute() {
    try {
        // TODO(b/64750076): Look into passing request directly to these methods to allow
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        }
    } finally {
        onExecutionComplete();
    }
}
```

可以看到，这里调用了startActivity\(\)方法来启动Launcher，最终会进入Launcher的onCreate\(\)方法，Launcher启动完成。

#### 7.3 Launcher中应用图标的显示过程

应用程序图标是进入应用程序的入口，接下来我们了解一下Launcher是如何显示应用程序图标的。首先从Launcher的onCreate\(\)方法开始，如下所示：

```text
@Override
protected void onCreate(Bundle savedInstanceState) {
    ...

    // 1
    LauncherAppState app = LauncherAppState.getInstance(this);
    mOldConfig = new Configuration(getResources().getConfiguration());

    // 2
    mModel = app.setLauncher(this);
    initDeviceProfile(app.getInvariantDeviceProfile());

    ...

    // We only load the page synchronously if the user rotates (or triggers a
    // configuration change) while launcher is in the foreground
    int currentScreen = PagedView.INVALID_RESTORE_PAGE;
    if (savedInstanceState != null) {
        currentScreen = savedInstanceState.getInt(RUNTIME_STATE_CURRENT_SCREEN, currentScreen);
    }

    // 3
    if (!mModel.startLoader(currentScreen)) {
        if (!internalStateHandled) {
            // If we are not binding synchronously, show a fade in animation when
            // the first page bind completes.
            mDragLayer.getAlphaProperty(ALPHA_INDEX_LAUNCHER_LOAD).setValue(0);
        }
    } else {
        // Pages bound synchronously.
        mWorkspace.setCurrentPage(currentScreen);

        setWorkspaceLoading(true);
    }

}
```

首先，在注释1处得到LauncherAppState的实例，在注释2处，调用了它的setLauncher\(\)方法将Launcher对象传进去，setLauncher\(\)方法如下所示：

```text
 LauncherModel setLauncher(Launcher launcher) {
    getLocalProvider(mContext).setLauncherProviderChangeListener(launcher);
    mModel.initialize(launcher);
    return mModel;
}
```

在setLauncher\(\)方法里面继续调用了LauncherModel的initialize\(\)方法，如下所示：

```text
/**
* Set this as the current Launcher activity object for the loader.
*/
public void initialize(Callbacks callbacks) {
    synchronized (mLock) {
        Preconditions.assertUIThread();
        mCallbacks = new WeakReference<>(callbacks);
    }
}
```

从此处我们可以得知Launcher被封装成了一个弱引用对象mCallbacks。我们再回到Launcher的onCreate\(\)方法的注释3处的LauncherModel的startLoader\(\)方法，如下所示：

```text
// 1
@Thunk static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");
static {
    sWorkerThread.start();
}

// 2
@Thunk static final Handler sWorker = new Handler(sWorkerThread.getLooper());

public boolean startLoader(int synchronousBindPage) {
    // Enable queue before starting loader. It will get disabled in Launcher#finishBindingItems
    InstallShortcutReceiver.enableInstallQueue(InstallShortcutReceiver.FLAG_LOADER_RUNNING);
    synchronized (mLock) {
        // Don't bother to start the thread if we know it's not going to do anything
        if (mCallbacks != null && mCallbacks.get() != null) {
            final Callbacks oldCallbacks = mCallbacks.get();
            // Clear any pending bind-runnables from the synchronized load process.
            mUiExecutor.execute(oldCallbacks::clearPendingBinds);

            // If there is already one running, tell it to stop.
            stopLoader();

            // 3
            LoaderResults loaderResults = new LoaderResults(mApp, sBgDataModel,
                    mBgAllAppsList, synchronousBindPage, mCallbacks);
            if (mModelLoaded && !mIsLoaderTaskRunning) {
                // Divide the set of loaded items into those that we are binding synchronously,
                // and everything else that is to be bound normally (asynchronously).
                loaderResults.bindWorkspace();
                // For now, continue posting the binding of AllApps as there are other
                // issues that arise from that.
                loaderResults.bindAllApps();
                loaderResults.bindDeepShortcuts();
                loaderResults.bindWidgets();
                return true;
            } else {
                // 4
                startLoaderForResults(loaderResults);
            }
        }
    }
    return false;
}
```

在注释1处，新建了具有消息循环的线程HandlerThread对象。注释2处，新建了Handler，并传入了HandlerThread的Looper，此处Handler就是用于向HandlerThread发送消息。接着，在注释3处，创建了LoaderResults，在注释4处，调用了startLoaderForResults\(\)方法并将LoaderResults传入，如下所示：

```text
public void startLoaderForResults(LoaderResults results) {
    synchronized (mLock) {
        stopLoader();
        mLoaderTask = new LoaderTask(mApp, mBgAllAppsList, sBgDataModel, results);
        runOnWorkerThread(mLoaderTask);
    }
}
```

在startLoaderForResults\(\)方法中，调用了runOnWorkerThread\(\)，如下所示：

```text
/** Runs the specified runnable immediately if called from the worker thread, otherwise it is
 * posted on the worker thread handler. */
private static void runOnWorkerThread(Runnable r) {
    // 1
    if (sWorkerThread.getThreadId() == Process.myTid()) {
        // 2
        r.run();
    } else {
        // If we are not on the worker thread, then post to the worker handler
        // 3
        sWorker.post(r);
    }
}
```

首先，注释1处会先判断当前的执行线程是否是工作线程，如果是则直接调用注释2处Runnable的run\(\)方法，否则，调用sWorker这个Handler对象的post\(\)方法将LoaderTask作为消息发送给HandlerThread。接下来，我们看看LoaderTask，它实现了Runnable接口，当其所描述的消息被处理时，则会调用它的run\(\)方法，如下所示：

```text
/**
* Runnable for the thread that loads the contents of the launcher:
*   - workspace icons
*   - widgets
*   - all apps icons
*   - deep shortcuts within apps
*/
public class LoaderTask implements Runnable {

    ...

     synchronized (this) {
        // Skip fast if we are already stopped.
        if (mStopped) {
            return;
        }
    }

    TraceHelper.beginSection(TAG);
    try (LauncherModel.LoaderTransaction transaction = mApp.getModel().beginLoader(this)) {
        TraceHelper.partitionSection(TAG, "step 1.1: loading workspace");
        // 1
        loadWorkspace();

        verifyNotStopped();
        TraceHelper.partitionSection(TAG, "step 1.2: bind workspace workspace");
        // 2
        mResults.bindWorkspace();

        // Notify the installer packages of packages with active installs on the first screen.
        TraceHelper.partitionSection(TAG, "step 1.3: send first screen broadcast");
        sendFirstScreenActiveInstallsBroadcast();

        // Take a break
        TraceHelper.partitionSection(TAG, "step 1 completed, wait for idle");
        waitForIdle();
        verifyNotStopped();

        // second step
        TraceHelper.partitionSection(TAG, "step 2.1: loading all apps");
        // 3
        loadAllApps();


        TraceHelper.partitionSection(TAG, "step 2.2: Binding all apps");
        verifyNotStopped();
        // 4
        mResults.bindAllApps();

        ...
     } catch (CancellationException e) {
        // Loader stopped, ignore
        TraceHelper.partitionSection(TAG, "Cancelled");
    }
    TraceHelper.endSection(TAG);
}
```

**Launcher是用工作区的形式来显示系统安装的应用程序快捷图标的，每一个工作区都是用来描述一个抽象桌面的，它由n个屏幕组成，每个屏幕又分为n个单元格，每个单元格用来显示一个应用程序的快捷图标。**

首先，在注释1、2处调用了loadWorkSpace\(\)和LoaderResults的bindWorkspace\(\)方法来加载和绑定工作区信息。注释3处调用了loadAllApps\(\)和LoaderResults的bindAllApps\(\)方法来加载系统已经安装的应用程序信息，bindAllApps\(\)方法如下所示：

```text
public void bindAllApps() {
    // shallow copy
    @SuppressWarnings("unchecked")
    final ArrayList<AppInfo> list = (ArrayList<AppInfo>) mBgAllAppsList.data.clone();

    Runnable r = new Runnable() {
        public void run() {
            // 1
            Callbacks callbacks = mCallbacks.get();
            if (callbacks != null) {
                // 2
                callbacks.bindAllApplications(list);
            }
        }
    };
    // 3
    mUiExecutor.execute(r);
}
```

首先，在注释1处会从mCallbacks这个Launcher的弱引用对象中取出Launcher对象，并在注释2处调用了它的bindAllApplication\(\)来绑定所有的应用程序信息，最后在注释3处使用mUiExecutor这个MainThreadExecutor执行器对象去执行这个创建好的Runnable对象。接下来，我们看看Launcher的bindAllApplications\(\)方法，如下所示：

```text
// Main container view for the all apps screen.
@Thunk AllAppsContainerView mAppsView;

/**
* Add the icons for all apps.
*
* Implementation of the method from LauncherModel.Callbacks.
*/
public void bindAllApplications(ArrayList<AppInfo> apps) {
    // 1
    mAppsView.getAppsStore().setApps(apps);

    if (mLauncherCallbacks != null) {
        mLauncherCallbacks.bindAllApplications(apps);
    }
}
```

在注释1处，调用了AllAppsContainerView的getAppsStore\(\)方法得到了一个AllAppsStore对象，AllAppsContainerView是所有App屏幕的主容器视图，AllAppsStore是一个负责维护所有app信息集合的通用工具类。下面，我们看看AllAppsStore对象的setApps\(\)方法：

```text
/**
 * Sets the current set of apps.
 */
public void setApps(List<AppInfo> apps) {
    mComponentToAppMap.clear();
    addOrUpdateApps(apps);
}
```

这里继续调用了addOrUpdateApps\(\)方法：

```text
 private final HashMap<ComponentKey, AppInfo> mComponentToAppMap = new HashMap<>();

/**
* Adds or updates existing apps in the list
*/
public void addOrUpdateApps(List<AppInfo> apps) {
    for (AppInfo app : apps) {
        mComponentToAppMap.put(app.toComponentKey(), app);
    }
    notifyUpdate();
}
```

可以看到，最终将所有app信息保存在了AllAppsStore的HashMap容器中。

当AllAppsContainerView加载完XML布局时，会调用自身的onFinishInflate\(\)方法，如下所示：

```text
@Override
protected void onFinishInflate() {
    super.onFinishInflate();

    // This is a focus listener that proxies focus from a view into the list view.  This is to
    // work around the search box from getting first focus and showing the cursor.
    setOnFocusChangeListener((v, hasFocus) -> {
        if (hasFocus && getActiveRecyclerView() != null) {
            getActiveRecyclerView().requestFocus();
        }
    });

    mHeader = findViewById(R.id.all_apps_header);

    // 1
    rebindAdapters(mUsingTabs, true /* force */);

    mSearchContainer = findViewById(R.id.search_container_all_apps);
    mSearchUiManager = (SearchUiManager) mSearchContainer;
    mSearchUiManager.initialize(this);
}
```

在注释1处，进行了适配器数据的绑定，我们继续查看rebindAdapters\(\)方法：

```text
private void rebindAdapters(boolean showTabs) {
    rebindAdapters(showTabs, false /* force */);
}

private void rebindAdapters(boolean showTabs, boolean force) {
    ...

    if (mUsingTabs) {
        // 1
        mAH[AdapterHolder.MAIN].setup(mViewPager.getChildAt(0), mPersonalMatcher);
        mAH[AdapterHolder.WORK].setup(mViewPager.getChildAt(1), mWorkMatcher);
        onTabChanged(mViewPager.getNextPage());
    } else {
        // 2
        mAH[AdapterHolder.MAIN].setup(findViewById(R.id.apps_list_view), null);
        mAH[AdapterHolder.WORK].recyclerView = null;
    }
    setupHeader();

    ...
}
```

可以看到，不管是否正在使用标签，最终都会调用到AdapterHolder的setup\(\)方法，它时AllAppsContainerView的内部类，如下所示：

```text
void setup(@NonNull View rv, @Nullable ItemInfoMatcher matcher) {
        appsList.updateItemFilter(matcher);
        recyclerView = (AllAppsRecyclerView) rv;
        recyclerView.setEdgeEffectFactory(createEdgeEffectFactory());

        // 1
        recyclerView.setApps(appsList, mUsingTabs);
        recyclerView.setLayoutManager(layoutManager);

        // 2
        recyclerView.setAdapter(adapter);
        recyclerView.setHasFixedSize(true);
        // No animations will occur when changes occur to the items in this RecyclerView.
        recyclerView.setItemAnimator(null);
        FocusedItemDecorator focusedItemDecorator = new FocusedItemDecorator(recyclerView);
        recyclerView.addItemDecoration(focusedItemDecorator);
        adapter.setIconFocusListener(focusedItemDecorator.getFocusListener());
        applyVerticalFadingEdgeEnabled(verticalFadingEdge);
        applyPadding();
    }
```

注释1处，会将app信息列表appsList设置给AllAppsRecyclerView对象，在注释2处，为其设置了Adapter。最终，应用程序快捷图标列表就会显示到屏幕上了。

