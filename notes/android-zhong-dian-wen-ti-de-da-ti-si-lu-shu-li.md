# Android面试问题及答案整理

## 1. 最常见重点问题

### 1. Handler相关

#### 1.1 简单叙述下handler的机制

![](https://user-gold-cdn.xitu.io/2019/2/26/16927e6099e1d48c)

* **Message**：消息分为硬件产生的消息\(如按钮、触摸\)和软件生成的消息；
* **MessageQueue**：消息队列的主要功能向消息池投递消息\(`MessageQueue.enqueueMessage`\)和取走消息池的消息\(`MessageQueue.next`\)；
* **Handler**：消息辅助类，主要功能向消息池发送各种消息事件\(`Handler.sendMessage`\)和处理相应消息事件\(`Handler.handleMessage`\)；
* **Looper**：不断循环执行\(`Looper.loop`\)，按分发机制将消息分发给目标处理者。

#### 1.2 MessageQueue的小知识

**1.2.1 消息的创建**

延迟方法和非延迟的方法（sendMessageDelayed方法），最后调用的核心的方法是enqueueMessage（），这里主要做了两件事

* 首先设置了Message的when属性，也就是代表了这个消息的处理时间。
* 就需要遍历这个队列，也就是链表，找出when小于某个节点的when，找到后插入。

在Handler机制中有三种消息：

**同步消息**。也就是普通的消息。

**异步消息**。通过setAsynchronous\(true\)设置的消息。

**同步屏障消息**。通过postSyncBarrier方法添加的消息，特点是target为空，也就是没有对应的handler。

三者的区别是，同步消息和异步消息是通过上面的when属性去进行判断，而同步屏障消息是直接去找异步消息，然后结合when来进行判断是否需要返回消息。（使用场景如View的绘制方法scheduleTraversals（））；

所以整体上来说MessageQueue就是一个按照消息时间排列的一个链表结构。

**1.2.2 消息的取出**

核心的方法就是next\(\)方法，如上图所示，他也是Java和Native层的纽带，这里就是死循环取出消息，（next方法是死循环，目的就是为了保证一定要返回一条信息）

![](http://gityuan.com/images/handler/poll_once.png)

next的内部会调用nativePollOnce方法，他是一个阻塞方法，nextPollTimeoutMillis参数就是阻塞的时间，阻塞的情况有两种

* 有消息，但是当前时间小于消息执行时间，这就是创建时候传入的when的属性
* 消息队列为空

这里的阻塞的方法是通过epoll机制，**epoll机制是一种IO多路复用的机制，具体逻辑就是一个进程可以监视多个描述符，当某个描述符就绪（一般是读或写的操作就绪）的时候，通知对应的程序进行读写操作，否则就是在阻塞状态**。在Android中 会创建一个Linux管道（Pipe）来处理阻塞和唤醒，

* 当消息队列为空，管道的读端等待管道中有新内容可读，就会通过epoll机制进入阻塞状态。
* 当有消息要处理，就会通过管道的写端写入内容，唤醒主线程

### 1.2 Looper的小知识点

```java
    private static void prepare(boolean quitAllowed) {
    //每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //创建Looper对象，并保存到当前线程的TLS区域
    sThreadLocal.set(new Looper(quitAllowed));
}
```

1. 每个线程只允许有一个Looper，当Looper不为空的时候就会抛出异常
2. 创建Looper的对象会存储在ThreadLocal中

   补充：ThreadLocal 是提供线程内的局部变量，内部有一个静态类ThreadLocalMap（其原理和HashMap类似k-v和hash算法）具体的会在线程安全中体现。采用这样的机制可以在不同的线程，访问同一个ThreadLocal对象，但是能获取到的值却不一样。

**1.3.1 消息的分发loop**

```java
public static void loop() {
    for (;;) {# Android面试问题及答案整理

## 1. 最常见重点问题

### 1. Handler相关

#### 1.1 简单叙述下handler的机制

<img src="https://user-gold-cdn.xitu.io/2019/2/26/16927e6099e1d48c" style="zoom:80%;" />

- **Message**：消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
- **MessageQueue**：消息队列的主要功能向消息池投递消息(`MessageQueue.enqueueMessage`)和取走消息池的消息(`MessageQueue.next`)；
- **Handler**：消息辅助类，主要功能向消息池发送各种消息事件(`Handler.sendMessage`)和处理相应消息事件(`Handler.handleMessage`)；
- **Looper**：不断循环执行(`Looper.loop`)，按分发机制将消息分发给目标处理者。



#### 1.2 MessageQueue的小知识

##### 1.2.1 消息的创建

延迟方法和非延迟的方法（sendMessageDelayed方法），最后调用的核心的方法是enqueueMessage（），这里主要做了两件事

- 首先设置了Message的when属性，也就是代表了这个消息的处理时间。
- 就需要遍历这个队列，也就是链表，找出when小于某个节点的when，找到后插入。

在Handler机制中有三种消息：

**同步消息**。也就是普通的消息。

**异步消息**。通过setAsynchronous(true)设置的消息。

**同步屏障消息**。通过postSyncBarrier方法添加的消息，特点是target为空，也就是没有对应的handler。

三者的区别是，同步消息和异步消息是通过上面的when属性去进行判断，而同步屏障消息是直接去找异步消息，然后结合when来进行判断是否需要返回消息。（使用场景如View的绘制方法scheduleTraversals（））；

所以整体上来说MessageQueue就是一个按照消息时间排列的一个链表结构。

##### 1.2.2 消息的取出

核心的方法就是next()方法，如上图所示，他也是Java和Native层的纽带，这里就是死循环取出消息，（next方法是死循环，目的就是为了保证一定要返回一条信息）

<img src="http://gityuan.com/images/handler/poll_once.png" style="zoom:67%;" />

next的内部会调用nativePollOnce方法，他是一个阻塞方法，nextPollTimeoutMillis参数就是阻塞的时间，阻塞的情况有两种

- 有消息，但是当前时间小于消息执行时间，这就是创建时候传入的when的属性
- 消息队列为空

这里的阻塞的方法是通过epoll机制，**epoll机制是一种IO多路复用的机制，具体逻辑就是一个进程可以监视多个描述符，当某个描述符就绪（一般是读或写的操作就绪）的时候，通知对应的程序进行读写操作，否则就是在阻塞状态**。在Android中 会创建一个Linux管道（Pipe）来处理阻塞和唤醒，

-  当消息队列为空，管道的读端等待管道中有新内容可读，就会通过epoll机制进入阻塞状态。
-  当有消息要处理，就会通过管道的写端写入内容，唤醒主线程

### 1.2 Looper的小知识点

```java
    private static void prepare(boolean quitAllowed) {
    //每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //创建Looper对象，并保存到当前线程的TLS区域
    sThreadLocal.set(new Looper(quitAllowed));
}
```

1. 每个线程只允许有一个Looper，当Looper不为空的时候就会抛出异常
2. 创建Looper的对象会存储在ThreadLocal中

   补充：ThreadLocal 是提供线程内的局部变量，内部有一个静态类ThreadLocalMap（其原理和HashMap类似k-v和hash算法）具体的会在线程安全中体现。采用这样的机制可以在不同的线程，访问同一个ThreadLocal对象，但是能获取到的值却不一样。

**1.3.1 消息的分发loop**

```java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // might block

        try {
            msg.target.dispatchMessage(msg);
        } 

        msg.recycleUnchecked();
    }
}
```

在loop的方法中进行消息分发，核心就是`msg.target.dispatchMessage(msg);`这里的的target是在使用Hanlder发送消息的时候，会设置msg.target = this，所以target就是当初把消息加到消息队列的那个Handler。

**衍生问题：Handler 的 post\(Runnable\) 与 sendMessage 有什么区别**

差别在dispatchMessage的方法中

```java
public void dispatchMessage(@NonNull Message msg) {
  //优先判断是否是runnable对象
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```

1. post方法是给Message设置了一个callback
2. 在消息的处理过程中， 优先分发给msg.callback，如果前者为空则返给Handler.Callback.handleMessage，如果后者为空则直接返回不处理

所以两者的区别是交给谁处理上。

**1.3.2 recycleUnchecked方法**

看上面的代码需要注意的一点就是在分发消息（dispatchMessage）之后，还调用了一个recycleUnchecked方法。

recycleUnchecked这个方法释放了消息所有资源，然后将当前的空消息插入到sPool（Message的链表，默认长度为50）头部，一些博客中讲得消息复用池就是这个。

Message的复用，可以看下代码，就是取出复用池的头部节点

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

**衍生问题：Looper.loop方法是死循环，为什么不会卡死（ANR）**

1. 主线程本身需要及时处理各种响应和交互（如View的绘制，Activity的生命周期等问题）,所以需要一个死循环来实时处理问题
2. 看代码，Looper本身只做消息的分发，操作时间很短，真正产生ANR的原因往往是，接受到对应消息后的处理引起的
3. 当没有消息的时候，会阻塞在loop的queue.next\(\)中的nativePollOnce\(\)方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生。所以死循环也不会特别消耗CPU资源。

#### 1.4 常见问题补充

**1.4.1 IdleHandler是啥？有什么使用场景？**

上面提到过，nativePollOnce是阻塞方法，其实在阻塞之前会有检查是否存在IdleHandler，如果存在就执行他的queueIdle方法。

也就是说，当前如果没有消息需要处理的时候，空闲状态可以处理一些空闲任务。常见的可以是作为启动优化，使用不当的话，可能导致异常情况。

**1.4.2 HandlerThread是啥？有什么使用场景？**

```java
public class HandlerThread extends Thread {
@Override
public void run() {
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
}
```

HandlerThread就是一个封装了Looper的Thread类。目的就是为了我们能更方便的使用handler，通过加锁保证获取当前Looper对象成功后，通知其他线程。

**1.4.3 IntentService是啥？有什么使用场景？**

1. 继承自Service
2. 并且内部维护了一个HandlerThread，也就是有完整的Looper在运行
3. 还维护了一个子线程的ServiceHandler
4. 启动Service后，会通过Handler执行onHandleIntent方法。
5. 完成任务后，会自动执行stopSelf停止当前Service

总结：他是一个可以在子线程进行耗时任务，并且在任务执行后自动停止的Service。

**1.4.4 BlockCanary使用过吗？说说原理**

BlockCanary是一个用来检测应用卡顿耗时的三方库。

View的绘制也是通过Handler来执行的，所以如果能知道每次Handler处理消息的时间，就能知道每次绘制的耗时了？那Handler消息的处理时间怎么获取呢？

在Looper.loop\(\)中loop方法内有一个Printer类，在dispatchMessage处理消息的前后分别打印了两次日志，把这个loop替换成我们自己的就行了。

**1.4.5 Handler内存泄露的原因是什么？**

`Handler`导致内存泄漏一般发生在发送延迟消息的时候，当`Activity`关闭之后，延迟消息还没发出，那么主线程中的`MessageQueue`就会持有这个消息的引用，而这个消息是持有`Handler`的引用，而`handler`作为匿名内部类持有了`Activity`的引用，所以就有了以下的一条引用链。

所以根本原因就是主线程永远不会被回收，导致了Activity一直不能回收，而Handler是引用关系中的一环。

### 2. Activity启动相关

![](https://user-gold-cdn.xitu.io/2020/7/12/17343b491419f3e9)

几个重点类的解释：

* **Instrumentation**: 监控应用与系统相关的交互行为。
* **ATMS**:组件管理调度中心，什么都不干，但是什么都管。
* **ActivityStarter**:Activity 启动的控制器，处理 Intent 与 Flag 对 Activity 启动的影响
* **ActivityStackSupervisior**:这个类的作用你从它的名字就可以看出来，它用来管理任务栈
* **ActivityStack**:用来管理任务栈里的 Activity。

主要还是上面的流程，这里做个答题总结：

1. 首先需要明确，进程间的关系，从起始App进程到SystemServer进程，最后到目标App进程
2. 然后逐个分析，首先在起始App进程中，Activity到Instrumentation
3. 进入到SystemServer进程中，这里强调一下Android 10 中引入的ATMS，然后再ATMS的startActivity方法，通过ActivityStarter最后执行到ActivityStack中**resumeTopActivityInnerLocked**的关键方法，这个方法里面主要做了几件事
   1. 暂停了上一个Activity
   2. 判断Activity是否启动（next.attachedToProcess（））
   3. 如果启动的Activity，直接安排Resume，如果没有启动就是进入启动Activity的流程，步骤4
4. 接着事件会来到**ActivityStackSupervisor**中，startSpecificActivityLocked，这个方法开面会进行判断，目标Activity的进程是否存在，不存在的话，需要通过Zygote去Fork一个进程。
5. 继续调用方法realStartActivityLocked，通过ClientTransaction将对应的事务发送到ApplicationThread，这时候就来到了目标App进程
6. 这里会通过的mH（handler的机制）将消息发送到主线程，然后ActivityThread通过对应的事件启动对应的Activity生命周期onCreate，onResume（handleLaunchActivity--》performLaunchActivity，performResumeActivity）等。

**关于Activity生命周期这里需要补充**

**performLaunchActivity**主要完成以下事情：

1. 从ActivityClientRecord获取待启动的Activity的组件信息
2. 创建一个ContextImpl对象
3. 通过mInstrumentation.newActivity方法使用类加载器创建activity实例
4. 如果没有Application的话，这里会通过LoadedApk的makeApplication方法创建一个Application对象，内部也是通过mInstrumentation使用类加载器，创建后就调用了instrumentation.callApplicationOnCreate方法，也就是Application的onCreate方法。
5. 并通过activity.attach方法对重要数据初始化，为Activity创建上下文。
6. 调用Activity的onCreate方法，是通过 mInstrumentation.callActivityOnCreate方法完成。

接下来看**handleResumeActivity**，他主要做了两件事：

1. 调用生命周期：通过performResumeActivity方法，内部调用生命周期onStart、onResume方法
2. 设置视图可见：通过activity.makeVisible方法，添加[window](https://blog.csdn.net/hfy8971613/article/details/103241153)、设置可见。（所以视图的真正可见是在onResume方法之后）

另外一点，Activity视图渲染到Window后，会设置window焦点变化，先走到DecorView的onWindowFocusChanged方法，最后是到Activity的onWindowFocusChanged方法，表示首帧绘制完成，此时Activity可交互。

至此Activity的启动全部流程结束。

**Zygote是如何fork进程的？**

补充一下创建进程的流程：

1. 通知Zygote需要创建一个进程
2. Zygote通过fork创建了一个进程
3. 在新建的进程中创建Binder线程池（此进程就支持了Binder IPC）
4. **最终是通过反射获取到了ActivityThread类并执行了main方法**
5. 在ActivityThread的main方法中，
   1. 创建ActivitThread的实例
   2. 创建消息循环机制
6. 调用attach方法，最后到AMS中
   1. 创建并绑定Application
   2. ApplicationThread关联到AMS
7. 通过ATMS启动根Activity

### 3. View绘制相关

先看下大概结构：

![](https://upload-images.jianshu.io/upload_images/2397836-f1f6a200704884a2.png)

#### 3.1 **DecorView** 被加载到 **Window** 中

Activity创建的时候，会先调用performLaunchActivity，然后是Activity的onCreate方法，这时候会创建**DecorView**和**Activity**。在handleResumeActivity方法中会通过WindowManager的addView方法将DecorView添加到Window中。

WindowManager最终会调用WindowManagerGlobal.addView，在这个方法里面会创建ViewRootImpl，ViewRootImpl是连接WindowManager和DecorView的纽带，View的操作都是由ViewRootImpl控制的。

ViewRootImpl的构造方法内创建了WindowSession\(Binder\)，通过它与WindowManagerService进行通信。

**window的机制**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/768cd62019764d129e24d432792e3638~tplv-k3u1fbpfcp-zoom-1.image)

我们知道WMS是window的最终管理者，在WMS中为每一个应用持有一个session，关于session前面我们讲过，每个应用都是全局单例，负责和WMS通信的binder对象。WMS为每个window都建立了一个windowStatus对象，同一个应用的Window使用同一个session进行通信，每一个windowStatus对应一个viewRootImpl，WMS通过viewRootImpl来控制view。

**Window中的token是什么，有什么用？**

addview方法中，还有个细节我们没说到，就是adjustLayoutParamsForSubWindow方法。

```java
//Window.java
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
    if (wp.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
            wp.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
        //子Window
        if (wp.token == null) {
            View decor = peekDecorView();
            if (decor != null) {
                wp.token = decor.getWindowToken();
            }
        }
    } else if (wp.type >= WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW &&
            wp.type <= WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
        //系统Window
    } else {
        //应用Window
        if (wp.token == null) {
            wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
        }

    }
}
```

上述代码分别代表了三个Window的类型：

* 子Window。需要从decorview中拿到token。
* 系统Window。不需要token。
* 应用Window。直接拿mAppToken，mAppToken是在setWindowManager方法中传进来的，也就是新建Window的时候就带进来了token。

然后在WMS中的addWindow方法会验证这个token，下次说到WMS的时候再看看。

所以这个token就是用来验证是否能够添加Window，可以理解为权限验证，其实也就是为了防止开发者乱用context创建window。

拥有token的context（比如Activity）就可以操作Window。没有token的上下文（比如Application）就不允许直接添加Window到屏幕（除了系统Window）。

#### 3.2 **View的开始绘制**

Activity启动的状况下的话，视图**ViewRootImpl.requestLayout** 就触发了第一次 View 的绘制。

View绘制会从根视图 ViewRootImpl 的 **performTraversals**\(\)方法开始，从上到下遍历整个ViewTree视图树，每个 View 控件负责绘制自己，而 ViewGroup 还需要负责通知自己的子View 进行绘制操作。

performTraversals内部就是就是熟悉的 performMeasure -&gt; performLayout -&gt; performDraw 三个流程了。

1. **理解MeasureSpec**

一个MeasureSpec（View的静态内部类）封装了从父容器传递给子容器的布局要求,这个MeasureSpec 封装的是父容器传递给子容器的布局要求，而不是父容器对子容器的布局要求，MeasureSpec是一个大小跟模式的组合值,MeasureSpec中的值是一个整型（32位）将size和mode打包成一个Int型，其中高两位是mode，后面30位存的是size，是为了减少对象的分配开支。

MeasureSpec一共有三种模式

* **UNSPECIFIED** : 父容器对于子容器没有任何限制,子容器想要多大就多大
* **EXACTLY**: 父容器已经为子容器设置了尺寸,子容器应当服从这些边界,不论子容器想要多大的空间。
* **AT\_MOST**：子容器可以是声明大小内的任意大小（有点像match\_parent）

**2. View 绘制流程之 performMeasure**

* 首先，在 ViewGroup 中的 measureChildren\(\)方法中会遍历测量所有子View，当View的状态为Gone时不测量
* 然后，测量某个指定的 View 时，根据父容器的 MeasureSpec 和子 View的LayoutParams来计算子View的MeasureSpec
* 最后，将计算出的 MeasureSpec 传入 View 的 measure 方法，这里ViewGroup没有定义具体的测量过程，其测量过程需要子类去实现

**getSuggestMinimumWidth** 分析

如果 View 没有设置背景，那么返回 android:minWidth 这个属性所指定的值，这个值可以为 0;如果 View 设置了背景，则返回 android:minWidth 和背景的最小宽度这两者中的最大值。

自定义 **View** 时手动处理 **wrap\_content** 时的情形

直接继承 View 的控件需要重写 onMeasure 方法并设置 wrap\_content 时的自身大小，否则在布局中使用 wrap\_content 就相当于使用 match\_parent。

在 **Activity** 中获取某个 **View** 的宽高

由于 View 的 measure 过程和 Activity 的生命周期方法不是同步执行的，如果 View还没有测量完毕，那么获得的宽/高就是 0。所以在 onCreate、onStart、onResume中均无法正确得到某个 View 的宽高信息。解决方式如下:

* Activity/View\#onWindowFocusChanged:此时 View 已经初始化完毕，当Activity的窗口获得和失去焦点时均会调用一次。
* view.post\(runnable\): 通过 post 可以将一个 runnable 投递到消息队列的尾部，等Looper调用runnable的时候View已经初始化好了。
* ViewTreeObserver\#addOnGlobalLayoutListener:当 View 树的状态发生改变或者ViewTree发生可变性变化的时候，会回调这个方法

**3. View 的绘制流程之 performLayout**

首先，会通过 **setFrame** 方法来设定 View 的四个顶点的位置，即 View 在父容器中的位置。然后，会执行到 onLayout 空方法，子类如果是 ViewGroup 类型，则重写这个方法，实现 ViewGroup 中所有 View 控件布局流程。

**4. View绘制流程值performDraw**

ViewRootImpl的performDraw会调用到draw，这里会把需要重新绘制的区域标记为dirty，然后会调用view的draw方法。

View中的`draw(canvas,parent,drawingTime)` - `draw(canvas)` - `onDraw` - `dispachDraw` - `drawChild`这条递归路径（下文简称**Draw路径**），调用了`Canvas.drawXxx()`方法，在软件渲染时用于实际绘制；在硬件加速时，用于构建DisplayList。

绘制基本上可以分为六个步骤:

* 首先绘制 View 的背景;
* 如果需要的话，保持 canvas 的图层，为 fading 做准备;
* 绘制View
* 绘制View的子View
* 如果需要的话，绘制View的fading边缘并恢复图层
* 绘制View的装饰\(例如滚动条等等\)

**Android中surface机制**

上面讲到的了widow的机制，实际上需要渲染到屏幕上还需要一个surface的机制。draw的过程就是讲View绘制到surface上。

`Surface`是`Window(ViewRootImpl)`的UI载体，也就是说一个`ViewRootImpl`就对应一个`Surface`。WMS在创建windowStatus之前，调用了requestLayout。可以看下面Viwe的绘制。

**真正的Surface创建是由`SurfaceControl`完成的，应用程序`ViewRootImpl`的`Surface`只是一个指针，指向这个`Surface`**。

surface创建完成后，会通过surfaceFlinger进行组装，最后展示到屏幕上。

**我们看的电影在24帧左右，为什么不觉得卡顿？**

电影有一个动态模糊的效果（将位移信息，像素变化填充到一帧的中，即时在低帧率的情况下，大脑也会自己进行补充），并且电影还限制了观众的视角，以便动态模糊效果达到最佳状态。

同样的游戏如果24帧会觉得卡顿，是因为视角的变化，导致动态模糊中的信息丢失。

**什么是RenderThread？**

RenderThread也就是我们常说的渲染线程直接看总结了：

1. 将Main Thread维护的Display List同步到Render Thread维护的Display List去。这个同步过程由Render Thread执行，但是Main Thread会被阻塞住。
2. 如果能够完全地将Main Thread维护的Display List同步到Render Thread维护的Display List去，那么Main Thread就会被唤醒，此后Main Thread和Render Thread就互不干扰，各自操作各自内部维护的Display List；否则的话，Main Thread就会继续阻塞，直到Render Thread完成应用程序窗口当前帧的渲染为止。
3. Render Thread在渲染应用程序窗口的Root Render Node的Display List之前，首先将那些设置了Layer的子Render Node的Display List渲染在各自的一个FBO（Frame Buffer Object）上，接下来再一起将这些FBO以及那些没有设置Layer的子Render Node的Display List一起渲染在Frame Buffer之上，也就是渲染在从Surface Flinger请求回来的一个图形缓冲区上。这个图形缓冲区最终会被提交给Surface Flinger合并以及显示在屏幕上。

其中第二步中的DisplayList同步很关键，它使得Main Thread和Render Thread可以并行执行，这意味着Render Thread在渲染应用程序窗口当前帧的Display List的同时，Main Thread可以去准备应用程序窗口下一帧的Display List，这样就使得应用程序窗口的UI更流畅。

**Vsync和Choreographer**

Vsync 信号一般由硬件产生，也可以用软件模拟。他的主要作用是唤醒 Choreographer 来做 App 的绘制操作。

渲染层\(App\)与 Vsync 打交道的是 Choreographer，而合成层与 Vsync 打交道的，则是 SurfaceFlinger。SurfaceFlinger 也会在 Vsync 到来的时候，将所有已经准备好的 Surface 进行合成操作。

**双缓冲机制**

这里需要注意的事GPU会将完成绘制的一帧放入Back buffer，然后将绘制完成的一帧图像copy到Frame Buffer。然后屏幕会从Frame Buffer中读取并显示。

**双缓存的交换 是在Vsyn到来时进行，交换后屏幕会取Frame buffer内的新数据，而实际 此时的Back buffer 就可以供GPU准备下一帧数据了。 如果 Vsyn到来时 CPU/GPU就开始操作的话，是有完整的16.6ms的，这样应该会基本避免jank的出现了**

**三缓存机制**

在双缓冲的基础上新增一个Buffer区，当一个VSYNC信号到来时，系统发现A、B缓冲区正在使用中，但是CPU此时处于空闲状态。此时，系统会再次分配一个缓冲区C，让CPU立即开始下一帧的计算。

**Choreographer**

Choreographer的主要作用有两个：

1. **承上**：负责接收和处理 App 的各种更新消息和回调，等到 Vsync 到来的时候统一处理。比如集中处理 Input\(主要是 Input 事件的处理\) 、Animation\(动画相关\)、Traversal\(包括 measure、layout、draw 等操作\) ，判断卡顿掉帧情况，记录 CallBack 耗时等
2. **启下**：负责请求和接收 Vsync 信号。接收 Vsync 事件回调\(通过 FrameDisplayEventReceiver.onVsync \)；请求 Vsync\(FrameDisplayEventReceiver.scheduleVsync\) .

**doFrame方法**

doFrame 函数主要做下面几件事

1. 计算掉帧逻辑： Vsync 信号到来的时候会标记一个 start\_time ，执行 doFrame 的时候标记一个 end\_time ，这两个时间差就是 Vsync 处理时延，也就是掉帧
2. 记录帧绘制信息
3. 执行 CALLBACK\_INPUT、CALLBACK\_ANIMATION、CALLBACK\_INSETS\_ANIMATION、CALLBACK\_TRAVERSAL、CALLBACK\_COMMIT
4. 根据情况出发下一个Vsync的申请

比如input事件经过处理，最终会传给 DecorView 的 dispatchTouchEvent，这就到了我们熟悉的 Input 事件分发。

**LinearLayout和RelativeLayout的性能对比**

（1）RelativeLayout慢于LinearLayout是因为它会让子View调用2次measure过程（横向和纵向各有一次），而LinearLayout只需一次，但是有weight属性存在时，LinearLayout也需要两次measure。

**硬件加速绘制和软件绘制的差别**

硬件加速绘制与软件绘制整个流程差异非常大，最核心就是我们通过 GPU 完成 Graphic Buffer 的内容绘制。此外硬件绘制还引入了一个 DisplayList 的概念，每个 View 内部都有一个 DisplayList，当某个 View 需要重绘时，将它标记为 Dirty。

当需要重绘时，仅仅只需要重绘一个 View 的 DisplayList，而不是像软件绘制那样需要向上递归。这样可以大大减少绘图的操作数量，因而提高了渲染效率。

### 4. Java的内存模型和GC算法

#### 4.1 JVM内存模型简述

![](https://pic1.zhimg.com/80/v2-354d31865d1fb3362f5a1ca938f9a770_1440w.jpg)

从上图中可以看到，JVM主要包括4个部分；

1. 类加载器\(ClassLoader\):在 JVM 启动时或者在类运行将需要的 class 加载到JVM中
2. 执行引擎:负责执行 class 文件中包含的字节码指令;
3. 内存区\(也叫运行时数据区\):是在 JVM 运行的时候操作所分配的内存区。内存区又被分为5个小部分：
   1. 方法区\(MethodArea\):用于存储类结构信息的地方，包括常量池、静态常量、构造函数等。
   2. java 堆\(Heap\):存储 java 实例或者对象的地方。这块是 GC 的主要区域。从存储的内容我们可以很容易知道，方法和堆是被所有 java 线程共享的。
   3. java 栈\(Stack\):java 栈总是和线程关联在一起，每当创一个线程时，JVM 就会为这个线程创建一个对应的 java 栈在这个 java 栈中,其中又会包含多个栈帧，每运行一个方法就建一个栈帧，用于存储局部变量表、操作栈、方法返回等。
   4. 程序计数器\(PCRegister\):用于保存当前线程执行的内存地址。
   5. 本地方法栈\(Native MethodStack\):和 java 栈的作用差不多，只不过是为 JVM 使用到 native 方法服务的。
4. 本地方法接口:主要是调用 C 或 C++实现的本地方法及回调结果。

#### 4.2 描述一下 GC 的原理和回收策略?

垃圾回收一般是找到垃圾，然后回收垃圾。

**1. 找到垃圾的方式**：

* **引用计数算法**（Reachability Counting）是通过在对象头中分配一个空间来保存该对象被引用的次数（Reference Count）。如果该对象被其它对象引用，则它的引用计数加1，如果删除对该对象的引用，那么它的引用计数就减1，当该对象的引用计数为0时，那么该对象就会被回收。
* **可达性分析算法**（Reachability Analysis）的基本思路是，通过一些被称为引用链（GC Roots）的对象作为起点，从这些节点开始向下搜索，搜索走过的路径被称为（Reference Chain\)，当一个对象到 GC Roots 没有任何引用链相连时（即从 GC Roots 节点到该节点不可达），则证明该对象是不可用的。

**2. 回收垃圾组合拳**

1. **标记清除算法**（Mark-Sweep）是最基础的一种垃圾回收算法，它分为2部分，先把内存区域中的这些对象进行标记，哪些属于可回收标记出来，然后把这些垃圾拎出来清理掉。（回收后导致内存碎片化，不连续）
2. **复制算法**（Copying）是在标记清除算法上演化而来，解决标记清除算法的内存碎片问题。它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。保证了内存的连续可用，内存分配时也就不用考虑内存碎片等复杂情况，逻辑清晰，运行高效。（代价太高，意味着只能使用一半的内存）
3. **标记整理算法**（Mark-Compact）标记过程仍然与标记 --- 清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，再清理掉端边界以外的内存区域。

**3. 内存模型与回收策略**

内存主要被分为三块：**新生代（Youn Generation）、老生代（Old Generation）、持久代（Permanent Generation）**。三代的特点不同，造就了他们使用的GC算法不同，新生代适合生命周期较短，快速创建和销毁的对象，旧生代适合生命周期较长的对象，持久代在Sun Hotpot虚拟机中就是指方法区（有些JVM根本就没有持久代这一说法）。

**新生代（Youn Generation）**：大致分为Eden区和Survivor区，Survivor区又分为大小相同的两部分：FromSpace和ToSpace。

* **Eden 区**

  大多数情况下，对象会在新生代 Eden 区中进行分配，当 Eden 区没有足够空间进行分配时，虚拟机会发起一次 Minor GC，Minor GC 相比 Major GC 更频繁，回收速度也更快。

  通过 Minor GC 之后，Eden 会被清空，Eden 区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到 Survivor 区。

* **Survivor 区**

  Survivor 区相当于是 Eden 区和 Old 区的一个缓冲，类似于我们交通灯中的黄灯。Survivor 又分为2个区，一个是 From 区，一个是 To 区。每次执行 Minor GC，会将 Eden 区和 From 存活的对象放到 Survivor 的 To 区（如果 To 区不够，则直接进入 Old 区）。

**为啥需要将新生代分为两个区？**

Survivor 的存在意义就是减少被送到老年代的对象，进而减少 Major GC 的发生。Survivor 的预筛选保证，只有经历16次 Minor GC 还能在新生代中存活的对象，才会被送到老年代

**为啥Survivor需要俩个？**

设置两个 Survivor 区最大的好处就是解决内存碎片化。可以参考上面的复制算法。

**旧生代（Old Generation）**：老年代占据着2/3的堆内存空间，只有在 Major GC 的时候才会进行清理，每次 GC 都会触发“Stop-The-World”，内存越大，stop的时间也越长，所以老年代这里采用的是标记 --- 整理算法。

此外还要注意几个点：

1. **大对象**：大对象指需要大量连续内存空间的对象，这部分对象不管是不是“朝生夕死”，都会直接进到老年代。避免发生大量的复制
2. **长期存活对象**：虚拟机给每个对象定义了一个对象年龄（Age）计数器。经历一次Minor GC年龄+1，到15岁后就进老年代了。
3. **动态对象年龄**：虚拟机并不重视要求对象年龄必须到15岁，才会放入老年区，如果 Survivor 空间中相同年龄所有对象大小的总合大于 Survivor 空间的一半，年龄大于等于该年龄的对象就可以直接进去老年区，无需等你“成年”。

**强引用置为 null，会不会被回收?**

不会立即释放对象占用的内存。 如果对象的引用被置为 null，只是断开了当前线程栈帧中对该对象的引用关系，而 垃圾收集器是运行在后台的线程，只有当用户线程运行到安全点\(safe point\)或者安全区域才会扫描对象引用关系，扫描到对象没有被引用则会标记对象，这时候仍然不会立即释放该对象内存，因为有些对象是可恢复的\(在 finalize 方法中恢复引用 \)。只有确定了对象无法恢复引用的时候才会清除对象内存。

### 5. HashMap、ConcurrentHashMap、hash\(\)相关原理解析?

#### 5.1 HashMap相关

在JDK1.7中，HashMap底层是用数组+链表的形式，在JDK1.8的时候就改成了数组+链表和红黑树了。

当链表长度大于8的时候，链表会转化为红黑树。

同一个key对应多个值是hash冲突导致的，HashMap是线程不安全的，在并发场景下链表可能变成形成环状链表。

#### 5.2 ConcurrentHashMap 线程安全的map

由于HashMap是线程不安全，相对应的就会有线程安全的ConcurrentHashMap，下面简单介绍下ConcurrentHashMap：

主要是1.7版本和1.8版本的差异，

1. 在1.7中ConcurrentHashMap采用了分段锁技术，其中 Segment 继承于 ReentrantLock。每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。

   比如put中，首先会通过key定位到对应的Segment，然后会尝试获取锁，如果获取失败肯定就有其他线程存在竞争，会通过自旋锁增加竞争，失败一定次数后，会变成阻塞锁竞争，确保竞争到资源。

2. 1.8是抛弃了原有的 Segment 分段锁，而采用了 `CAS + synchronized` 来保证并发安全性。采取跟HashMap一样的策略，hash冲突后面会转化成红黑树。

#### 5.3 ArrayMap 跟 SparseArray 在 HashMap 上面的改进?

**SparseArray**

SparseArray 比 HashMap 更省内存，在某些条件下性能更好，主要是因为它避免了对key的自动装箱，内部是通过两个数组来进行数据存储的，一个存key，一个存value。它存储的元素是按key从小到大有序的，所以查找的时候可以用二分查找。

**ArrayMap**

有一个数组mHashes，存储key的hash值，mArrray数组是mHashes的两倍，依次保存key和value。查找的时候也是通过二分查找的方式，如果出现hash冲突会在相邻的位置插入。

### 6. Java的多线程安全问题

在并发编程中，我们通常会遇到以下三个问题：原子性问题，可见性问题，有序性问题。我们先看具体看一下这三个概念：

* **原子性**：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

  只有简单的读取、赋值（而且必须是将数字赋值给某个变量，变量之间的相互赋值不是原子操作）才是原子操作（如y=x,i++不是原子操作）。

* **可见性：**是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。
* **有序性**：即程序执行的顺序按照代码的先后顺序执行。举个简单的例子，看下面这段代码：

#### 6.1 synchronized的原理

synchronized 代码块是由一对儿 monitorenter/monitorexit 指令实现的。JVM对此进行优化，提供了几种不同的锁，自旋锁，偏向锁，轻量级锁，重量级锁。并能在检测到不同的竞争环境的时候，进行不同锁的切换，也就是锁的升级，降级。

**自旋锁**：就是让CPU做无用功，目的是占着CPU不放，等待获取锁的机会。

**偏向锁**：无竞争关系的时候，也就是相当于添加了一个变量，判断为true的时候无需再走加锁、解锁的流程

**轻量级锁**：当偏向锁运行在同步代码块的时候，另一个线程过来竞争的时候，偏向锁升级为轻量级锁；

**重量级锁**：重量锁在 JVM 中又叫对象监视器\(Monitor\)，他包含至少一个竞争锁队列，和一个阻塞队列吧。前者负责互斥，后者负责同步；

此外synchronized 修饰静态方法获取的是类锁\(类的字节码文件对象\)。

synchronized 修饰普通方法或代码块获取的是对象锁。这种机制确保了同一时刻对于每一个类实例，其所有声明为 synchronized 的成员函数中至多只有一个处于可执行状态。

#### 6.2 volatile 原理。

观察加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令，lock前缀指令实际上相当于一个**内存屏障**（也成内存栅栏），内存屏障会提供3个功能：

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. 它会强制将对缓存的修改操作立即写入主存；
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

根据内存屏障的功能，保证了可见性和禁止指令重排序。

#### 6.3 ReentrantLock的原理

ReetrantLock实现依赖于AQS\(AbstractQueuedSynchronizer\)，支持公平锁和非公平锁的重入锁。

* 公平锁：多个线程申请获取同一资源时，必须按照申请顺序，依次获取资源。
* 非公平锁：资源释放时，任何线程都有机会获得资源，而不管其申请顺序。

**公平锁**

FairSync继承Sync，而Sync继承AbstractQueuedSynchronizer。ReentrantLock调用lock方法，最终会调用sync的tryAcquire函数，获取资源。FairSync的tryAcquire函数，**当前线程只有在队列为空或者时队首节点的时候，才能获取资源，否则会被加入到阻塞队列中。**下面的hasQueuedPredecessors函数用于判断队列是否为空，或者当前元素是否为队首元素。

**非公平锁**

NoFairSync同样继承Sync，ReentrantLock调用lock方法，最终会调用sync的tryAcquire函数，获取资源。而NoFairSync的tryAcquire函数，会调用父类Sync的方法nofairTryAcquire函数。通过对比可以发现，如果资源释放时，新的线程会尝试CAS操作获取资源，而不管阻塞队列中时候有先于其申请的线程。

1. 先获取state值，若为0，意味着此时没有线程获取到资源，CAS将其设置为1， 设置成功则代表获取到排他锁了;
2. 若 state 大于 0，肯定有线程已经抢占到资源了，此时再去判断是否就是自己 抢占的，是的话，state 累加，返回 true，重入成功，state 的值即是线程重入的次数
3. 其他情况，则获取锁失败。

**可重入性**

获取独占资源的线程，可以重复得获取该独占资源，不需要重复请求。注意释放资源的时候，重入多少次，就必须释放多少次。

#### 6.4 CAS的介绍

CAS，Compare and Swap 即比较并交换，设计并发算法时常用到的一种技术，这个指令也带有`Lock`前缀，而且由于CAS是硬件级别的操作，效率会比普通锁更高一些。

#### 6.5 线程死锁的4个条件

**什么是死锁？**

当线程 A 持有独占锁 a，并尝试去获取独占锁 b 的同时，线程 B 持有独占锁 b，并尝试获取独占锁 a 的情况下，就会发生 AB 两个线程由于互相持有对方需要的锁，而发生的阻塞现象，我们称为死锁。

造成死锁的四个条件:

* 互斥条件:一个资源每次只能被一个线程使用。
* 请求与保持条件:一个线程因请求资源而阻塞时，对已获得的资源保持不放
* 不剥夺条件:线程已获得的资源，在未使用完之前，不能强行剥夺。
* 循环等待条件:若干线程之间形成一种头尾相接的循环等待资源关系。

#### 6.6 什么导致线程阻塞?

为了解决对共享存储区的访问冲突，Java 引入了同步机制，Java 提供了大量方法来支持阻塞，下面 让我们逐一分析。

**suspend\(\) 和 resume\(\) 方法**:两个方法配套使用，suspend\(\)使得线程进入阻塞状态，并且不会自动恢复，必须其对应的 resume\(\) 被调用，才能使得线程重新进入可执行状态。

**yield\(\) 方法**: yield\(\) 使得线程放弃当前分得的 CPU 时间，但是不使线程阻塞，即线程仍处于可执行状态，随时可能再次分得 CPU 时间。调用 yield\(\) 的效果等价于调度程序认为该线程已执行了足够的时间从而转到另一个线程。

**join方法**：join方法的本质调用的是Object中的wait方法实现线程的阻塞，不过他阻塞的事主线程。

**wait、sleep 的区别**

最大的不同是在等待时 wait 会释放锁，而 sleep 一直持有锁。wait 通常被用于线程间交互，sleep 通常被用于暂停执行。

* 首先，要记住这个差别，“sleep 是 Thread 类的方法,wait 是 Object 类中。
* Thread.sleep 不会导致锁行为的改变，如果当前线程是拥有锁的，那么sleep后依然持有锁
* Thread.sleep 和 Object.wait 都会暂停当前的线程，对于 CPU 资源来说，他都暂时不需要占用CPU执行时间，区别是代用wait后，需要别的线程执行 notify/notifyAll 才能够重新获得 CPU 执行时间。
* 线程的状态参考 Thread.State 的定义。新创建的但是没有执行\(还没有调用 start\(\)\)的线程处于“就绪”，或者说 Thread.State.NEW 状态。而Thread.State.BLOCKED\(阻塞\)表示线程正在获取锁时，因为锁不能获取到而被迫暂停执行下面的指令，一直等到这个锁被别的线程释放。

**notify** 运行过程：

当线程 A\(消费者\)调用 wait\(\)方法后，线程 A 让出锁，自己进入等待状态，同时加入锁对象的等待队列。 线程 B\(生产者\)获取锁后，调用 notify 方法通知锁对象的等待队列，使得线程 A 从等待队列进入阻塞队列。 线程 A 进入阻塞队列后，直至线程 B 释放锁后，线程 A 竞争得到锁继续从 wait\(\)方法后执行。

#### 6.7 线程的生命周期

* NEW:创建状态，线程创建之后，但是还未启动。
* RUNNABLE:运行状态，处于运行状态的线程，但有可能处于等待状态，
* WAITING:等待状态，一般是调用了 wait\(\)、join\(\)、LockSupport.spark\(\)
* TIMED\_WAITING:超时等待状态，也就是带时间的等待状态。一般是调wait\(time\)等方法。
* BLOCKED:阻塞状态，等待锁的释放，例如调用了 synchronized 增加了锁
* TERMINATED:终止状态，一般是线程完成任务后退出或者异常终止。

#### 6.8 run\(\)和 start\(\)方法区别?

1. start\(\)方法来启动线程，真正实现了多线程运行，这时无需等待 run 方法体代码执行完毕而直接继续执行下面的代码:
2. run\(\)方法当作普通方法的方式调用，程序还是要顺序执行，还是要等待 run 方法体执行完毕后才可继续执行下面的代码:

#### 6.9 如何停止一个线程

1. Thread有直接暂停的方法——Stop，但是这个方法被废弃了，因为不安全
2. Thread的间接停止方式——通知目标线程自行停止
   * interrupt 系统支持，出发方式是抛出异常，且调用JNI方法具有一定的开销
   * volatile 修饰对的Boolean  通过布尔值进行判断也可以抛出异常 （一般推荐）

### 7.Android中的跨进程通信

我们知道Android是基于Linux，Linux中常见的IPC方式。

**共享内存**

属于共享物理内存方式：多个进程共享同一段物理内存，当某个进程改变内存内容时，其它进程都能够知道。此种方式无需拷贝内容，但是需要信号量进行进程间同步。

**通过内核中转**

先将A发送的内容拷贝到内核，这过程可以理解为存储，再从内核拷贝到B的用户空间，这过程可以理解为转发，因此一次"存储-转发"过程需要两次内容拷贝。

**Android中的Binder**

虽然Linux提供了上述\(还有其它的如信号量等\)的IPC方式，但是由于每种方式都有其缺点，因此Android另起炉灶弄了另一种方式：Binder。

先来了解一下内存映射\(mmap\)。

上面提到过，"存储-转发"过程需要在用户空间和内核空间之间拷贝数据，还有一种不需要拷贝的方式：内存映射，内存映射就是将文件内容映射到用户空间中，只要在用户空间对映射对象进行操作，就相当于对文件进行操作。

![](https://mmbiz.qpic.cn/mmbiz_png/MOu2ZNAwZwMdfb25nt76qmRW3aHYfib3JiaZj0x78BznyLs3JUt4BbtVaY8ibVlxApXyoszeCfq3R8iccicp9Tyy5rg/640)

如上图，进程空间与内核空间共享磁盘上的一个映射文件，当进程空间往映射文件写入数据后（此处不是真正的I/O，而是内存读写），相当于写入了内核空间，反之亦然。

**Binder**

![](https://user-gold-cdn.xitu.io/2020/4/7/17154f81bb8bd672?)

进程A给进程B发送一段信息，流程如下：

> 1、进程A通过系统调用拷贝内容到内核空间。
>
> 2、由于内核空间与进程B做了内存映射，因此进程B能够知道内核空间的信息。

从上可知，通过Binder，进程A给进程B发送信息只进行了一次数据拷贝.

Binder也可以看成一个CS架构，客户端在请求服务端通信的时候，并不是直接和服务端的某个对象联系，而是用到了服务端的一个代理对象，通过对这个代理对象操作，然后代理类会把方法对应的code、传输的序列化数据、需要返回的序列化数据交给底层，也就是Binder驱动。

为什么不采用共享内存：多进程使用同一内存，容易导致死锁，不安全。

**Binder的优势**

1. 效率高，拷贝次数越少，传输效率越高
2. 稳定性：共享内存需要处理并发同步问题，容易出现死锁和资源竞争，Binder 基于 C/S 架构 ，Server 端与 Client 端相 对独立，稳定性较好。Socket同样是基于C/S架构但是主要用于网络间传输，效率较低。
3. 安全性：Binder 机制为每个进程分配了 UID/PID，且 在 Binder 通信时会根据 UID/PID 进行有效性检测。

**AIDL**

AIDL 的本质是系统提供了一套可快速实现 Binder 的工具。关键类和方法:

* AIDL 接口:继承 IInterface。
* Stub 类:Binder 的实现类，服务端通过这个类来提供服务。
* Proxy 类:服务端的本地代理，客户端通过这个类调用服务端的方法。
* asInterface\(\):客户端调用，将服务端返回的 Binder 对象，转换成客户端所需要的AIDL对象，如果客户端和服务端位于同一进程中，则返回stub对象，否则返回stub.proxy
* asBinder\(\):根据当前调用情况返回代理 Proxy 的 Binder 对象。
* onTransact\(\):运行在服务端的 Binder 线程池中，当客户端发起跨进程请求时，远程请求会通过底层封装后交由此方法处理
* transact\(\):运行在客户端，当客户端发起远程请求的同时将当前线程挂起。

当有多个业务模块都需要 AIDL 来进行 IPC，此时需要为每个模块创建特定的 aidl文件，那么相应的 Service 就会很多。必然会出现系统资源耗费严重、应用过度重量级的问题。解决办法是建立 Binder 连接池，即将每个业务模块的 Binder 请求统一转发到一个远程 Service 中去执行，从而避免重复创建 Service。

工作原理:每个业务模块创建自己的 AIDL 接口并实现此接口，然后向服务端提供自己的唯一标识和其对应的 Binder 对象。服务端只需要一个 Service 并提供一个 queryBinder 接口，它会根据业务模块的特征来返回相应的 Binder 对象，不同的业务模块拿到所需的 Binder 对象后就可以进行远程方法的调用了。

### 8. Android系统的启动流程

**Android系统架构**

首先来看Android系统的架构图，如下所示：

![](https://developer.android.com/guide/platform/images/android-stack_2x.png?hl=zh-cn)

1. **应用程序**
2. **Java FrameWork层**
3. **系统运行层**：
   * C/C++库：许多核心 Android 系统组件和服务\(例如 ART 和 HAL\)构建自原生代码，需要用c来编写
   * Android Runtime：对于运行 Android 5.0（API 级别 21）或更高版本的设备，每个应用都在其自己的进程中运行，并且有其自己的 [Android Runtime \(ART\)](https://source.android.com/devices/tech/dalvik/index.html?hl=zh-cn) 实例。ART 编写为通过执行 DEX 文件在低内存设备上运行多个虚拟机，DEX 文件是一种专为 Android 设计的字节码格式，经过优化，使用的内存很少。编译工具链（例如 [Jack](https://source.android.com/source/jack.html?hl=zh-cn)）将 Java 源代码编译为 DEX 字节码，使其可在 Android 平台上运行。ART的主要功能：
     * 预先 \(AOT\) 和即时 \(JIT\) 编译
     * 优化的垃圾回收 \(GC\)
     * 在 Android 9（API 级别 28）及更高版本的系统中，支持将应用软件包中的 [Dalvik Executable 格式 \(DEX\) 文件转换为更紧凑的机器代码](https://developer.android.com/about/versions/pie/android-9.0?hl=zh-cn#art-aot-dex)。
     * 更好的调试支持，包括专用采样分析器、详细的诊断异常和崩溃报告，并且能够设置观察点以监控特定字段
4. **硬件抽象层**：硬件抽象层 \(HAL\) 提供标准界面，向更高级别的 Java API 框架显示设备硬件功能。HAL 包含多个库模块，其中每个模块都为特定类型的硬件组件实现一个界面，例如相机或蓝牙模块。当框架 API 要求访问设备硬件时，Android 系统将为该硬件组件加载库模块。
5. **Linux 内核层**

**Android系统的启动**

Android系统的启动可以参考架构图，是从下到上开始进行的。

![](https://mmbiz.qpic.cn/mmbiz_png/jfNJsrA21FJvM8ibETbuD2XeblOibqJyK2GgwnKYRfzhibLLhHkDnjWJYjAmYC2tovOjQW9ZAibXuyB1jichecgvYicQ/640)

1. 启动电源以及系统启动
2. 引导程序BootLoader：它是Android操作系统开始运行前的一个小程序，主要将操作系统OS拉起来并进行
3. Linux内核启动（Linux层）
4. Swapper进程（加载硬件功能HAL）
5. 启动init进程（Native层）：
   1. 创建和挂载启动所需的文件目录
   2. 初始化和启动属性服务
   3. 解析init.rc配置文件并启动Zygote进程
6. 启动Zygote进程：
   1. 创建AppRuntime，执行其start方法，启动Zygote进程\(加载虚拟机\)。
   2. 创建JVM并为JVM注册JNI方法。
   3. 使用JNI调用**ZygoteInit的main函数进入Zygote的Java FrameWork层**。
   4. 使用registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等等AMS的请求去创建新的应用进程。
   5. 启动SystemServer进程。
7. SystemServer进程启动:
   1. 启动Binder线程池，这样就可以与其他进程进行Binder跨进程通信。
   2. 创建SystemServiceManager，它用来对系统服务进行创建、启动和生命周期管理。
   3. 启动各种系统服务：引导服务、核心服务、其他服务，共100多种。应用开发主要关注引导服务ActivityManagerService、PackageManagerService和其他服务WindowManagerService、InputManagerService即可
8. fork并启动Launcher进程。

**Activity启动流程中，大部分都是用Binder通讯，为啥跟Zygote通信的时候要用socket呢？**

* ServiceManager不能保证在zygote起来的时候已经初始化好，所以无法使用Binder。
* Socket 的所有者是 root，只有系统权限用户才能读写，多了安全保障。
* Binder工作依赖于多线程，但是fork的时候是不允许存在多线程的，多线程情况下进程fork容易造成死锁，所以就不用Binder了

### 9. ANR异常

Application no Response程序无响应，四大组件都有可能引起ANR。底层逻辑其实就是Handler的机制，就是组件利用延迟消息发送一个ANR的message，Msg能够及时收到响应就取消延迟消息的发送，否则出发ANR。

每次产生ANR之后，系统都会向`/data/anr/traces.txt`中写入新的数据

四大组件的超时时间都不一致，主要讲一下input事件响应超时。

应用进程自身引起 例如： 1.主线程阻塞、挂起、死循环 2.应用进程的其他线程的CPU占用率高，使得主线程无法抢占到CPU时间片

其他进程间接引起（误伤） 例如： 1.当前应用进程进行进程间通信请求其他进程，其他进程的操作长时间没有反馈 2.其他进程的CPU占用率极高，使得当前应用进程无法抢占到CPU时间片

#### 通过input来看ANR

首先了解几个关键类：

* **InputReader线程**

  主要工作是添加event到mInBoundQueue，并唤醒InputDispatcher的mLooper

* **mInBoundQueue**

  相当于消息机制中的消息队列

* **InputDispatcher**

  输入事件分发线程：InputDispatcher永远只能单线程处理一个mPendingEvent，如果分发失败，下一次会继续分发同一个mPendingEvent。

* **InputChannel**

  `InputChannel`的创建是在 `ViewRootImpl`中`setView`方法中。利用Socket的手段，接受事件。

大致流程如下：InputReader发现新的event，放到InBoundQueue，**InputDispatcher**取出对应的事件，进行分发。通过InputChannel把Event传递到App层。当App处理完消息，会发送一个finish信号回来，移除waitQueue中的event。并唤醒mLooper进行第二步。

### 10.线程池相关的

#### 10.1 线程池的主要组成

一个线程池包括以下四个基本组成部分：

1. 线程池管理器（ThreadPool）：用于创建并管理线程池，包括 创建线程池，销毁线程池，添加新任务；
2. 工作线程（WorkThread）：线程池中线程，在没有任务时处于等待状态，可以循环的执行任务；
3. 任务接口（Task）：每个任务必须实现的接口，以供工作线程调度任务的执行，它主要规定了任务的入口，任务执行完后的收尾工作，任务的执行状态等；
4. 任务队列（taskQueue）：用于存放没有处理的任务。提供一种缓冲机制。

#### 10.2 创建线程池的几个参数

```java
new ThreadPoolExecutor(corePoolSize, maximumPoolSize,keepAliveTime, 
milliseconds,runnableTaskQueue, threadFactory,handler);
```

**corePoolSize（线程池的基本大小）**：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程。

**maximumPoolSize（线程池最大大小）**：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是如果使用了无界的任务队列这个参数就没什么效果。

**runnableTaskQueue（任务队列）**：用于保存等待执行的任务的阻塞队列。

**ThreadFactory**：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字，Debug和定位问题时非常又帮助。

**RejectedExecutionHandler（拒绝策略）**：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。以下是JDK1.5提供的四种策略。n AbortPolicy：直接抛出异常。

**keepAliveTime（线程活动保持时间）**：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。

**TimeUnit（线程活动保持时间的单位）**：可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒\(MILLISECONDS\)，微秒\(MICROSECONDS, 千分之一毫秒\)和毫微秒\(NANOSECONDS, 千分之一微秒\)。

**线程池的运行策略**

1. 线程数量未达到corePoolSize，则新建一个线程\(核心线程\)执行任务
2. 线程数量达到了corePools，则将任务移入队列等待
3. 队列已满，新建线程\(非核心线程\)执行任务
4. 队列已满，总线程数又达到了maximumPoolSize，就会由\(RejectedExecutionHandler\)抛出异常（执行拒绝策略）

**常见的四种拒绝策略**

1. AbortPolicy：丢弃任务并抛出RejectedExecutionException异常
2. DiscardPolicy：丢弃任务，但是不抛出异常。
3. DisCardOldSetPolicy：丢弃队列最前面的任务，然后提交新来的任务
4. CallerRunPolicy：由调用线程（提交任务的线程，主线程）处理该任务

#### 10.3 常见的4种线程池

1. CachedThreadPool\(\)：可缓存线程池。
   1. 线程数无限制
   2. 有空闲线程则复用空闲线程，若无空闲线程则新建线程 一定程序减少频繁创建/销毁线程，减少系统开销
2. FixedThreadPool\(\)：定长线程池。
   1. 可控制线程最大并发数（同时执行的线程数）
   2. 超出的线程会在队列中等待
3. ScheduledThreadPool\(\)：定时线程池   支持定时及周期性任务执行。
4. SingleThreadExecutor\(\)：单线程化的线程池

#### 10.4 线程池常见的5种状态

![](https://user-gold-cdn.xitu.io/2020/3/20/170f73cc0dad76a5)

1. 线程池的初始化状态是RUNNING，能够接收新任务，以及对已添加的任务进行处理。
2. 线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务。 调用线程池的shutdown\(\)接口时，线程池由RUNNING -&gt; SHUTDOWN。
3. 线程池处在STOP状态时，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。 调用线程池的shutdownNow\(\)接口时，线程池由\(RUNNING or SHUTDOWN \) -&gt; STOP。
4. 当所有的任务已终止，ctl记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated\(\)。terminated\(\)在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated\(\)函数来实现。
5. 当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -&gt; TIDYING。
6. 当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -&gt; TIDYING。 线程池彻底终止，就变成TERMINATED状态。线程池处在TIDYING状态时，执行完terminated\(\)之后，就会由 TIDYING -&gt; TERMINATED。

**关闭线程池的两种方式**

**shutDown\(\)**

当线程池调用该方法时,线程池的状态则立刻变成SHUTDOWN状态。此时，则不能再往线程池中添加任何任务，否则将会抛出RejectedExecutionException异常。但是，此时线程池不会立刻退出，直到添加到线程池中的任务都已经处理完成，才会退出。

**shutdownNow\(\)**

根据JDK文档描述，大致意思是：执行该方法，线程池的状态立刻变成STOP状态，并试图停止所有正在执行的线程，不再处理还在池队列中等待的任务，当然，它会返回那些未执行的任务。

它试图终止线程的方法是通过调用Thread.interrupt\(\)方法来实现的，但是大家知道，这种方法的作用有限，如果线程中没有sleep、wait、Condition、定时锁等应用, interrupt\(\)方法是无法中断当前的线程的。所以，ShutdownNow\(\)并不代表线程池就一定立即就能退出，它可能必须要等待所有正在执行的任务都执行完成了才能退出。

**设置线程池的大小**

* 如果是CPU密集型应用，则线程池大小设置为N+1
* 如果是IO密集型应用，则线程池大小设置为2N+1

## 2. Android常见问题补充

### 1. 横竖屏切换时候 Activity 的生命周期

不设置 Activity 的 android:configChanges 时，切屏会重新回调各个生命周期，切横屏时会执行一次，切竖屏时会执行两次。 设置 Activity 的android:configChanges=”orientation”时，切屏还是会调用各个生命周期，切换横竖屏只会执行一次 设置 Activity 的 android:configChanges=”orientation\|keyboardHidden”时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged 方法。

### 2. AsyncTask 的缺陷和问题，说说他的原理。

**AsyncTask** 是什么?

AsyncTask 是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程并在主线程中更新 UI。

他的本质是两个线程池，和一个handler。线程池一个用来任务排队，一个用来执行任务

**关于线程池**

AsyncTask 对应的线程池 ThreadPoolExecutor 都是进程范围内共享的，且都是static 的，所以是 Asynctask 控制着进程范围内所有的子类实例。也可以自定义线程池。

补充问题：

1. 生命周期：AsynTask 会一直执行，直到 doInBackground\(\)方法 执行完毕，不会随着Activity的销毁而销毁，所以要记得做正确的取消操作
2. 内存泄漏：如果 AsyncTask 被声明为 Activity 的非静态内部类，那么 AsyncTask 会保留一个对 Activity 的引用。如果 Activity 已经被销毁，AsyncTask 的后台线程还在执行，它将继续在内存里保留这个引用，导致 Activity 无法被回收，引起内存泄漏。
3. 结果丢失：屏幕旋转或 Activity 在后台被系统杀掉等情况会导致 Activity 的重新创建，之前运行的 AsyncTask 会持有一个之前 Activity 的引用，这个引用已经无效，这时调用 onPostExecute\(\)再去更新界面将不再生效。
4. 并行还是串行：在 Android1.6 之前的版本，AsyncTask 是串行的，在 1.6 之后3.0之前是并行，后来又改为串行的执行任务

### 3. 保存 & 恢复Activity/Fragment状态缓存 - onSaveInstanceState\(\)、onRestoreInstanceState\(\)

**Activity的状态保存**

onSaveInstanceState：是当系统 **未经你许可** 时，**可能** 销毁了你的Activity（如内存不足，用户按下home键，横竖屏切换），则会被系统调用 。如果是主动销毁，比如说按返回键则不会触发。

onRestoreInstanceState（）：若 异常关闭了Activity，即调用了onSaveInstanceState（） & 下次启动时会调用onRestoreInstanceState（）。此时的调用顺序是：

1. onCreate（）
2. onStart（）
3. onRestoreInstanceState（）
4. onResume（）

onSaveInstanceState的bundle参数会传递到onCreate方法中，可选择在onCreate（）中做数据还原。

**Fragment的状态保存**

1. Fragment 的状态保存, 在 Activity 的 onSaveInstanceState\(\)里, 调用了FragmentManger 的 saveAllState\(\)方法, 其中会对 mActive 中各个 Fragment 的实例状态和 View 状态分别进行保存.
2. FragmentManager 还提供了 public 方法: saveFragmentInstanceState\(\), 可以对单个 Fragment 进行状态保存, 这是提供给我们用的。
3. FragmentManager 的 moveToState\(\)方法中, 当状态回退到ACTIVITY\_CREATED, 会调用 saveFragmentViewState\(\)方法, 保存 View 的状态.

### 4. Android中的动画

Android中的动画主要分三大类：

1. tween 补间动画。通过指定 View 的初末状态和变化方式，对 View 的内容进行一系列的变化。只是影像变化，view 的实际位置还在原来地方。
2. frame 帧动画。AnimationDrawable 控制 animation-list.xml 布局，类似电影一帧一帧的动画效果
3. PropertyAnimation 属性动画 ，3.0 引入，属性动画核心思想是对值的变化，他真正的实现了 view 的移动。

其中最重要的就是属性动画，其中属性动画有几个重要的步骤；

1. 计算属性值
   * 计算已完成动画分数 elapsed fraction。为了执行一个动画，你需要创建一个ValueAnimator，并且指定目标对象属性的开始、结束和持续时间。
   * 计算插值\(动画变化率\)interpolated fraction 。当 ValueAnimator 计算完已完成的动画分数后，它会调用当前设置的 TimeInterpolator，去计算得到一个interpolated\(插值\)分数，在计算过程中，已完成动画百分比会被加入到新的插值计算中。
   * 计算属性值当插值分数计算完成后，ValueAnimator 会根据插值分数调用合适的TypeEvaluator 去计算运动中的属性值。 以上分析引入了两个概念:已完成动画分数\(elapsed fraction\)、插值分数\( interpolated fraction \)。
2. 为目标对象的属性设置属性值，即应用和刷新动画

### 5. Android中的Context

![](https://upload-images.jianshu.io/upload_images/10107787-ab22cce86fa4f1da.png)

如上图所示，**Context 使用了装饰模式，除了 ContextImpl 外，其他 Context 都是 ContextWrapper 的子类**

注意几点：

1. 任何一个 Context 的实例，只要调用 getApplicationContext\(\)方法都可以拿到我们的 Application 对象。getApplication也是获取Application的但是只有Activity个Service才能调用。
2. 创建对话框时不可以用 Application 的 context，只能用 Activity 的context

**activity 的 startActivity 和 context 的 startActivity 区别?**

\(1\)、从 Activity 中启动新的 Activity 时可以直接 mContext.startActivity\(intent\)就ok

\(2\)、如果从其他 Context 中启动 Activity 则必须给 intent 设置 Flag

怎么在 **Service** 中创建 **Dialog** 对话框?

1. 在我们取得 Dialog 对象后，需给它设置类型，即:dialog.getWindow\(\).setType\(WindowManager.LayoutParams.TYPE\_SYSTEM\_ALERT\)
2. 在 Manifest 中加上权限:

### 6.  bundle传递对象为什么需要序列化?Serialzable 和 Parcelable 的区别?

因为 bundle 传递数据时只支持基本数据类型，所以在传递对象时需要序列化转换成可存储或可传输的本质状态\(字节流\)。

1. 两者最大的区别在于 **存储媒介的不同**，`Serializable` 使用 **I/O 读写存储在硬盘上**，而 `Parcelable` 是直接 **在内存中读写**。很明显，内存的读写速度通常大于 IO 读写，所以在 Android 中传递数据优先选择 `Parcelable`。
2. Serializable 会使用反射，反射就开销大
3. Serializable序列化和反序列化过程需要大量 I/O 操作，Parcelable 自已实现封送和解封\(marshalled &unmarshalled\)操作不需要用反射，数据也存放在 Native 内存中，效率要快很多。

### 7. Bitmap 使用时候注意什么?

1. 选择合适的bitmap类型，图片规格:
   * ALPHA\_8  每个像素占用 1byte 内存
   * ARGB\_4444 每个像素占用 2byte 内存
   * ARGB\_8888 每个像素占用 4byte 内存\(默认\)
   * RGB\_565 每个像素占用 2byte 内存（没有透明通道）
2. 降低采样率inSampleSize。
   1. inSampleSize的值必须大于1时才会有效果，且采样率同时作用于宽和高；
   2. 假设一张1024_1024，模式为ARGB\_8888的图片,inSampleSize=2，原始占用内存大小是4MB，采样后的图片占用内存大小就是\(1024/2\)_  \(1024/2 \)\* 4 = 1MB
3. 复用内存。即，通过软引用\(内存不够的时候才会回收掉\)，复用内存块，不需要再重新给这个 bitmap 申请一块新的内存，避免了一次内存的分配和回收，从而改善了运行效率。
4. 使用 recycle\(\)方法及时回收内存。
5. 压缩图片

**在 Bitmap 里有两个获取内存占用大小的方法。**

getByteCount\(\):API12 加入，代表存储 Bitmap 的像素需要的最少内存。

getAllocationByteCount\(\):API19 加入，代表在内存中为 Bitmap 分配的内存大小，代替了 getByteCount\(\) 方法。在不复用 Bitmap 时，getByteCount\(\) 和getAllocationByteCount 返回的结果是一样的。在通过复用 Bitmap 来解码图片时，那么 getByteCount\(\) 表示新解码图片占用内存的大 小，

### 8. 编译期注解跟运行时注解

运行期注解\(RunTime\)利用反射去获取信息还是比较损耗性能的。

编译期\(Compile time\)注解，以及处理编译期注解的手段 APT 和 Javapoet，

### 9. Service的启动方式有什么区别

1. startService:

   onCreate\(\)---&gt;onStartCommand\(\) ---&gt; onDestory\(\)

   如果服务已经开启，不会重复的执行 onCreate\(\)， 而是会调用onStartCommand\(\)。一旦服务开启跟调用者\(开启者\)就没有任何关系了。 开启者退出了，开启者挂了，服务还在后台长期的运行。 开启者不能调用服务里面的方法。

2. bindService

   onCreate\(\) ---&gt;onBind\(\)---&gt;onunbind\(\)---&gt;onDestory\(\)

   bind 的方式开启服务，绑定服务，调用者挂了，服务也会跟着挂掉。 绑定者可以调用服务里面的方法。

### 10. WebView的使用

**为什么Webview加载较慢**

这是因为在客户端中，加载H5页面之前，需要先初始化WebView，在WebView完全初始化完成之前，后续的界面加载过程都是被阻塞的。

常见的方式

* 全局 WebView。
* 客户端代理页面请求。WebView 初始化完成后向客户端请求数据。
* asset 存放离线包。

**Android** 通过 **WebView** 调用 **JS** 代码:

1、通过 WebView 的 loadUrl\(\):

设置与 Js 交互的权限:webSettings.setJavaScriptEnabled\(true\)

设置允许 JS 弹窗:webSettings.setJavaScriptCanOpenWindowsAutomatically\(true\)

载入 JS 代码:mWebView.loadUrl\("file:///android\_asset/javascript.html"\)

webview 只是载体，内容的渲染需要使用 webviewChromClient 类去实现，通过设置 WebChromeClient 对象处理 JavaScript 的对话框。PS：JS 代码调用一定要在 onPageFinished\(\) 回调之后才能调用，否则不会调用。

2、通过 WebView 的 evaluateJavascript\(\):

只需要将第一种方法的 loadUrl\(\)换成 evaluateJavascript\(\)即可，通过onReceiveValue\(\)回调接收返回值。

**JS** 通过 **WebView** 调用 **Android** 代码:

1、通过 WebView 的 addJavascriptInterface\(\)进行对象映射:

2、通过 WebViewClient 的方法 shouldOverrideUrlLoading \(\)回调拦截 url:Android 通过 WebViewClient 的回调方法 shouldOverrideUrlLoading \(\)，拦截Url，解析该 url 的协议。

3、通过 WebChromeClient 的 onJsAlert\(\)、onJsConfirm\(\)、onJsPrompt\(\)方法回调拦截 JS 对话框 alert\(\)、confirm\(\)、prompt\(\) 消息:

### 12. View 的事件分发机制?滑动冲突怎么解决?

View 事件分发本质就是对 MotionEvent 事件分发的过程。即当一个 MotionEvent发生后，系统将这个点击事件传递到一个具体的 View 上。

**事件分发流程**：

1. 当屏幕被触摸，首先会通过硬件产生触摸事件传入内核，然后走到FrameWork层中SystemServer进程中的InputManagerService。
2. InputReader线程去读取这个事件，然后通过InputDispatcher分发这个事件。
3. 让我们回到`ViewRootImpl`的`setView`方法：这里创建了一个InputChannel

   ```java
   public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
       synchronized (this) {
         //创建InputChannel
         mInputChannel = new InputChannel();
         //通过Binder进入systemserver进程
         res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                     getHostVisibility(), mDisplay.getDisplayId(),
                     mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                     mAttachInfo.mOutsets, mInputChannel);
       }
   }
   ```

4. 事件到达应用端的主线程，会通过ViewRootImpl进行一系列InputStage来处理事件
5. 经过一系列分发，最终会执行到mView的`dispatchTouchEvent`方法，而这个mView就是DecorView

最后经过一系列事件处理到达ViewRootImpl的processPointerEvent方法.

```java
//ViewRootImpl.java
 private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;
            ...
            //mView分发Touch事件，mView就是DecorView
            boolean handled = mView.dispatchPointerEvent(event);
            ...
        }

//DecorView.java
    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            //分发Touch事件
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //cb其实就是对应的Activity
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }


//Activity.java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }

//PhoneWindow.java
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }

//DecorView.java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```

事件的分发流程就比较清楚了：

DecorView——&gt;Activity——&gt;PhoneWindow——&gt;DecorView——&gt;ViewGroup

但是这个流程确实有些奇怪，为什么绕来绕去的呢，光DecorView就走了两遍。

参考链接中的说法我还是比较认同的，主要原因就是解耦。

* ViewRootImpl并不知道有Activity这种东西存在，它只是持有了DecorView。所以先传给了DecorView，而DecorView知道有AC，所以传给了AC。
* Activity也不知道有DecorView，它只是持有PhoneWindow，所以这么一段调用链就形成了。

链路清楚之后就关心一下，事件分发过程中的三个核心方法吧：

**dispatchTouchEvent**:方法返回值为 true 表示事件被当前视图消费掉;返回为super.dispatchTouchEvent 表示继续分发该事件，返回为 false 表示交给父类的onTouchEvent 处理。

**onInterceptTouchEvent**:方法返回值为 true 表示拦截这个事件并交由自身的onTouchEvent 方法进行消费;返回 false 表示不拦截，需要继续传递给子视图。如果 return super.onInterceptTouchEvent\(ev\)， 事件拦截分两种情况:

* 1.如果该 View 存在子 View 且点击到了该子 View, 则不拦截, 继续分发 给子View处理
* 2.如果该 View 没有子 View 或者有子 View 但是没有点击中子 View\(此时ViewGroup相当于普通View）的onTouch方法响应。

**onTouchEvent**:方法返回值为 true 表示当前视图可以处理对应的事件;返回值为false，表示不处理，则会返回到父类中去进一步处理。如果 return super.onTouchEvent\(ev\)，事件处理分为两种情况:

* 1.如果该 View 是 clickable 或者 longclickable 的,则会返回 true, 表示消费了该事件与返回true一样
* 2.如果该 View 不是 clickable 或者 longclickable 的,则会返回 false, 表示不消费该事件，继续向上传递。

注意:在 Android 系统中，拥有事件传递处理能力的类有以下三种:

1. Activity:拥有分发和消费两个方法。
2. ViewGroup:拥有分发、拦截和消费三个方法。
3. View:拥有分发、消费两个方法。

**一些重要的结论**:

1. 事件传递优先级:onTouchListener.onTouch &gt; onTouchEvent &gt;onClickListener.onClick。
2. 正常情况下，一个事件序列只能被一个 View 拦截且消耗。因为一旦一个元 素拦截了此事件，那么同一个事件序列内的所有事件都会直接交给它处理\(即不 会再调用这个 View 的拦截方法去询问它是否要拦截了，而是把剩余的ACTION\_MOVE、ACTION\_DOWN 等事件直接交给它来处理\)。特例:通过将重 写 View 的 onTouchEvent 返回 false 可强行将事件转交给其他 View 处理
3. 如果 View 不消耗除 ACTION\_DOWN 以外的其他事件，那么这个点击事件会消失，此时父元素的 onTouchEvent 并不会被调用，并且当前 View 可以持续收到后续的事件，最终这些消失的点击事件会传递给 Activity 处理。
4. ViewGroup默认不拦截
5. View 的 onTouchEvent 默认都会消耗事件\(返回 true\)，除非它是不可点击的\(clickable 和 longClickable 同时为 false\)。View 的 longClickable 属性默认都为 false，clickable 属性要分情况，比如 Button 的 clickable 属性默认为 true
6. View 的 enable 属性不影响 onTouchEvent 的默认返回值。
7. 通过 **requestDisallowInterceptTouchEvent** 方法可以在子元素中干预父元素的事件分发过程，但是 ACTION\_DOWN 事件除外。

### 13. APK 的安装流程

![](http://solart.cc/images/Install_apk.png)

**第一步：拷贝文件到指定的目录：**

**第二步：解压缩apk，创建应用的数据目录**

**第三步：解析apk的AndroidManifest.xml文件**

**第四步：安装完毕，发送广播，显示快捷方式**

### 14. Android的打包流程

![](https://user-gold-cdn.xitu.io/2019/5/6/16a8c918da070cc0)

* 通过 AAPT 工具进行资源文件\(包括 AndroidManifest.xml、布局文件等）生成R.java
* 通过 AIDL 工具处理 AIDL 文件，生成相应的 Java 文件。
* 通过 Java Compiler 编译 R.java、Java 接口文件、Java 源文件，生成.class文件
* 通过 dex 命令，将.class 文件和第三方库中的.class 文件处理生成.dex
* 通过 ApkBuilder 工具将资源文件、DEX 文件打包生成 APK 文件。
* 通过 Jarsigner 工具，利用 KeyStore 对生成的 APK 文件进行签名。
* 如果是正式版的 APK，还会利用 ZipAlign 工具进行对齐处理，这样通过内存映射的访问Apk文件的速度会更快，并且可以减少内存占用。

### 15. Android中的签名

**什么是签名?**

在 Apk 中写入一个“指纹”。指纹写入以后，Apk 中有任何修改，都会导致这个指纹无效，Android 系统在安装 Apk 进行签名校验时就会不通过，从而保证了安全性。

**数字摘要**

对一个任意长度的数据，通过一个 Hash 算法计算后，都可以得到一个固定长度的二进制数据，这个数据就称为“摘要”。

**签名和校验的主要过程**

签名就是在摘要的基础上再进行一次加密，对摘要加密后的数据就可以当作数字签名

**签名过程**：

1、计算摘要:通过 Hash 算法提取出原始数据的摘要。

2、计算签名:再通过基于密钥\(私钥\)的非对称加密算法对提取出的摘要，加密后的数据就是签名

3、写入签名:将签名信息写入原始数据的签名区块内。

**校验过程**:

1、首先用同样的 Hash 算法从接收到的数据中提取出摘要。

2、解密签名:使用发送方的公钥对数字签名进行解密，解密出原始摘要。

3、比较摘要:如果解密后的数据和提取的摘要一致，则校验通过;否则不通过。

**V1和V2的区别**

V1：应该是通过ZIP条目进行验证，这样APK 签署后可进行许多修改 - 可以移动甚至重新压缩文件。

V2：验证压缩文件的所有字节，而不是单个 ZIP 条目，因此，在签名后无法再更改\(包括 zipalign\)。正因如此，现在在编译过程中，我们将压缩、调整和签署合并成一步完成。好处显而易见，更安全而且新的签名可缩短在设备上进行验证的时间（不需要费时地解压缩然后验证），从而加快应用安装速度。

### 16. Glide

**Glide 的三层缓存机制:**

Glide 缓存机制大致分为三层:内存缓存、弱引用缓存、磁盘缓存。

取的顺序是:内存、弱引用、磁盘。

存的顺序是:弱引用、内存、磁盘。

通过Glide来加载图片的时候会优先从内存中去，如果内存中没有，就从一个弱引用的map activeResources中找，如果也没有就从Lrucache。如果都没有，就会发起请求，请求成功后，资源放到 diskLrucache 和 activeResources 中。

**LruCache** 原理

内存缓存基于 LruCache 实现，磁盘缓存基于 DiskLruCache 实现。这两个类都基于 Lru 算法和 LinkedHashMap 来实现。

LRU 是 Least Recently Used 的缩写，最近最少使用算法。LruCache 的原理就是利用 LinkedHashMap 持有对象的强引用，按照 Lru 算法进行对象淘汰。具体说来假设我们从表尾访问数据，在表头删除数据，当访问的数据项在链表中存在时，则将该数据项移动到表尾，否则在表尾新建一个数据项。当链表容量超过一定阈值，则移除表头的数据。

Lrucache 需要移除一个缓存时，会调用 resource.recycle\(\)方法。注意到该方法将不用的Bitmap放到BitmapPool中。

**LinkedHashMap的原理**

LinkedHashMap 几乎和 HashMap 一样\(他是HashMap的子类\): 他的基本数据结构就是双向链表+HashMap，一个保证有序，一个保证查找复杂度为O1；

#### 自定义LRUCache注意的点

1. 使用哨兵模式
2. 线程安全问题 （CAS+Synchronized）
3. 使用ConcurrentHashMap（分段锁）替代hashMap

### 17. Leakcanary的原理

主要分为如下7个步骤：

* 1、RefWatcher.watch\(\)创建了一个KeyedWeakReference用于去观察对象。
* 2、然后，在后台线程中，它会检测引用是否被清除了，并且是否没有触发GC。
* 3、如果引用仍然没有被清除，那么它将会把堆栈信息保存在文件系统中的.hprof文件里。
* 4、HeapAnalyzerService被开启在一个独立的进程中，并且HeapAnalyzer使用了HAHA开源库解析了指定时刻的堆栈快照文件heap dump。
* 5、从heap dump中，HeapAnalyzer根据一个独特的引用key找到了KeyedWeakReference，并且定位了泄露的引用。
* 6、HeapAnalyzer为了确定是否有泄露，计算了到GC Roots的最短强引用路径，然后建立了导致泄露的链式引用。
* 7、这个结果被传回到app进程中的DisplayLeakService，然后一个泄露通知便展现出来了。

官方的原理简单来解释就是这样的：**在一个Activity执行完onDestroy\(\)之后，将它放入WeakReference中，然后将这个WeakReference类型的Activity对象与ReferenceQueque关联。这时再从ReferenceQueque中查看是否有没有该对象，如果没有，执行gc，再次查看，还是没有的话则判断发生内存泄露了。最后用HAHA这个开源库去分析dump之后的heap内存**

### 18. Android中的热修复，插件化

#### Android中的ClassLoader

![](https://upload-images.jianshu.io/upload_images/1437930-bb9d359f4c7e9935.png)

1. BootClassLoader：用于加载Framework层的class
2. **PathClassLoader**：用于加载已经安装到系统中的apk中的class
3. **DexClassLoader**：支持加载APk，dex和jar也可以从SD卡中加载。
4. BaseDexClassLoader: 是 PathClassLoader 和 DexClassLoader 的父类。

**什么是双亲委派机制？**

简单来说，当一个类被类加载器加载的时候，它会先判断自己的父级能不能能够加载该class文件，一直向上传导，向上传导过程中，如果有一个classLoader加载过，就不需要加载，一直到最上层都没有加载过，就开始从上到下继续传递。然后每一个层级，继续判断当前层级能否加载该类，如果能就加载这个class，否则继续向下，如果最下层都无法处理，则抛出classNotfound异常。

**Tinker如何实现热修复**

1. 原理就是将新的Dex插入到Runtime老的dex前面
2. 重点就是基于dexDiff的差分算法（双指针同步处理，new和old 两个apk文件）
3. 利用 PathClassLoader 和 DexClassLoader 去加载与 bug 类同名的类。
4. 资源的修复参考下面的资源修复，主要是利用5AssetsMangager
5. 同时注意异常监控的，做了很好的闭环，以及良好的注释

**资源修复**：

很多热修复框架的资源修复参考了 Instant Run 的资源修复的原理。AssetManager是加载资源的核心，并且AssetPath本身就支持多个资源

1、创建新的 AssetManager，通过反射调用 addAssetPath 方法加载外部的资源，这样新建的AssetManager就含有了外部的资源。

2、将 AssetManager 类型的 mAssets 字段的引用全部替换为新创建的新的AssetManager

**插件化的原理**

1. 占坑：预先在在AndroidManifest中注册Activity来占坑。
2. 通过hook的方式，拦截startActivity的方法，判断是启动插件还是内部的Activity
3. 将上一步中的参数，重新封装一个 subIntent 用来启动 StubActivity，并将前面得到的 TargetActivity保存到intent中
4. 这时候启动目标就变成了StubActivity，绕过AMS的校验
5. ActivityThread中重写mH中handleMessage对于launch activity的处理，将subActivity还原成原来的targetActivity，并且将intent还原。

**插件 Activity 的生命周期:** AMS 和 ActivityThread 之间的通信采用了 token 来对 Activity 进行标识，并且此后的 Activity 的生命周期处理也是根据 token 来对 Activity 进行标识，performLaunchActivity的r.token就是TargetActivity。所以TargetActivity是具有生命周期的。

**资源插件化:**

资源的插件化方案主要有两种:

* 合并资源方案，将插件的资源全部添加到宿主的 Resources 中，这种方案可以访问宿主的资源
* 构建插件资源方案，每个插件都构造出独立的 Resources，这种方案插件不能访问宿主的资源

### 19. RecyclerView 相关问题?

**RecyclerView是如何获取一屏中的高度的**

在RecyclerView添加item的时候，每个item添加进来，会将添加进入的item的高度累加，判断是否大于或等于当前屏幕的高度，当高度大于或等于当前高度的时候，即当前屏幕的数量count确定。

**RecyclerView的缓存机制**

ListView和RecyclerView缓存机制基本一致：缓存离开屏幕的Item，给进入的item使用。

RecyclerView具有四重缓存机制：

* **Scrap**：ListView 的Active View，就是屏幕内的缓存数据，可以直接拿来复用。
* **CacheView**：刚刚移出屏幕的view，默认大小是两个
* **ViewCacheExtension**：自定义缓存策略
* **RecycledViewPool**：当cache满了之后，会将viewholder从cache中转移到这里来

前两者是根据position来获取，第三个如果没有自定义的话，可以忽略，第四个是根据viewType来获取View的（并在方法中讲View重置），如果说缓存中没有，就会重新创建了。

另外需要注意的事，这里的缓存需要注意View的多种布局形式。并且缓存池是其中速度最慢的，因为从中取出的 ViewHolder 需要重新执行`onBindViewHolder()`重新为表项绑定数据。

1\). RecyclerView缓存RecyclerView.ViewHolder，抽象可理解为：View + ViewHolder\(避免每次createView时调用findViewById\) + flag\(标识状态\)；

2\). ListView缓存View。

**RecyclerView针对ListView的优化**

用法上差不多，RecyclerView 相比 ListView 在基础使用上的区别主要有如下几点:

* ViewHolder 的编写规范化了
* RecyclerView 复用 Item 的工作 Google 全帮你搞定，不再需要像ListView那样自己设置tag了
* RecyclerView 需要多出一步 LayoutManager 的设置工作
* RecyclerView 支持 线性布局、网格布局、瀑布流布局 三种
* RecyclerView支持单个Item的更新
* RecyclerView支持所有RecyclerView，公用一个RecyclerViewPool

ListView 的有点比较好，默认支持顶部和底部布局。

**局部刷新闪屏问题解决**

对于RecyclerView的Item Animator，有一个常见的坑就是“闪屏问题”。这个问题的原因是当调用`notifyItemChanged()`时，会调用DefaultItemAnimator的`animateChangeImpl()`执行change动画，该动画会使得Item的透明度从0变为1，从而造成闪屏。

**RecyclerView的局部刷新**

以RecyclerView中notifyItemRemoved\(1\)为例，最终会调用requestLayout\(\)，使整个RecyclerView重新绘制，过程为：onMeasure\(\)--&gt;onLayout\(\)--&gt;onDraw\(\)

其中，onLayout\(\)为重点，分为三步：

1. dispathLayoutStep1\(\)：记录RecyclerView刷新前列表项ItemView的各种信息，如Top,Left,Bottom,Right，用于动画的相关计算；
2. dispathLayoutStep2\(\)：真正测量布局大小，位置，核心函数为layoutChildren\(\)；
3. dispathLayoutStep3\(\)：计算布局前后各个ItemView的状态，如Remove，Add，Move，Update等，如有必要执行相应的动画.

需要指出，ListView和RecyclerView最大的区别在于数据源改变时的缓存的处理逻辑，ListView是"一锅端"，将所有的mActiveViews都移入了二级缓存mScrapViews，而RecyclerView则是更加灵活地对每个View修改标志位，区分是否重新bindView。

### 20. OKHTTP的实现原理

**okhttp的原理**

1. 发起请求，请求会进入两个队列中，运行时队列和等待队列
2. 请求进入运行中的队列，利用线程池，发起对应的网络请求
3. 每次请求之后，将符合条件的等待中的队列，取出放入运行时的队列中

okhttp的使用是建造者模式，而添加网络拦截器是用了责任链模式。

**OkHttp的拦截器**

在上面的代码中OkHttp通过各种拦截器处理请求。这里简单介绍下OkHttp的拦截器：

* 自定义拦截器：提供给用户的定制的拦截器。
* 失败和重定向拦截器（RetryAndFollowUpInterceptor）：请求在失败的时候重新开始已经请求重定向的拦截器。
* 桥接拦截器（BridgeInterceptor）：主要用来构造请求。
* 缓存拦截器（CacheInterceptor）：主要处理HTTP缓存。
* 连接拦截器（ConnectInterceptor）：主要处理HTTP链接。
* 网络请求拦截器（CallServerInterceptor）：负责发起网络请求。

**okhttp的缓存CacheInterceptor**

缓存拦截器主要是处理HTTP请求缓存的，通过缓存拦截器可以有效的使用缓存减少网络请求。

流程大致如下：

1. 根据请求（以Request为键值）取出缓存；
2. 验证缓存是否可用？可用，则直接返回缓存，否则进行下一步；
3. 继续执行下一个拦截器，直到但会结果；
4. 如果之前有缓存，则更新缓存，否则新增缓存。

**ConnectInterceptor**

ConnectInterceptor的作用就是建立一个与服务端的连接。

网络请求的的链接缓存在connectionPool中，

1、判断连接是否可用，不可用则从 ConnectionPool 获取连接，ConnectionPool，无连接，创建新连接，握手，放入 ConnectionPool。

2、它是一个 Deque，add 添加 Connection，使用线程池负责定时清理缓存。

3、使用连接复用省去了进行 TCP 和 TLS 握手的一个过程。

**CallServerInterceptor**

CallServerInterceptor是最后一个拦截器，理所当然这个拦截器负责向服务端发送数据。

发起请求并读取返回的数据。

## 3. Java相关问题的补充

### 1.谈谈对 java 多态的理解?

多态的定义:指允许不同类的对象对同一消息做出响应。即同一消息可以根据发 送对象的不同而采用多种不同的行为方式。

多态的三个必要条件:

* 继承父类。
* 重写父类的方法。
* 父类的引用指向子类对象。

可以实现动态加载技术，因为他消除了类型间的耦合关系。

### 2. Java中集合框架

![](https://pic3.zhimg.com/80/v2-76c3c04de2e8609c488fa0081fb99c26_1440w.png)_\*\*_

总体汇聚成上面的图

List：有序、可重复;索引查询速度快;插入、删除伴随数据移动，速度慢;

Set：无序，不可重复;

Queue：一种数据结构吧FIFO

剩下问题如Vector和ArrayList和Linkedlist的差别，基本就是数据结构的区别。

Vector 作为动态数组，是有能力在数组中的任何位置添加或者删除元素的。因此，Stack 继承了 Vector，Stack 也有这样的能力,

所以java官网推荐Stack使用是

```java
   Deque<Integer> stack = new ArrayDeque<Integer>();
```

### 3. Java反射

**反射的原理**

首先回顾下JVM加载Java文件的过程：

* 编译阶段，.java文件会被编译成.class文件，.class文件是一种二进制文件，内容是JVM能够识别的机器码。
* .class文件里面依次存储着类文件的各种信息，比如：版本号、类的名字、字段的描述和描述符、方法名称和描述、是不是public、类索引、字段表集合，方法集合等等数据。
* 然后，JVM中的类加载器会读取字节码文件，取出二进制数据，加载到内存中，并且解析.class文件的信息。
* 类加载器会获取类的二进制字节流，在内存中生成代表这个类的java.lang.Class对象。
* 最后会开始类的生命周期，比如连接、初始化等等。

而反射，就是去操作这个 java.lang.Class对象，这个对象中有整个类的结构，包括属性方法等等。

总结来说就是，.class是一种有顺序的结构文件,而Class对象就是对这种文件的一种表示，所以我们能从Class对象中获取关于类的所有信息，这就是反射的原理。

**反射可以修改final类型成员变量吗？** 

可以修改，包括对象和基本类型。

这两者还是有区别的：

* 对象可以直接通过打印日志比较，得知数据被修改。
* 基本类型（String）：由于内联函数的问题，JVM编译内联优化后会变成一个常量值。（如果需要获取改变后的值，可以通过反射中的Field.get\(Object obj\)方法获取）

**反射获取static静态变量**

我们知道，静态变量是在类的实例化之前就进行了初始化（类的初始化阶段），所以静态变量是跟着类本身走的，跟具体的对象无关，所以我们获取变量就不需要传入对象，直接传入null即可：

```java
public class User {
 public static String name;
}


field2 = clz.getDeclaredField("name");
field2.setAccessible(true);
//获取静态变量
Object getname=field2.get(null);
System.out.println("修改前"+getname);

//修改静态变量
field2.set(null, "xixi");
System.out.println("修改后"+User.name);
```

* Field.get\(null\) 可以获取静态变量。
* Field.set\(null,object\) 可以修改静态变量。

**怎么提升反射效率?**

1. 利用缓存，其实我不说大家也都知道，在平时项目中用到多次的对象也会进行缓存，谁也不会多次去创建。
2. 关闭安全检查：之前我们说过当遇到私有变量和方法的时候，会用到setAccessible\(true\)方法关闭安全检查。这个安全检查其实也是耗时的。
3. 利用第三方库：如ReflectASM这样的库，直接使用字节码来提供高性能的反射处理。

### 4. 泛型

**4.1 简单介绍一下 java 中的泛型，泛型擦除以及相关的概念**

泛型的基本原理就是Java中的类型擦除，类型擦除是指，使用泛型时加上的类型参数，在编译时会去掉，最后得到编译后的产物（字节码）是不包含泛型参数的。

**类型擦除也会带来一些问题**

* 类型擦除后，如何保证只使用指定类型，编译器做了优化，先检查，再编译
* 基本类型不能作为泛型实参，这就意味过程中会有装箱和拆箱的开销
* 泛型类型无法作为方法重载（类型擦除后，参数类型是object），但是可以重写。编译器会自己生成两个桥方法，方法的参数类型就是object，桥方法的内部实现就是调用我们自己的重写方法。
* 泛型无法作为真实的类型使用 比如数new一个对象等，这也是为什么Gson.fromJson需要传入Class的原因
* 静态方法无法引用类泛型参数

  泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数。

* 泛型在类型强转时会增加运行时开销

类型虽然被擦除了，但是被擦除的信息还是会以元素附加的签名信息，存储下来。

#### 4.2 方法分派

分派也就是java的多态的体现，他分为静态分派和动态分派

首先需要理解变量类型，比如：`Object obj = new String("");`

* 静态类型

  定义变量时声明的类型。这里的obj的静态类型就是Object，静态类型在编译期就确定了

* 实际类型

  obj的实际类型就是String，实际类型时在运行时确定的

然后我们在看静态分配和动态分配

* **静态分配 ——方法的重载**

  根据变量的静态类型匹配调用方法的过程，他的字节码中显示，他是按照静态分派选择匹配静态类型的重载方法，而不是按照实际类型

* **动态分配——方法的重写**

  根据变量的实际类型匹配调用方法的过程，就是动态分配。

  Java虚拟机在方法区中建立一个虚方法表（Virtual Method Table），通过使用方法表的索引来代替元数据查找以提高性能。虚方法表中存放着各个方法的实际入口地址（由于Java虚拟机自己建立并维护的方法表，所以没有必要使用符号引用，那不是跟自己过不去嘛），如果子类没有覆盖父类的方法，那么子类的虚方法表里面的地址入口与父类是一致的；如果重写父类的方法，那么子类的方法表的地址将会替换为子类实现版本的地址。

  方法表是在类加载的连接阶段（验证、准备、解析）进行初始化，准备了子类的初始化值后，虚拟机会把该类的虚方法表也进行初始化。

### 5. Java 的 char 是两个字节，是怎么存 Utf-8 的字符的?

1. java中的char不存utf-8它存储的是UTF-16
2. UTF-8是Unicode的一种实现方式，他是一种可变长度的编码方式，对于英文字母的UTF-8编码和ASCII码是相同的
3. UTF-16 具体定义了 Unicode 字符在计算机中存取方法，用两个字节来表示 Unicode 转化格式，这个是定长的表示方法，不论什么字符都能用两个字节
4. Unicode 是字符集（存储了所有的符号），他的作用是将字符转换成码点，也就是从人类的认知翻译成计算机可识别的代码

   人类认知的字符=&gt;字符集=&gt;计算机存储

5. java 中string 的length不是字符数而是char数组的长度，所以有时候会出现他的长度和字符数不想等的情况
6. 另外需要注意的是中文字符“中”获取UTF-16的字符长度为4个字节，比预计多出两个字节的，额外的两个字节是字符序

### 6. Java 的string 能有多长

1. 当String 存储在栈中
   * 受到字节码中CONSTANT\_UTF8\_info的限制，最终通过MUTF-8编码格式的字节数不能超过65535
   * 拉丁字符受到javac限制，最多是65534
   * 如果运行方法去较小，也会受到方法区大小的限制
2. 当String 存储在堆中
   * 受到虚拟机的指令限制，理论上最大值Integer.MAX\_VALUE
   * 实际可能小于理论上的值，虚拟机本身保留空间给头信息
   * 堆内存很小的话也会限制长度本身

### 7. Java的匿名内部类有哪些限制

1. 匿名内部类就是没有名字的内部类，只能继承一个父类或一个接口

   虚拟机中编译的匿名内部类其实是由名字的，比如外部类的路径$1

2. 匿名内部类分为静态和非静态
   * 静态匿名内部类

     静态匿名内部类不会持有外部引用

   * 非静态匿名内部类

     需要父类的外部实例来进行初始化
3. 匿名内部类的构造方法

   匿名内部类的构造方法是由编译器自动生成，当他是非静态的时候，默认的构造方法参数中会有外部的实例，以及自己外部类的实例

4. 只能捕获外部作用域内的final变量
5. java的中的匿名内部类不能被继承，kotlin 是可以的。

### 8. Java中对异常如何分类

1. Java 异常结构中定义有 Throwable 类。 Exception 和 Error 为其子类。
2. Error 是程序无法处理的错误，比如 OutOfMemoryError、StackOverflowError。发生时会终止程序
3. Exception 是程序本身可以处理的异常，这种异常分两大类运行时异常和非运行异常，程序应当去处理
4. 运行时异常都是 RuntimeException 类及其子类异常，如 NullPointerException等，这些异常不是编译时异常，如果遇到要尽量处理。

**NoClassDefFoundError** 和 **ClassNotFoundException** 有什么区别?

**ClassNotFoundException** 的产生原因主要是: Java 支持使用反射方式在运行时动态加载类，如调用反射的时候，但是在对应的路径没有找到这个类，就会抛出这个异常。

**NoClassDefFoundError** 产生的原因在于: 如果 JVM 或者 ClassLoader 实例尝试加载类的时候，找不到类的定义，编译时存在，但是运行时却找不到。

### 9. String 为什么要设计成不可变的?

String 本质上是个 final 的 char\[\]数组，不可变可以线程安全和字符串常量池的实现 。

### 10. Java 里的幂等性了解吗?

幂等性：对同一个系统，使用同样的条件，一次请求和重复的多次请求对系统资源的影响是一致的。

类似token的机制，或者订单id，通过一个标志表示订单状态，可以避免重复操作。

### 11. String，StringBuffer，StringBuilder 有哪些不同?

三者在执行速度方面的比较:StringBuilder &gt; StringBuffer &gt; String

String 每次变化一个值就会开辟一个新的内存空间

StringBuilder:线程非安全的

StringBuffer:线程安全的

单线程情况下使用StringBuilder，少的直接用string，大量数据处理需要线程安全则使用StringBuffer

### 12. 静态属性和静态方法是否可以被继承?是否可以被重写?以及原因?

结论:java 中静态属性和静态方法可以被继承，但是不可以被重写而是被隐藏。

1. 静态方法和属性是属于类的，调用的时候直接通过类名.方法名完成，不需要继承机制就能调用。
2. 多态之所以能够实现依赖于继承、接口和重写、重载\(继承和重写最为关键\)。有了继承和重写就可以实现父类的引用指向不同子类的对象。重写的功能是:"重写"后子类的优先级要高于父类的优先级，但是“隐藏”是没有这个优先级之分。
3. 静态属性、静态方法和非静态的属性都可以被继承和隐藏而不能被重写，因此不能实现多态，不能实现父类的引用可以指向不同子类的对象。非静态方法可以被继承和重写，因此可以实现多态。

### 13. Java 中对象的生命周期

在 Java 中，对象的生命周期包括以下几个阶段:

1. 创建阶段\(Created\)：JVM加载这个类
2. 应用阶段\(In Use\)：至少有一个强引用关系
3. 不可见阶段\(Invisible\)：程序本身不持有这个对象强引用
4. 不可达阶段\(Unreachable\)：该对象不再具有任何强引用了（和不可见相比，不可见可能仍然被其他程序锁持有）；
5. 回收阶段：垃圾收集器对对象空间进行回收，并进行空间再分配

### 14. java 中==和 equals 和 hashCode 的区别?

默认情况下也就是从超类 Object 继承而来的 equals 方法与‘==’是完全等价的，比较的都是对象的内存地址，但我们可以重写 equals 方法，使其按照我们的需求的方式进行比较，如 String 类重写了 equals 方法，使其比较的是字符的序列，而不再是内存地址。在 java 的集合中，判断两个对象是否相等的规则是:

1. 判断两个对象的 hashCode 是否相等。
2. 判断两个对象用 equals 运算是否相等。

此外还有一条，相同的对象hashCode一定相等，hashCode相同的对象不一样相等。

### 15. 类加载的过程，如Person p = new Person\(\)

1. 因为 new 用到了 Person.class，所以会先找到 Person.class 文件，并加载到内存中
2. 执行static方法，如果有的话，给Person类进行初始化操作
3. 在堆内存中开辟空间分配内存地址;
4. 在堆内存中建立对象的特有属性，并进行默认初始化;
5. 对属性进行显示初始化;
6. 对对象进行构造代码块初始化;
7. 对对象进行与之对应的构造函数进行初始化;
8. 将内存地址付给栈内存中的 p 变量。

### 16. 深拷贝和浅拷贝的区别

**浅拷贝**：浅拷贝是创建一个新对象，这个对象有着原始对象属性值的一份精确拷贝。如果属性是基本类型，拷贝的就是基本类型的值，如果属性是引用类型，拷贝的就是内存地址 ，所以**如果其中一个对象改变了这个地址，就会影响到另一个对象**。

**深拷贝**：深拷贝是将一个对象从内存中完整的拷贝一份出来,从堆内存中开辟一个新的区域存放新对象,且**修改新对象不会影响原对象**。

## 网络部分面试题

### 1. HTTP和HTTPS有啥区别？

* `HTTP`是超文本传输协议，信息是明文传输，`HTTPS`则是在HTTP层下加了一层具有安全性的SSL/TLS加密传输协议，要用到CA证书。
* `HTTP`是没有身份认证的，客户端无法知道对方的真实身份。`HTTPS`加入了CA证书，可以确认对方信息。
* `HTTP`默认端口为80，`HTTPS`为443。
* `HTTP`因为明文传输，容易被攻击或者流量劫持。

### 2. Http 1.0，1.1和2.0有哪些区别

#### 2.1 http 1.0和1.1的区别

**1. 缓存处理**，在HTTP1.0中主要使用header里的If-Modified-Since,Expires来做为缓存判断的标准，HTTP1.1则引入了更多的缓存控制策略例如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。

**2. 带宽优化及网络连接的使用**，HTTP1.0中，存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能，HTTP1.1则在请求头引入了range头域，它允许只请求资源的某个部分，即返回码是206（Partial Content），这样就方便了开发者自由的选择以便于充分利用带宽和连接。

**3. 错误通知的管理**，在HTTP1.1中新增了24个错误状态响应码，如409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。

**4. Host头处理**，在HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此，请求消息中的URL并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误（400 Bad Request）。

**5. 长连接**，HTTP 1.1支持长连接（PersistentConnection）和请求的流水线（Pipelining）处理，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟，在HTTP1.1中默认开启Connection： keep-alive，一定程度上弥补了HTTP1.0每次请求都要创建连接的缺点。

#### 2.2 SPDY：HTTP1.x的优化

2012 年 google 如一声惊雷提出了 SPDY 的方案，优化了 HTTP1.X 的请求延迟，解决了HTTP1.X的安全性：

1. **降低延迟**，针对HTTP高延迟的问题，SPDY优雅的采取了多路复用（multiplexing）。多路复用通过多个请求stream共享一个tcp连接的方式，解决了HOL blocking的问题，降低了延迟同时提高了带宽的利用率。
2. **请求优先级**（request prioritization）。多路复用带来一个新的问题是，在连接共享的基础之上有可能会导致关键请求被阻塞。SPDY允许给每个request设置优先级，这样重要的请求就会优先得到响应。比如浏览器加载首页，首页的html内容应该优先展示，之后才是各种静态资源文件，脚本文件等加载，这样可以保证用户能第一时间看到网页内容。
3. **header压缩。**前面提到HTTP1.x的header很多时候都是重复多余的。选择合适的压缩算法可以减小包的大小和数量。
4. **基于HTTPS的加密协议传输**，大大提高了传输数据的可靠性。
5. **服务端推送**（server push），采用了SPDY的网页，例如我的网页有一个sytle.css的请求，在客户端收到sytle.css数据的同时，服务端会将sytle.js的文件推送给客户端，当客户端再次尝试获取sytle.js时就可以直接从缓存中获取到，不用再发请求了。SPDY构成图：

![img](https://user-gold-cdn.xitu.io/2017/8/3/1706626a996d717c9d424646578813c2)

SPDY位于HTTP之下，TCP和SSL之上，这样可以轻松兼容老版本的HTTP协议\(将HTTP1.x的内容封装成一种新的frame格式\)，同时可以使用已有的SSL功能。

#### 2.3 HTTP2.0性能惊人：SPDY的升级版

**新的二进制格式**（Binary Format），HTTP1.x的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合。基于这种考虑HTTP2.0的协议解析决定采用二进制格式，实现方便且健壮。

**多路复用**（MultiPlexing），即连接共享，即每一个request都是是用作连接共享机制的。一个request对应一个id，这样一个连接上可以有多个request，每个连接的request可以随机的混杂在一起，接收方可以根据request的 id将request再归属到各自不同的服务端请求里面。

**header压缩**，如上文中所言，对前面提到过HTTP1.x的header带有大量信息，而且每次都要重复发送，HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一份header fields表，既避免了重复header的传输，又减小了需要传输的大小。

**服务端推送**（server push），同SPDY一样，HTTP2.0也具有server push功能。

### 3. Https 请求慢的解决办法

1. 不通过DNS解析，使用IP直连
2. 解决链路复用的问题，比如使用长连接

### 4. Http 的 request 和 response 的协议组成

#### 1. Request 组成

* 第一部分:请求行，第一行明了是 post 请求，以及 http1.1 版本。
* 第二部分:请求头部
* 第三部分:空行
* 第四部分:请求数据

#### 2. Response的组成

* 第一部分:状态行，由 HTTP 协议版本号， 状态码， 状态消息 三部分组成。
* 第二部分:消息报头，用来说明客户端要使用的一些附加信息
* 第三部分:空行，消息报头后面的空行是必须的
* 第四部分：响应正文

### 5. 说说http缓存

强制缓存:需要服务端参与判断是否继续使用缓存，当客户端第一次请求数据时，服务端返 回了缓存的过期时间\(Expires 与 Cache-Control\)，没有过期就可以继续使用缓存，否则则 不适用，无需再向服务端询问。

对比缓存:需要服务端参与判断是否继续使用缓存，当客户端第一次请求数据时，服务端会将缓存标识\(Last-Modified/If-Modified-Since 与 Etag/If-None-Match\)与数据一起返回给客户端，客户端将两者都备份到缓存中 ，再次请求数据时，客户端将上次备份的缓存 标识发送给服务端，服务端根据缓存标识进行判断， 如果返回 304，则表示通知客户端可以继续使用缓存。 强制缓存优先于对比缓存。

### 6. 长连接。

长连接一般是指TCP长连接，就是保持连接状态，并且可以复用的TCP（三次握手）建立的通道，避免连接解开连接的开销。

下面补充一个面试常见问题：

**socket断线重连，怎么实现，心跳机制又是如何实现的**？

套接字\(socket\)是通信的基石，是支持 TCP/IP 协议的网络通信的基本操作单元。TCP/IP协议提供了socket接口，应用层和传输层，通过socket区分来自不同应用程序或网络连接的通信，实现数据传输。

正常情况下Socket断开需要发一个fin包给服务端，服务端收到fin包之后，才会知道连接断开。

这时候需要重新发起socket连接。（通常情况下可以认为socket连接就是tcp连接）。

**如何确定长连接是否断开？**

1. 客户端向服务端发送心跳包
2. 服务端在一定时间内未收到心跳包，就反向向客户端发送心跳包，确认是否断开。

### 7. Https的加密

https 加密一般分为两种，对称加密和非对称加密（也可以加一个hash算法）。

* 对称加密：加密解密用的是同一个秘钥
* 非对称加密：加密解密用的是公钥和私钥，比如RSA加密算法

#### 7.1 客户端如何校验 CA 证书?

1. 首先浏览器读取证书中的证书所有者、有效期等信息进行校验，校验证书的网站域名是否与证书颁发的域名一致，校验证书是否在有效期内
2. 浏览器开始查找操作系统中已内置的受信任的证书发布机构CA，与服务器发来的证书中的颁发者CA比对，用于校验证书是否为合法机构颁发
3. 如果找到，那么浏览器就会从操作系统中取出颁发者CA 的公钥\(多数浏览器开发商发布版本时，会事先在内部植入常用认证机关的公开密钥\)，然后对服务器发来的证书里面的签名进行解密
4. 浏览器使用相同的hash算法计算出服务器发来的证书的hash值，将这个计算的hash值与证书中签名做对比
5. 对比结果一致，则证明服务器发来的证书合法，没有被冒充

#### 7.2 HTTPS 中的 SSL 握手建立过程（https的连接过程）

1. 客户端发起Https的请求
2. 服务端发送证书（公钥）给客户端
3. 客户端验证证书和公钥的有效性，如果有效，则生成共享密钥并使用公开密钥加密发送到服务器端
4. 服务器端使用私有密钥解密数据，并使用收到的共享密钥加密数据，发送到客户端
5. 客户端验证服务端的握手信息，完成握手。

#### 7.4 **请给我讲解一下数字签名，为什么真实可靠**

数字签名，其实也是一种非对称加密的用法。

它的使用方法是：

A使用私钥对数据的哈希值进行加密，这个加密后的密文就叫做签名，然后将这个密文和数据本身传输给B。

B拿到后，签名用公钥解密出来，然后和传过来数据的哈希值做比较，如果一样，就说明这个签名确实是A签的，而且只有A才可以签，因为只有A有私钥。

反应实际情况就是：

服务器端将数据，也就是我们要传的数据（公钥），用另外的私钥签名数据的哈希值，然后和数据（公钥）一起传过去。然后客户端用另外的公钥对签名解密，如果解密数据和数据（公钥）的哈希值一致，就能证明来源正确，不是被伪造的。

* 来源可靠。数字签名只能拥有私钥的一方才能签名，所以它的存在就保证了这个数据的来源是正确的
* 数据可靠。hash值是固定的，如果签名解密的数据和本身的数据哈希值一致，说明数据是未被修改的。

**证书链安全机制**

服务器会拿自己的公钥以及服务器的一些信息传给CA，然后CA会返回给服务器一个数字证书，这个证书里面包括：

* 服务器的公钥
* 签名算法
* 服务器的信息，包括主机名等。
* CA自己的私钥对这个证书的签名

### 8. 为什么 tcp 要经过三次握手，四次挥手?

**三次握手**

SYN，同步序列编号，是TCP/IP建立连接时使用的握手信号，如果这个值为1就代表是连接消息。ACK，确认标志，如果这个值为1就代表是确认消息。Seq，数据包序号，是发送数据的一个顺序编号。Ack Number，确认数字号，是接收数据的一个顺序编号。

![](https://user-gold-cdn.xitu.io/2019/10/8/16da9fd28a45bd19)

* 创建套接字Socket，服务器会在启动的时候就创建好，客户端是在需要访问服务器的时候创建套接字。
* 然后发起连接操作，其实就是Socket的connect方法。
* 这时候客户端会生成一个TCP数据包。这个数据包的TCP头部有几个重要信息：SYN、ACK、Seq、Ack。
* 所以客户端就生成了这样一个数据包，其中头部信息的控制位SYN设置为1，代表连接。SEQ设置一个随机数，代表初始序号，比如100。
* 然后服务器端收到这个消息，知道了客户端是要来连接的（SYN=1），知道了传输数据的初始序号（SEQ=100）。
* 服务器端也要生成一个数据包发送给客户端，这个数据包的TCP头部会包含：表示我也要连接你的SYN（SYN=1），我已经收到了你的上个数据包的确认号ACK=1（Ack=Seq+1=101），以及服务器端随机生成的一个序号Seq（比如Seq=200）。
* 最后客户端收到这个消息后，表示客户端到服务器的连接是无误了，然后再发送一个数据包表示也确认收到了服务器发来的数据包，这个数据包的头部就主要就是一个ACK=1（Ack=Seq+1=201）。
* 至此，连接成功，三次握手结束，后面数据就会正常传输，并且每次都要带上TCP头部中的Seq和Ack。

**为什么需要三次握手**

这里既要确认客户端的发送和接受能力，也要确定服务端的发送接受能力，至少三次。

**四次挥手：**

![](https://user-gold-cdn.xitu.io/2019/10/8/16da9fd28b49f652)

和连接阶段一样，TCP头部也有一个专门用作关闭连接的值叫做FIN。

* 客户端准备关闭连接，会发送一个TCP数据包，头部信息中包括（FIN=1代表要断开连接）。
* 服务器端收到消息，回复一个数据包给客户端，头部信息中包括Ack确认号。但是此时服务器端的正常业务可能没有完成，还要处理下数据，收个尾。
* 客户端收到消息。
* 服务器继续处理数据。
* 服务器处理数据完毕，准备关闭连接，会发送一个TCP数据包给客户端，头部信息中包括（FIN=1代表要断开连接）。
* 客户端端收到消息，回复一个数据包给服务器端，头部信息中包括Ack确认号。
* 服务器收到消息，到此服务器端完成连接关闭工作。
* 客户端经过一段时间（2MSL），自动进入关闭状态，到此客户端完成连接关闭工作。

​

### 9. TCP 可靠传输原理实现\(滑动窗口\)

TCP协议作为一个可靠的面向流的传输协议，其可靠性和流量控制由滑动窗口协议保证，而拥塞控制则由控制窗口结合一系列的控制算法实现。

* 确认和重传:接收方收到报文后就会进行确认，发送方一段时间没有收到确认就会重传。 数据校验。
* 数据合理分片与排序：TCP 会对数据进行分片，接收方会缓存为按序到达的数据，重新排序 后再提交给应用层。
* 流程控制:当接收方来不及接收发送的数据时，则会提示发送方降低发送的速度，防止包丢失。
* 拥塞控制:当网络发生拥塞时，减少数据的发送

### 10. Tcp 和 Udp 的区别?

比如TCP的使用场景：浏览网页，UDP的使用场景，游戏。

|  | UDP | TCP |
| :--- | :--- | :--- |
| 是否基于连接 | 无连接 | 面向连接 |
| 是否可靠 | 不可靠传输，不使用流量控制和拥塞控制 | 可靠传输，使用流量控制和拥塞控制 |
| 连接对象个数 | 支持一对一，一对多，多对一和多对多交互通信 | 只能是一对一通信 |
| 传输方式 | 面向报文 | 面向字节流 |
| 是否保证顺序 | 不保证 | 保证数据顺序 |
| 适用场景 | 适用于实时应用（IP电话、视频会议、直播等） | 适用于要求可靠传输的应用，例如文件传输 |

**如何设计在 UDP 上层保证 UDP 的可靠性传输?**

* 添加 seq/ack 机制，确保数据发送到对端
* 添加发送和接收缓冲区，主要是用户超时重传。 
* 添加超时重传机制。

### 10. 浏览器输入地址到返回结果发生了什么?

**客户端：**

1. 在浏览器输入网址。
2. 浏览器解析网址，并生成http请求消息。
3. 浏览器调用系统解析器，发送消息到DNS服务器查询域名对应的ip。
4. 拿到ip后，和请求消息一起交给操作系统协议栈的TCP模块。
5. 将数据分成一个个数据包，并加上TCP报头形成TCP数据包。
6. TCP报头包括发送方端口号、接收方端口号、数据包的序号、ACK号。
7. 然后将TCP消息交给IP模块。
8. IP模块会添加IP头部和MAC头部。
9. IP头部包括IP地址，为IP模块使用，MAC头部包括MAC地址，为数据链路层使用。
10. IP模块会把整个消息包交给网络硬件，也就是数据链路层，比如以太网，WIFI等。
11. 然后网卡会将这些包转换成电信号或者在光信号，通过网线或者光纤发送出去，再由路由器等转发设备送达接收方。

**服务器端：**

1. 数据包到达服务器的数据链路层，比如以太网，然后会将其转换为数据包（数字信号）交给IP模块。
2. IP模块会将MAC头部和IP头部后面的内容，也就是TCP数据包发送给TCP模块。

   TCP模块会解析TCP头信息，然后和客户端沟通表示收到这个数据包了。

3. TCP模块在收到消息的所有数据包之后，就会封装好消息，生成相应报文发给应用层，也就是HTTP层。
4. HTTP层收到消息，比如是HTML数据，就会解析这个HTML数据，最终绘制到浏览器页面上。

### 11.**socket和WebSocket**

虽然这两个货名字类似，但其实不是一个层级的概念。

* socket，套接字。上文说过了，在TCP建立连接的过程中，是调用了Socket的相关API，建立了这个连接通道。所以它只是一个接口，一个类。
* WebSocket，是和HTTP同等级，属于应用层协议。它是为了解决长时间通信的问题，由HTML5规范引出，是一种建立在TCP协议基础上的全双工通信的协议，同样下层也需要TCP建立连接,所以也需要socket。

> 科普：WebSocket在TCP连接建立后，还要通过Http进行一次握手，也就是通过Http发送一条GET请求消息给服务器，告诉服务器我要建立WebSocket连接了，你准备好哦，具体做法就是在头部信息中添加相关参数。然后服务器响应我知道了，并且将连接协议改成WebSocket，开始建立长连接。

如果硬要说这两者有关系，那就是WebSocket协议也用到了TCP连接，而TCP连接用到了Socket的API。

## 其他

### 1. 项目经历

主要讲讲工作中实际遇到的，拿下载部分功能举例吧：

1. 确定核心需求，apk下载安装功能
2. 动态权限获取
3. 考虑多线程同时下载的问题（监听网络状态判断是线性还是并发策略）
4. 是否需要对任务队列缓存池进行优化（动态扩容策略，拒绝策略）
5. 断点续传
6. 状态调整：新增游戏详情页，判断app是否安装，更新数据库数据
7. 下载进程 Message msg = queue.next\(\); // might block

   ```text
    try {
        msg.target.dispatchMessage(msg);
    } 

    msg.recycleUnchecked();
   ```

   } }

   \`\`\`

在loop的方法中进行消息分发，核心就是`msg.target.dispatchMessage(msg);`这里的的target是在使用Hanlder发送消息的时候，会设置msg.target = this，所以target就是当初把消息加到消息队列的那个Handler。

**衍生问题：Handler 的 post\(Runnable\) 与 sendMessage 有什么区别**

差别在dispatchMessage的方法中

```java
public void dispatchMessage(@NonNull Message msg) {
  //优先判断是否是runnable对象
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

private static void handleCallback(Message message) {
    message.callback.run();
}
```

1. post方法是给Message设置了一个callback
2. 在消息的处理过程中， 优先分发给msg.callback，如果前者为空则返给Handler.Callback.handleMessage，如果后者为空则直接返回不处理

所以两者的区别是交给谁处理上。

**1.3.2 recycleUnchecked方法**

看上面的代码需要注意的一点就是在分发消息（dispatchMessage）之后，还调用了一个recycleUnchecked方法。

recycleUnchecked这个方法释放了消息所有资源，然后将当前的空消息插入到sPool（Message的链表，默认长度为50）头部，一些博客中讲得消息复用池就是这个。

Message的复用，可以看下代码，就是取出复用池的头部节点

```java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

**衍生问题：Looper.loop方法是死循环，为什么不会卡死（ANR）**

1. 主线程本身需要及时处理各种响应和交互（如View的绘制，Activity的生命周期等问题）,所以需要一个死循环来实时处理问题
2. 看代码，Looper本身只做消息的分发，操作时间很短，真正产生ANR的原因往往是，接受到对应消息后的处理引起的
3. 当没有消息的时候，会阻塞在loop的queue.next\(\)中的nativePollOnce\(\)方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生。所以死循环也不会特别消耗CPU资源。

#### 1.4 常见问题补充

**1.4.1 IdleHandler是啥？有什么使用场景？**

上面提到过，nativePollOnce是阻塞方法，其实在阻塞之前会有检查是否存在IdleHandler，如果存在就执行他的queueIdle方法。

也就是说，当前如果没有消息需要处理的时候，空闲状态可以处理一些空闲任务。常见的可以是作为启动优化，使用不当的话，可能导致异常情况。

**1.4.2 HandlerThread是啥？有什么使用场景？**

```java
public class HandlerThread extends Thread {
@Override
public void run() {
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
}
```

HandlerThread就是一个封装了Looper的Thread类。目的就是为了我们能更方便的使用handler，通过加锁保证获取当前Looper对象成功后，通知其他线程。

**1.4.3 IntentService是啥？有什么使用场景？**

1. 继承自Service
2. 并且内部维护了一个HandlerThread，也就是有完整的Looper在运行
3. 还维护了一个子线程的ServiceHandler
4. 启动Service后，会通过Handler执行onHandleIntent方法。
5. 完成任务后，会自动执行stopSelf停止当前Service

总结：他是一个可以在子线程进行耗时任务，并且在任务执行后自动停止的Service。

**1.4.4 BlockCanary使用过吗？说说原理**

BlockCanary是一个用来检测应用卡顿耗时的三方库。

View的绘制也是通过Handler来执行的，所以如果能知道每次Handler处理消息的时间，就能知道每次绘制的耗时了？那Handler消息的处理时间怎么获取呢？

在Looper.loop\(\)中loop方法内有一个Printer类，在dispatchMessage处理消息的前后分别打印了两次日志，把这个loop替换成我们自己的就行了。

**1.4.5 Handler内存泄露的原因是什么？**

`Handler`导致内存泄漏一般发生在发送延迟消息的时候，当`Activity`关闭之后，延迟消息还没发出，那么主线程中的`MessageQueue`就会持有这个消息的引用，而这个消息是持有`Handler`的引用，而`handler`作为匿名内部类持有了`Activity`的引用，所以就有了以下的一条引用链。

所以根本原因就是主线程永远不会被回收，导致了Activity一直不能回收，而Handler是引用关系中的一环。

### 2. Activity启动相关

![](https://user-gold-cdn.xitu.io/2020/7/12/17343b491419f3e9)

几个重点类的解释：

* **Instrumentation**: 监控应用与系统相关的交互行为。
* **ATMS**:组件管理调度中心，什么都不干，但是什么都管。
* **ActivityStarter**:Activity 启动的控制器，处理 Intent 与 Flag 对 Activity 启动的影响
* **ActivityStackSupervisior**:这个类的作用你从它的名字就可以看出来，它用来管理任务栈
* **ActivityStack**:用来管理任务栈里的 Activity。

主要还是上面的流程，这里做个答题总结：

1. 首先需要明确，进程间的关系，从起始App进程到SystemServer进程，最后到目标App进程
2. 然后逐个分析，首先在起始App进程中，Activity到Instrumentation
3. 进入到SystemServer进程中，这里强调一下Android 10 中引入的ATMS，然后再ATMS的startActivity方法，通过ActivityStarter最后执行到ActivityStack中**resumeTopActivityInnerLocked**的关键方法，这个方法里面主要做了几件事
   1. 暂停了上一个Activity
   2. 判断Activity是否启动（next.attachedToProcess（））
   3. 如果启动的Activity，直接安排Resume，如果没有启动就是进入启动Activity的流程，步骤4
4. 接着事件会来到**ActivityStackSupervisor**中，startSpecificActivityLocked，这个方法开面会进行判断，目标Activity的进程是否存在，不存在的话，需要通过Zygote去Fork一个进程。
5. 继续调用方法realStartActivityLocked，通过ClientTransaction将对应的事务发送到ApplicationThread，这时候就来到了目标App进程
6. 这里会通过的mH（handler的机制）将消息发送到主线程，然后ActivityThread通过对应的事件启动对应的Activity生命周期onCreate，onResume（handleLaunchActivity--》performLaunchActivity，performResumeActivity）等。

**关于Activity生命周期这里需要补充**

**performLaunchActivity**主要完成以下事情：

1. 从ActivityClientRecord获取待启动的Activity的组件信息
2. 创建一个ContextImpl对象
3. 通过mInstrumentation.newActivity方法使用类加载器创建activity实例
4. 如果没有Application的话，这里会通过LoadedApk的makeApplication方法创建一个Application对象，内部也是通过mInstrumentation使用类加载器，创建后就调用了instrumentation.callApplicationOnCreate方法，也就是Application的onCreate方法。
5. 并通过activity.attach方法对重要数据初始化，为Activity创建上下文。
6. 调用Activity的onCreate方法，是通过 mInstrumentation.callActivityOnCreate方法完成。

接下来看**handleResumeActivity**，他主要做了两件事：

1. 调用生命周期：通过performResumeActivity方法，内部调用生命周期onStart、onResume方法
2. 设置视图可见：通过activity.makeVisible方法，添加[window](https://blog.csdn.net/hfy8971613/article/details/103241153)、设置可见。（所以视图的真正可见是在onResume方法之后）

另外一点，Activity视图渲染到Window后，会设置window焦点变化，先走到DecorView的onWindowFocusChanged方法，最后是到Activity的onWindowFocusChanged方法，表示首帧绘制完成，此时Activity可交互。

至此Activity的启动全部流程结束。

**Zygote是如何fork进程的？**

补充一下创建进程的流程：

1. 通知Zygote需要创建一个进程
2. Zygote通过fork创建了一个进程
3. 在新建的进程中创建Binder线程池（此进程就支持了Binder IPC）
4. **最终是通过反射获取到了ActivityThread类并执行了main方法**
5. 在ActivityThread的main方法中，
   1. 创建ActivitThread的实例
   2. 创建消息循环机制
6. 调用attach方法，最后到AMS中
   1. 创建并绑定Application
   2. ApplicationThread关联到AMS
7. 通过ATMS启动根Activity

### 3. View绘制相关

先看下大概结构：

![](https://upload-images.jianshu.io/upload_images/2397836-f1f6a200704884a2.png)

#### 3.1 **DecorView** 被加载到 **Window** 中

Activity创建的时候，会先调用performLaunchActivity，然后是Activity的onCreate方法，这时候会创建**DecorView**和**Activity**。在handleResumeActivity方法中会通过WindowManager的addView方法将DecorView添加到Window中。

WindowManager最终会调用WindowManagerGlobal.addView，在这个方法里面会创建ViewRootImpl，ViewRootImpl是连接WindowManager和DecorView的纽带，View的操作都是由ViewRootImpl控制的。

ViewRootImpl的构造方法内创建了WindowSession\(Binder\)，通过它与WindowManagerService进行通信。

**window的机制**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/768cd62019764d129e24d432792e3638~tplv-k3u1fbpfcp-zoom-1.image)

我们知道WMS是window的最终管理者，在WMS中为每一个应用持有一个session，关于session前面我们讲过，每个应用都是全局单例，负责和WMS通信的binder对象。WMS为每个window都建立了一个windowStatus对象，同一个应用的Window使用同一个session进行通信，每一个windowStatus对应一个viewRootImpl，WMS通过viewRootImpl来控制view。

**Android中surface机制**

上面讲到的了widow的机制，实际上

#### 3.2 **View的开始绘制**

Activity启动的状况下的话，视图**ViewRootImpl.requestLayout** 就触发了第一次 View 的绘制。

View绘制会从根视图 ViewRoot 的 **performTraversals**\(\)方法开始，从上到下遍历整个ViewTree视图树，每个 View 控件负责绘制自己，而 ViewGroup 还需要负责通知自己的子View 进行绘制操作。

performTraversals内部就是就是熟悉的 performMeasure -&gt; performLayout -&gt; performDraw 三个流程了。

1. **理解MeasureSpec**

一个MeasureSpec（View的静态内部类）封装了从父容器传递给子容器的布局要求,这个MeasureSpec 封装的是父容器传递给子容器的布局要求，而不是父容器对子容器的布局要求，MeasureSpec是一个大小跟模式的组合值,MeasureSpec中的值是一个整型（32位）将size和mode打包成一个Int型，其中高两位是mode，后面30位存的是size，是为了减少对象的分配开支。

MeasureSpec一共有三种模式

* **UNSPECIFIED** : 父容器对于子容器没有任何限制,子容器想要多大就多大
* **EXACTLY**: 父容器已经为子容器设置了尺寸,子容器应当服从这些边界,不论子容器想要多大的空间。
* **AT\_MOST**：子容器可以是声明大小内的任意大小（有点像match\_parent）

**View 绘制流程之 Measure**

* 首先，在 ViewGroup 中的 measureChildren\(\)方法中会遍历测量所有子View，当View的状态为Gone时不测量
* 然后，测量某个指定的 View 时，根据父容器的 MeasureSpec 和子 View的LayoutParams来计算子View的MeasureSpec
* 最后，将计算出的 MeasureSpec 传入 View 的 measure 方法，这里ViewGroup没有定义具体的测量过程，其测量过程需要子类去实现

**getSuggestMinimumWidth** 分析

如果 View 没有设置背景，那么返回 android:minWidth 这个属性所指定的值，这个值可以为 0;如果 View 设置了背景，则返回 android:minWidth 和背景的最小宽度这两者中的最大值。

自定义 **View** 时手动处理 **wrap\_content** 时的情形

直接继承 View 的控件需要重写 onMeasure 方法并设置 wrap\_content 时的自身大小，否则在布局中使用 wrap\_content 就相当于使用 match\_parent。

在 **Activity** 中获取某个 **View** 的宽高

由于 View 的 measure 过程和 Activity 的生命周期方法不是同步执行的，如果 View还没有测量完毕，那么获得的宽/高就是 0。所以在 onCreate、onStart、onResume中均无法正确得到某个 View 的宽高信息。解决方式如下:

* Activity/View\#onWindowFocusChanged:此时 View 已经初始化完毕，当Activity的窗口获得和失去焦点时均会调用一次。
* view.post\(runnable\): 通过 post 可以将一个 runnable 投递到消息队列的尾部，等Looper调用runnable的时候View已经初始化好了。
* ViewTreeObserver\#addOnGlobalLayoutListener:当 View 树的状态发生改变或者ViewTree发生可变性变化的时候，会回调这个方法

**3. View 的绘制流程之 Layout**

首先，会通过 **setFrame** 方法来设定 View 的四个顶点的位置，即 View 在父容器中的位置。然后，会执行到 onLayout 空方法，子类如果是 ViewGroup 类型，则重写这个方法，实现 ViewGroup 中所有 View 控件布局流程。

**4. View绘制流程值Draw**

绘制基本上可以分为六个步骤:

* 首先绘制 View 的背景;
* 如果需要的话，保持 canvas 的图层，为 fading 做准备;
* 绘制View
* 绘制View的子View
* 如果需要的话，绘制View的fading边缘并恢复图层
* 绘制View的装饰\(例如滚动条等等\)

**LinearLayout和RelativeLayout的性能对比**

（1）RelativeLayout慢于LinearLayout是因为它会让子View调用2次measure过程（横向和纵向各有一次），而LinearLayout只需一次，但是有weight属性存在时，LinearLayout也需要两次measure。

**硬件加速绘制和软件绘制的差别**

硬件加速绘制与软件绘制整个流程差异非常大，最核心就是我们通过 GPU 完成 Graphic Buffer 的内容绘制。此外硬件绘制还引入了一个 DisplayList 的概念，每个 View 内部都有一个 DisplayList，当某个 View 需要重绘时，将它标记为 Dirty。

当需要重绘时，仅仅只需要重绘一个 View 的 DisplayList，而不是像软件绘制那样需要向上递归。这样可以大大减少绘图的操作数量，因而提高了渲染效率。

### 4. Java的内存模型和GC算法

#### 4.1 JVM内存模型简述

![](https://pic1.zhimg.com/80/v2-354d31865d1fb3362f5a1ca938f9a770_1440w.jpg)

从上图中可以看到，JVM主要包括4个部分；

1. 类加载器\(ClassLoader\):在 JVM 启动时或者在类运行将需要的 class 加载到JVM中
2. 执行引擎:负责执行 class 文件中包含的字节码指令;
3. 内存区\(也叫运行时数据区\):是在 JVM 运行的时候操作所分配的内存区。内存区又被分为5个小部分：
   1. 方法区\(MethodArea\):用于存储类结构信息的地方，包括常量池、静态常量、构造函数等。
   2. java 堆\(Heap\):存储 java 实例或者对象的地方。这块是 GC 的主要区域。从存储的内容我们可以很容易知道，方法和堆是被所有 java 线程共享的。
   3. java 栈\(Stack\):java 栈总是和线程关联在一起，每当创一个线程时，JVM 就会为这个线程创建一个对应的 java 栈在这个 java 栈中,其中又会包含多个栈帧，每运行一个方法就建一个栈帧，用于存储局部变量表、操作栈、方法返回等。
   4. 程序计数器\(PCRegister\):用于保存当前线程执行的内存地址。
   5. 本地方法栈\(Native MethodStack\):和 java 栈的作用差不多，只不过是为 JVM 使用到 native 方法服务的。
4. 本地方法接口:主要是调用 C 或 C++实现的本地方法及回调结果。

#### 4.2 描述一下 GC 的原理和回收策略?

垃圾回收一般是找到垃圾，然后回收垃圾。

**找到垃圾的方式**：

* **引用计数算法**（Reachability Counting）是通过在对象头中分配一个空间来保存该对象被引用的次数（Reference Count）。如果该对象被其它对象引用，则它的引用计数加1，如果删除对该对象的引用，那么它的引用计数就减1，当该对象的引用计数为0时，那么该对象就会被回收。
* **可达性分析算法**（Reachability Analysis）的基本思路是，通过一些被称为引用链（GC Roots）的对象作为起点，从这些节点开始向下搜索，搜索走过的路径被称为（Reference Chain\)，当一个对象到 GC Roots 没有任何引用链相连时（即从 GC Roots 节点到该节点不可达），则证明该对象是不可用的。

**2. 回收垃圾组合拳**

1. **标记清除算法**（Mark-Sweep）是最基础的一种垃圾回收算法，它分为2部分，先把内存区域中的这些对象进行标记，哪些属于可回收标记出来，然后把这些垃圾拎出来清理掉。（回收后导致内存碎片化，不连续）
2. **复制算法**（Copying）是在标记清除算法上演化而来，解决标记清除算法的内存碎片问题。它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。保证了内存的连续可用，内存分配时也就不用考虑内存碎片等复杂情况，逻辑清晰，运行高效。（代价太高，意味着只能使用一半的内存）
3. **标记整理算法**（Mark-Compact）标记过程仍然与标记 --- 清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，再清理掉端边界以外的内存区域。

**内存模型与回收策略**

内存主要被分为三块：**新生代（Youn Generation）、旧生代（Old Generation）、持久代（Permanent Generation）**。三代的特点不同，造就了他们使用的GC算法不同，新生代适合生命周期较短，快速创建和销毁的对象，旧生代适合生命周期较长的对象，持久代在Sun Hotpot虚拟机中就是指方法区（有些JVM根本就没有持久代这一说法）。

**新生代（Youn Generation）**：大致分为Eden区和Survivor区，Survivor区又分为大小相同的两部分：FromSpace和ToSpace。

* **Eden 区**

  大多数情况下，对象会在新生代 Eden 区中进行分配，当 Eden 区没有足够空间进行分配时，虚拟机会发起一次 Minor GC，Minor GC 相比 Major GC 更频繁，回收速度也更快。

  通过 Minor GC 之后，Eden 会被清空，Eden 区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到 Survivor 区。

* **Survivor 区**

  Survivor 区相当于是 Eden 区和 Old 区的一个缓冲，类似于我们交通灯中的黄灯。Survivor 又分为2个区，一个是 From 区，一个是 To 区。每次执行 Minor GC，会将 Eden 区和 From 存活的对象放到 Survivor 的 To 区（如果 To 区不够，则直接进入 Old 区）。

**为啥需要将新生代分为两个区？**

Survivor 的存在意义就是减少被送到老年代的对象，进而减少 Major GC 的发生。Survivor 的预筛选保证，只有经历16次 Minor GC 还能在新生代中存活的对象，才会被送到老年代

**为啥Survivor需要俩个？**

设置两个 Survivor 区最大的好处就是解决内存碎片化。可以参考上面的复制算法。

**旧生代（Old Generation）**：老年代占据着2/3的堆内存空间，只有在 Major GC 的时候才会进行清理，每次 GC 都会触发“Stop-The-World”，内存越大，stop的时间也越长，所以老年代这里采用的是标记 --- 整理算法。

此外还要注意几个点：

1. **大对象**：大对象指需要大量连续内存空间的对象，这部分对象不管是不是“朝生夕死”，都会直接进到老年代。避免发生大量的复制
2. **长期存活对象**：虚拟机给每个对象定义了一个对象年龄（Age）计数器。经历一次Minor GC年龄+1，到15岁后就进老年代了。
3. **动态对象年龄**：虚拟机并不重视要求对象年龄必须到15岁，才会放入老年区，如果 Survivor 空间中相同年龄所有对象大小的总合大于 Survivor 空间的一半，年龄大于等于该年龄的对象就可以直接进去老年区，无需等你“成年”。

**强引用置为 null，会不会被回收?**

不会立即释放对象占用的内存。 如果对象的引用被置为 null，只是断开了当前线程栈帧中对该对象的引用关系，而 垃圾收集器是运行在后台的线程，只有当用户线程运行到安全点\(safe point\)或者安全区域才会扫描对象引用关系，扫描到对象没有被引用则会标记对象，这时候仍然不会立即释放该对象内存，因为有些对象是可恢复的\(在 finalize 方法中恢复引用 \)。只有确定了对象无法恢复引用的时候才会清除对象内存。

### 5. HashMap、ConcurrentHashMap、hash\(\)相关原理解析?

#### 5.1 HashMap相关

在JDK1.7中，HashMap底层是用数组+链表的形式，在JDK1.8的时候就改成了数组+链表和红黑树了。

当链表长度大于8的时候，链表会转化为红黑树。

同一个key对应多个值是hash冲突导致的，HashMap是线程不安全的，在并发场景下链表可能变成形成环状链表。

#### 5.2 ConcurrentHashMap 线程安全的map

由于HashMap是线程不安全，相对应的就会有线程安全的ConcurrentHashMap，下面简单介绍下ConcurrentHashMap：

主要是1.7版本和1.8版本的差异，

1. 在1.7中ConcurrentHashMap采用了分段锁技术，其中 Segment 继承于 ReentrantLock。每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。

   比如put中，首先会通过key定位到对应的Segment，然后会尝试获取锁，如果获取失败肯定就有其他线程存在竞争，会通过自旋锁增加竞争，失败一定次数后，会变成阻塞锁竞争，确保竞争到资源。

2. 1.8是抛弃了原有的 Segment 分段锁，而采用了 `CAS + synchronized` 来保证并发安全性。采取跟HashMap一样的策略，hash冲突后面会转化成红黑树。

#### 5.3 ArrayMap 跟 SparseArray 在 HashMap 上面的改进?

**SparseArray**

SparseArray 比 HashMap 更省内存，在某些条件下性能更好，主要是因为它避免了对key的自动装箱，内部是通过两个数组来进行数据存储的，一个存key，一个存value。它存储的元素是按key从小到大有序的，所以查找的时候可以用二分查找。

**ArrayMap**

有一个数组mHashes，存储key的hash值，mArrray数组是mHashes的两倍，依次保存key和value。查找的时候也是通过二分查找的方式，如果出现hash冲突会在相邻的位置插入。

### 6. Java的多线程安全问题

在并发编程中，我们通常会遇到以下三个问题：原子性问题，可见性问题，有序性问题。我们先看具体看一下这三个概念：

* **原子性**：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

  只有简单的读取、赋值（而且必须是将数字赋值给某个变量，变量之间的相互赋值不是原子操作）才是原子操作（如y=x,i++不是原子操作）。

* **可见性：**是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。
* **有序性**：即程序执行的顺序按照代码的先后顺序执行。举个简单的例子，看下面这段代码：

#### 6.1 synchronized的原理

synchronized 代码块是由一对儿 monitorenter/monitorexit 指令实现的。JVM对此进行优化，提供了几种不同的锁，自旋锁，偏向锁，轻量级锁，重量级锁。并能在检测到不同的竞争环境的时候，进行不同锁的切换，也就是锁的升级，降级。

**自旋锁**：就是让CPU做无用功，目的是占着CPU不放，等待获取锁的机会。

**偏向锁**：无竞争关系的时候，也就是相当于添加了一个变量，判断为true的时候无需再走加锁、解锁的流程

**轻量级锁**：当偏向锁运行在同步代码块的时候，另一个线程过来竞争的时候，偏向锁升级为轻量级锁；

**重量级锁**：重量锁在 JVM 中又叫对象监视器\(Monitor\)，他包含至少一个竞争锁队列，和一个阻塞队列吧。前者负责互斥，后者负责同步；

此外synchronized 修饰静态方法获取的是类锁\(类的字节码文件对象\)。

synchronized 修饰普通方法或代码块获取的是对象锁。这种机制确保了同一时刻对于每一个类实例，其所有声明为 synchronized 的成员函数中至多只有一个处于可执行状态。

#### 6.2 volatile 原理。

观察加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令，lock前缀指令实际上相当于一个**内存屏障**（也成内存栅栏），内存屏障会提供3个功能：

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. 它会强制将对缓存的修改操作立即写入主存；
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

根据内存屏障的功能，保证了可见性和禁止指令重排序。

#### 6.3 ReentrantLock的原理

ReetrantLock实现依赖于AQS\(AbstractQueuedSynchronizer\)，支持公平锁和非公平锁的重入锁。

* 公平锁：多个线程申请获取同一资源时，必须按照申请顺序，依次获取资源。
* 非公平锁：资源释放时，任何线程都有机会获得资源，而不管其申请顺序。

**公平锁**

FairSync继承Sync，而Sync继承AbstractQueuedSynchronizer。ReentrantLock调用lock方法，最终会调用sync的tryAcquire函数，获取资源。FairSync的tryAcquire函数，**当前线程只有在队列为空或者时队首节点的时候，才能获取资源，否则会被加入到阻塞队列中。**下面的hasQueuedPredecessors函数用于判断队列是否为空，或者当前元素是否为队首元素。

**非公平锁**

NoFairSync同样继承Sync，ReentrantLock调用lock方法，最终会调用sync的tryAcquire函数，获取资源。而NoFairSync的tryAcquire函数，会调用父类Sync的方法nofairTryAcquire函数。通过对比可以发现，如果资源释放时，新的线程会尝试CAS操作获取资源，而不管阻塞队列中时候有先于其申请的线程。

1. 先获取state值，若为0，意味着此时没有线程获取到资源，CAS将其设置为1， 设置成功则代表获取到排他锁了;
2. 若 state 大于 0，肯定有线程已经抢占到资源了，此时再去判断是否就是自己 抢占的，是的话，state 累加，返回 true，重入成功，state 的值即是线程重入的次数
3. 其他情况，则获取锁失败。

**可重入性**

获取独占资源的线程，可以重复得获取该独占资源，不需要重复请求。注意释放资源的时候，重入多少次，就必须释放多少次。

#### 6.4 CAS的介绍

CAS，Compare and Swap 即比较并交换，设计并发算法时常用到的一种技术，这个指令也带有`Lock`前缀，而且由于CAS是硬件级别的操作，效率会比普通锁更高一些。

#### 6.5 线程死锁的4个条件

**什么是死锁？**

当线程 A 持有独占锁 a，并尝试去获取独占锁 b 的同时，线程 B 持有独占锁 b，并尝试获取独占锁 a 的情况下，就会发生 AB 两个线程由于互相持有对方需要的锁，而发生的阻塞现象，我们称为死锁。

造成死锁的四个条件:

* 互斥条件:一个资源每次只能被一个线程使用。
* 请求与保持条件:一个线程因请求资源而阻塞时，对已获得的资源保持不放
* 不剥夺条件:线程已获得的资源，在未使用完之前，不能强行剥夺。
* 循环等待条件:若干线程之间形成一种头尾相接的循环等待资源关系。

#### 6.6 什么导致线程阻塞?

为了解决对共享存储区的访问冲突，Java 引入了同步机制，Java 提供了大量方法来支持阻塞，下面 让我们逐一分析。

**suspend\(\) 和 resume\(\) 方法**:两个方法配套使用，suspend\(\)使得线程进入阻塞状态，并且不会自动恢复，必须其对应的 resume\(\) 被调用，才能使得线程重新进入可执行状态。

**yield\(\) 方法**: yield\(\) 使得线程放弃当前分得的 CPU 时间，但是不使线程阻塞，即线程仍处于可执行状态，随时可能再次分得 CPU 时间。调用 yield\(\) 的效果等价于调度程序认为该线程已执行了足够的时间从而转到另一个线程。

**join方法**：join方法的本质调用的是Object中的wait方法实现线程的阻塞，不过他阻塞的事主线程。

**wait、sleep 的区别**

最大的不同是在等待时 wait 会释放锁，而 sleep 一直持有锁。wait 通常被用于线程间交互，sleep 通常被用于暂停执行。

* 首先，要记住这个差别，“sleep 是 Thread 类的方法,wait 是 Object 类中。
* Thread.sleep 不会导致锁行为的改变，如果当前线程是拥有锁的，那么sleep后依然持有锁
* Thread.sleep 和 Object.wait 都会暂停当前的线程，对于 CPU 资源来说，他都暂时不需要占用CPU执行时间，区别是代用wait后，需要别的线程执行 notify/notifyAll 才能够重新获得 CPU 执行时间。
* 线程的状态参考 Thread.State 的定义。新创建的但是没有执行\(还没有调用 start\(\)\)的线程处于“就绪”，或者说 Thread.State.NEW 状态。而Thread.State.BLOCKED\(阻塞\)表示线程正在获取锁时，因为锁不能获取到而被迫暂停执行下面的指令，一直等到这个锁被别的线程释放。

**notify** 运行过程：

当线程 A\(消费者\)调用 wait\(\)方法后，线程 A 让出锁，自己进入等待状态，同时加入锁对象的等待队列。 线程 B\(生产者\)获取锁后，调用 notify 方法通知锁对象的等待队列，使得线程 A 从等待队列进入阻塞队列。 线程 A 进入阻塞队列后，直至线程 B 释放锁后，线程 A 竞争得到锁继续从 wait\(\)方法后执行。

#### 6.7 线程的生命周期

* NEW:创建状态，线程创建之后，但是还未启动。
* RUNNABLE:运行状态，处于运行状态的线程，但有可能处于等待状态，
* WAITING:等待状态，一般是调用了 wait\(\)、join\(\)、LockSupport.spark\(\)
* TIMED\_WAITING:超时等待状态，也就是带时间的等待状态。一般是调wait\(time\)等方法。
* BLOCKED:阻塞状态，等待锁的释放，例如调用了 synchronized 增加了锁
* TERMINATED:终止状态，一般是线程完成任务后退出或者异常终止。

#### 6.8 run\(\)和 start\(\)方法区别?

1. start\(\)方法来启动线程，真正实现了多线程运行，这时无需等待 run 方法体代码执行完毕而直接继续执行下面的代码:
2. run\(\)方法当作普通方法的方式调用，程序还是要顺序执行，还是要等待 run 方法体执行完毕后才可继续执行下面的代码:

#### 6.9 如何停止一个线程

1. Thread有直接暂停的方法——Stop，但是这个方法被废弃了，因为不安全
2. Thread的间接停止方式——通知目标线程自行停止
   * interrupt 系统支持，出发方式是抛出异常，且调用JNI方法具有一定的开销
   * volatile 修饰对的Boolean  通过布尔值进行判断也可以抛出异常 （一般推荐）

### 7.Android中的跨进程通信

我们知道Android是基于Linux，Linux中常见的IPC方式。

**共享内存**

属于共享物理内存方式：多个进程共享同一段物理内存，当某个进程改变内存内容时，其它进程都能够知道。此种方式无需拷贝内容，但是需要信号量进行进程间同步。

**通过内核中转**

先将A发送的内容拷贝到内核，这过程可以理解为存储，再从内核拷贝到B的用户空间，这过程可以理解为转发，因此一次"存储-转发"过程需要两次内容拷贝。

**Android中的Binder**

虽然Linux提供了上述\(还有其它的如信号量等\)的IPC方式，但是由于每种方式都有其缺点，因此Android另起炉灶弄了另一种方式：Binder。

先来了解一下内存映射\(mmap\)。

上面提到过，"存储-转发"过程需要在用户空间和内核空间之间拷贝数据，还有一种不需要拷贝的方式：内存映射，内存映射就是将文件内容映射到用户空间中，只要在用户空间对映射对象进行操作，就相当于对文件进行操作。

![](https://mmbiz.qpic.cn/mmbiz_png/MOu2ZNAwZwMdfb25nt76qmRW3aHYfib3JiaZj0x78BznyLs3JUt4BbtVaY8ibVlxApXyoszeCfq3R8iccicp9Tyy5rg/640)

如上图，进程空间与内核空间共享磁盘上的一个映射文件，当进程空间往映射文件写入数据后（此处不是真正的I/O，而是内存读写），相当于写入了内核空间，反之亦然。

**Binder**

![](https://user-gold-cdn.xitu.io/2020/4/7/17154f81bb8bd672?)

进程A给进程B发送一段信息，流程如下：

> 1、进程A通过系统调用拷贝内容到内核空间。
>
> 2、由于内核空间与进程B做了内存映射，因此进程B能够知道内核空间的信息。

从上可知，通过Binder，进程A给进程B发送信息只进行了一次数据拷贝.

为什么不采用共享内存：多进程使用同一内存，容易导致死锁，不安全。

**Binder的优势**

1. 效率高，拷贝次数越少，传输效率越高
2. 稳定性：共享内存需要处理并发同步问题，容易出现死锁和资源竞争，Binder 基于 C/S 架构 ，Server 端与 Client 端相 对独立，稳定性较好。Socket同样是基于C/S架构但是主要用于网络间传输，效率较低。
3. 安全性：Binder 机制为每个进程分配了 UID/PID，且 在 Binder 通信时会根据 UID/PID 进行有效性检测。

**AIDL**

AIDL 的本质是系统提供了一套可快速实现 Binder 的工具。关键类和方法:

* AIDL 接口:继承 IInterface。
* Stub 类:Binder 的实现类，服务端通过这个类来提供服务。
* Proxy 类:服务端的本地代理，客户端通过这个类调用服务端的方法。
* asInterface\(\):客户端调用，将服务端返回的 Binder 对象，转换成客户端所需要的AIDL对象，如果客户端和服务端位于同一进程中，则返回stub对象，否则返回stub.proxy
* asBinder\(\):根据当前调用情况返回代理 Proxy 的 Binder 对象。
* onTransact\(\):运行在服务端的 Binder 线程池中，当客户端发起跨进程请求时，远程请求会通过底层封装后交由此方法处理
* transact\(\):运行在客户端，当客户端发起远程请求的同时将当前线程挂起。

当有多个业务模块都需要 AIDL 来进行 IPC，此时需要为每个模块创建特定的 aidl文件，那么相应的 Service 就会很多。必然会出现系统资源耗费严重、应用过度重量级的问题。解决办法是建立 Binder 连接池，即将每个业务模块的 Binder 请求统一转发到一个远程 Service 中去执行，从而避免重复创建 Service。

工作原理:每个业务模块创建自己的 AIDL 接口并实现此接口，然后向服务端提供自己的唯一标识和其对应的 Binder 对象。服务端只需要一个 Service 并提供一个 queryBinder 接口，它会根据业务模块的特征来返回相应的 Binder 对象，不同的业务模块拿到所需的 Binder 对象后就可以进行远程方法的调用了。

### 8. Android系统的启动流程

1. 启动电源以及系统启动
2. 引导程序BootLoader：它是Android操作系统开始运行前的一个小程序，主要将操作系统OS拉起来并进行
3. Linux内核启动
4. init进程启动： 1. 创建和挂载启动所需的文件目录 2. 初始化和启动属性服务 3. 解析init.rc配置文件并启动Zygote进程
5. Zygote进程的启动:
   1. 创建AppRuntime，执行其start方法，启动Zygote进程。。
   2. 创建JVM并为JVM注册JNI方法。
   3. 使用JNI调用ZygoteInit的main函数进入Zygote的Java FrameWork层。
   4. 使用registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等等AMS的请求去创建新的应用进程。
   5. 启动SystemServer进程。
6. SystemServer进程启动:
   1. 启动Binder线程池，这样就可以与其他进程进行Binder跨进程通信。
   2. 创建SystemServiceManager，它用来对系统服务进行创建、启动和生命周期管理。
   3. 启动各种系统服务：引导服务、核心服务、其他服务，共100多种。应用开发主要关注引导服务ActivityManagerService、PackageManagerService和其他服务WindowManagerService、InputManagerService即可
7. Launcher启动。

## 2. Android常见问题补充

### 1. 横竖屏切换时候 Activity 的生命周期

不设置 Activity 的 android:configChanges 时，切屏会重新回调各个生命周期，切横屏时会执行一次，切竖屏时会执行两次。 设置 Activity 的android:configChanges=”orientation”时，切屏还是会调用各个生命周期，切换横竖屏只会执行一次 设置 Activity 的 android:configChanges=”orientation\|keyboardHidden”时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged 方法。

### 2. AsyncTask 的缺陷和问题，说说他的原理。

**AsyncTask** 是什么?

AsyncTask 是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程并在主线程中更新 UI。

他的本质是两个线程池，和一个handler。线程池一个用来任务排队，一个用来执行任务

**关于线程池**

AsyncTask 对应的线程池 ThreadPoolExecutor 都是进程范围内共享的，且都是static 的，所以是 Asynctask 控制着进程范围内所有的子类实例。也可以自定义线程池。

补充问题：

1. 生命周期：AsynTask 会一直执行，直到 doInBackground\(\)方法 执行完毕，不会随着Activity的销毁而销毁，所以要记得做正确的取消操作
2. 内存泄漏：如果 AsyncTask 被声明为 Activity 的非静态内部类，那么 AsyncTask 会保留一个对 Activity 的引用。如果 Activity 已经被销毁，AsyncTask 的后台线程还在执行，它将继续在内存里保留这个引用，导致 Activity 无法被回收，引起内存泄漏。
3. 结果丢失：屏幕旋转或 Activity 在后台被系统杀掉等情况会导致 Activity 的重新创建，之前运行的 AsyncTask 会持有一个之前 Activity 的引用，这个引用已经无效，这时调用 onPostExecute\(\)再去更新界面将不再生效。
4. 并行还是串行：在 Android1.6 之前的版本，AsyncTask 是串行的，在 1.6 之后3.0之前是并行，后来又改为串行的执行任务

### 3. 保存 & 恢复Activity/Fragment状态缓存 - onSaveInstanceState\(\)、onRestoreInstanceState\(\)

**Activity的状态保存**

onSaveInstanceState：是当系统 **未经你许可** 时，**可能** 销毁了你的Activity（如内存不足，用户按下home键，横竖屏切换），则会被系统调用 。如果是主动销毁，比如说按返回键则不会触发。

onRestoreInstanceState（）：若 异常关闭了Activity，即调用了onSaveInstanceState（） & 下次启动时会调用onRestoreInstanceState（）。此时的调用顺序是：

1. onCreate（）
2. onStart（）
3. onRestoreInstanceState（）
4. onResume（）

onSaveInstanceState的bundle参数会传递到onCreate方法中，可选择在onCreate（）中做数据还原。

**Fragment的状态保存**

1. Fragment 的状态保存, 在 Activity 的 onSaveInstanceState\(\)里, 调用了FragmentManger 的 saveAllState\(\)方法, 其中会对 mActive 中各个 Fragment 的实例状态和 View 状态分别进行保存.
2. FragmentManager 还提供了 public 方法: saveFragmentInstanceState\(\), 可以对单个 Fragment 进行状态保存, 这是提供给我们用的。
3. FragmentManager 的 moveToState\(\)方法中, 当状态回退到ACTIVITY\_CREATED, 会调用 saveFragmentViewState\(\)方法, 保存 View 的状态.

### 4. Android中的动画

Android中的动画主要分三大类：

1. tween 补间动画。通过指定 View 的初末状态和变化方式，对 View 的内容进行一系列的变化。只是影像变化，view 的实际位置还在原来地方。
2. frame 帧动画。AnimationDrawable 控制 animation-list.xml 布局，类似电影一帧一帧的动画效果
3. PropertyAnimation 属性动画 ，3.0 引入，属性动画核心思想是对值的变化，他真正的实现了 view 的移动。

其中最重要的就是属性动画，其中属性动画有几个重要的步骤；

1. 计算属性值
   * 计算已完成动画分数 elapsed fraction。为了执行一个动画，你需要创建一个ValueAnimator，并且指定目标对象属性的开始、结束和持续时间。
   * 计算插值\(动画变化率\)interpolated fraction 。当 ValueAnimator 计算完已完成的动画分数后，它会调用当前设置的 TimeInterpolator，去计算得到一个interpolated\(插值\)分数，在计算过程中，已完成动画百分比会被加入到新的插值计算中。
   * 计算属性值当插值分数计算完成后，ValueAnimator 会根据插值分数调用合适的TypeEvaluator 去计算运动中的属性值。 以上分析引入了两个概念:已完成动画分数\(elapsed fraction\)、插值分数\( interpolated fraction \)。
2. 为目标对象的属性设置属性值，即应用和刷新动画

### 5. Android中的Context

![](https://upload-images.jianshu.io/upload_images/10107787-ab22cce86fa4f1da.png)

如上图所示，**Context 使用了装饰模式，除了 ContextImpl 外，其他 Context 都是 ContextWrapper 的子类**

注意几点：

1. 任何一个 Context 的实例，只要调用 getApplicationContext\(\)方法都可以拿到我们的 Application 对象。getApplication也是获取Application的但是只有Activity个Service才能调用。
2. 创建对话框时不可以用 Application 的 context，只能用 Activity 的context

**activity 的 startActivity 和 context 的 startActivity 区别?**

\(1\)、从 Activity 中启动新的 Activity 时可以直接 mContext.startActivity\(intent\)就ok

\(2\)、如果从其他 Context 中启动 Activity 则必须给 intent 设置 Flag

怎么在 **Service** 中创建 **Dialog** 对话框?

1. 在我们取得 Dialog 对象后，需给它设置类型，即:dialog.getWindow\(\).setType\(WindowManager.LayoutParams.TYPE\_SYSTEM\_ALERT\)
2. 在 Manifest 中加上权限:

### 6.  bundle传递对象为什么需要序列化?Serialzable 和 Parcelable 的区别?

因为 bundle 传递数据时只支持基本数据类型，所以在传递对象时需要序列化转换成可存储或可传输的本质状态\(字节流\)。

两者最大的区别在于 **存储媒介的不同**，`Serializable` 使用 **I/O 读写存储在硬盘上**，而 `Parcelable` 是直接 **在内存中读写**。很明显，内存的读写速度通常大于 IO 读写，所以在 Android 中传递数据优先选择 `Parcelable`。

Serializable 会使用反射，序列化和反序列化过程需要大量 I/O 操作，Parcelable 自已实现封送和解封\(marshalled &unmarshalled\)操作不需要用反射，数据也存放在 Native 内存中，效率要快很多。

### 7. Bitmap 使用时候注意什么?

1. 选择合适的bitmap类型，图片规格:
   * ALPHA\_8  每个像素占用 1byte 内存
   * ARGB\_4444 每个像素占用 2byte 内存
   * ARGB\_8888 每个像素占用 4byte 内存\(默认\)
   * RGB\_565 每个像素占用 2byte 内存（没有透明通道）
2. 降低采样率inSampleSize。
   1. inSampleSize的值必须大于1时才会有效果，且采样率同时作用于宽和高；
   2. 假设一张1024_1024，模式为ARGB\_8888的图片,inSampleSize=2，原始占用内存大小是4MB，采样后的图片占用内存大小就是\(1024/2\)_  \(1024/2 \)\* 4 = 1MB
3. 复用内存。即，通过软引用\(内存不够的时候才会回收掉\)，复用内存块，不需要再重新给这个 bitmap 申请一块新的内存，避免了一次内存的分配和回收，从而改善了运行效率。
4. 使用 recycle\(\)方法及时回收内存。
5. 压缩图片

**在 Bitmap 里有两个获取内存占用大小的方法。**

getByteCount\(\):API12 加入，代表存储 Bitmap 的像素需要的最少内存。

getAllocationByteCount\(\):API19 加入，代表在内存中为 Bitmap 分配的内存大小，代替了 getByteCount\(\) 方法。在不复用 Bitmap 时，getByteCount\(\) 和getAllocationByteCount 返回的结果是一样的。在通过复用 Bitmap 来解码图片时，那么 getByteCount\(\) 表示新解码图片占用内存的大 小，

### 8. 编译期注解跟运行时注解

运行期注解\(RunTime\)利用反射去获取信息还是比较损耗性能的。

编译期\(Compile time\)注解，以及处理编译期注解的手段 APT 和 Javapoet，

### 9. Service的启动方式有什么区别

1. startService:

   onCreate\(\)---&gt;onStartCommand\(\) ---&gt; onDestory\(\)

   如果服务已经开启，不会重复的执行 onCreate\(\)， 而是会调用onStartCommand\(\)。一旦服务开启跟调用者\(开启者\)就没有任何关系了。 开启者退出了，开启者挂了，服务还在后台长期的运行。 开启者不能调用服务里面的方法。

2. bindService

   onCreate\(\) ---&gt;onBind\(\)---&gt;onunbind\(\)---&gt;onDestory\(\)

   bind 的方式开启服务，绑定服务，调用者挂了，服务也会跟着挂掉。 绑定者可以调用服务里面的方法。

### 10. WebView的使用

**为什么Webview加载较慢**

这是因为在客户端中，加载H5页面之前，需要先初始化WebView，在WebView完全初始化完成之前，后续的界面加载过程都是被阻塞的。

常见的方式

* 全局 WebView。
* 客户端代理页面请求。WebView 初始化完成后向客户端请求数据。
* asset 存放离线包。

**Android** 通过 **WebView** 调用 **JS** 代码:

1、通过 WebView 的 loadUrl\(\):

设置与 Js 交互的权限:webSettings.setJavaScriptEnabled\(true\)

设置允许 JS 弹窗:webSettings.setJavaScriptCanOpenWindowsAutomatically\(true\)

载入 JS 代码:mWebView.loadUrl\("file:///android\_asset/javascript.html"\)

webview 只是载体，内容的渲染需要使用 webviewChromClient 类去实现，通过设置 WebChromeClient 对象处理 JavaScript 的对话框。PS：JS 代码调用一定要在 onPageFinished\(\) 回调之后才能调用，否则不会调用。

2、通过 WebView 的 evaluateJavascript\(\):

只需要将第一种方法的 loadUrl\(\)换成 evaluateJavascript\(\)即可，通过onReceiveValue\(\)回调接收返回值。

**JS** 通过 **WebView** 调用 **Android** 代码:

1、通过 WebView 的 addJavascriptInterface\(\)进行对象映射:

2、通过 WebViewClient 的方法 shouldOverrideUrlLoading \(\)回调拦截 url:Android 通过 WebViewClient 的回调方法 shouldOverrideUrlLoading \(\)，拦截Url，解析该 url 的协议。

3、通过 WebChromeClient 的 onJsAlert\(\)、onJsConfirm\(\)、onJsPrompt\(\)方法回调拦截 JS 对话框 alert\(\)、confirm\(\)、prompt\(\) 消息:

### 11. Android 系统架构

![](https://developer.android.com/guide/platform/images/android-stack_2x.png?hl=zh-cn)

1. **应用程序**
2. **Java FrameWork层**
3. **系统运行层**：
   * C/C++库：许多核心 Android 系统组件和服务\(例如 ART 和 HAL\)构建自原生代码，需要用c来编写
   * Android Runtime：对于运行 Android 5.0（API 级别 21）或更高版本的设备，每个应用都在其自己的进程中运行，并且有其自己的 [Android Runtime \(ART\)](https://source.android.com/devices/tech/dalvik/index.html?hl=zh-cn) 实例。ART 编写为通过执行 DEX 文件在低内存设备上运行多个虚拟机，DEX 文件是一种专为 Android 设计的字节码格式，经过优化，使用的内存很少。编译工具链（例如 [Jack](https://source.android.com/source/jack.html?hl=zh-cn)）将 Java 源代码编译为 DEX 字节码，使其可在 Android 平台上运行。ART的主要功能：
     * 预先 \(AOT\) 和即时 \(JIT\) 编译
     * 优化的垃圾回收 \(GC\)
     * 在 Android 9（API 级别 28）及更高版本的系统中，支持将应用软件包中的 [Dalvik Executable 格式 \(DEX\) 文件转换为更紧凑的机器代码](https://developer.android.com/about/versions/pie/android-9.0?hl=zh-cn#art-aot-dex)。
     * 更好的调试支持，包括专用采样分析器、详细的诊断异常和崩溃报告，并且能够设置观察点以监控特定字段
4. **硬件抽象层**：硬件抽象层 \(HAL\) 提供标准界面，向更高级别的 Java API 框架显示设备硬件功能。HAL 包含多个库模块，其中每个模块都为特定类型的硬件组件实现一个界面，例如相机或蓝牙模块。当框架 API 要求访问设备硬件时，Android 系统将为该硬件组件加载库模块。
5. **Linux 内核层**

### 12. View 的事件分发机制?滑动冲突怎么解决?

View 事件分发本质就是对 MotionEvent 事件分发的过程。即当一个 MotionEvent发生后，系统将这个点击事件传递到一个具体的 View 上。

**事件分发流程**：

**dispatchTouchEvent**:方法返回值为 true 表示事件被当前视图消费掉;返回为super.dispatchTouchEvent 表示继续分发该事件，返回为 false 表示交给父类的onTouchEvent 处理。

**onInterceptTouchEvent**:方法返回值为 true 表示拦截这个事件并交由自身的onTouchEvent 方法进行消费;返回 false 表示不拦截，需要继续传递给子视图。如果 return super.onInterceptTouchEvent\(ev\)， 事件拦截分两种情况:

* 1.如果该 View 存在子 View 且点击到了该子 View, 则不拦截, 继续分发 给子View处理
* 2.如果该 View 没有子 View 或者有子 View 但是没有点击中子 View\(此时ViewGroup相当于普通View）的onTouch方法响应。

**onTouchEvent**:方法返回值为 true 表示当前视图可以处理对应的事件;返回值为false，表示不处理，则会返回到父类中去进一步处理。如果 return super.onTouchEvent\(ev\)，事件处理分为两种情况:

* 1.如果该 View 是 clickable 或者 longclickable 的,则会返回 true, 表示消费了该事件与返回true一样
* 2.如果该 View 不是 clickable 或者 longclickable 的,则会返回 false, 表示不消费该事件，继续向上传递。

注意:在 Android 系统中，拥有事件传递处理能力的类有以下三种:

1. Activity:拥有分发和消费两个方法。
2. ViewGroup:拥有分发、拦截和消费三个方法。
3. View:拥有分发、消费两个方法。

**一些重要的结论**:

1. 事件传递优先级:onTouchListener.onTouch &gt; onTouchEvent &gt;onClickListener.onClick。
2. 正常情况下，一个事件序列只能被一个 View 拦截且消耗。因为一旦一个元 素拦截了此事件，那么同一个事件序列内的所有事件都会直接交给它处理\(即不 会再调用这个 View 的拦截方法去询问它是否要拦截了，而是把剩余的ACTION\_MOVE、ACTION\_DOWN 等事件直接交给它来处理\)。特例:通过将重 写 View 的 onTouchEvent 返回 false 可强行将事件转交给其他 View 处理
3. 如果 View 不消耗除 ACTION\_DOWN 以外的其他事件，那么这个点击事件会消失，此时父元素的 onTouchEvent 并不会被调用，并且当前 View 可以持续收到后续的事件，最终这些消失的点击事件会传递给 Activity 处理。
4. ViewGroup默认不拦截
5. View 的 onTouchEvent 默认都会消耗事件\(返回 true\)，除非它是不可点击的\(clickable 和 longClickable 同时为 false\)。View 的 longClickable 属性默认都为 false，clickable 属性要分情况，比如 Button 的 clickable 属性默认为 true
6. View 的 enable 属性不影响 onTouchEvent 的默认返回值。
7. 通过 **requestDisallowInterceptTouchEvent** 方法可以在子元素中干预父元素的事件分发过程，但是 ACTION\_DOWN 事件除外。

### 13. APK 的安装流程

![](http://solart.cc/images/Install_apk.png)

**第一步：拷贝文件到指定的目录：**

**第二步：解压缩apk，创建应用的数据目录**

**第三步：解析apk的AndroidManifest.xml文件**

**第四步：安装完毕，发送广播，显示快捷方式**

### 14. Android的打包流程

![](https://user-gold-cdn.xitu.io/2019/5/6/16a8c918da070cc0)

* 通过 AAPT 工具进行资源文件\(包括 AndroidManifest.xml、布局文件等）生成R.java
* 通过 AIDL 工具处理 AIDL 文件，生成相应的 Java 文件。
* 通过 Java Compiler 编译 R.java、Java 接口文件、Java 源文件，生成.class文件
* 通过 dex 命令，将.class 文件和第三方库中的.class 文件处理生成.dex
* 通过 ApkBuilder 工具将资源文件、DEX 文件打包生成 APK 文件。
* 通过 Jarsigner 工具，利用 KeyStore 对生成的 APK 文件进行签名。
* 如果是正式版的 APK，还会利用 ZipAlign 工具进行对齐处理，这样通过内存映射的访问Apk文件的速度会更快，并且可以减少内存占用。

### 15. Android中的签名

**什么是签名?**

在 Apk 中写入一个“指纹”。指纹写入以后，Apk 中有任何修改，都会导致这个指纹无效，Android 系统在安装 Apk 进行签名校验时就会不通过，从而保证了安全性。

**数字摘要**

对一个任意长度的数据，通过一个 Hash 算法计算后，都可以得到一个固定长度的二进制数据，这个数据就称为“摘要”。

**签名和校验的主要过程**

签名就是在摘要的基础上再进行一次加密，对摘要加密后的数据就可以当作数字签名

**签名过程**：

1、计算摘要:通过 Hash 算法提取出原始数据的摘要。

2、计算签名:再通过基于密钥\(私钥\)的非对称加密算法对提取出的摘要，加密后的数据就是签名

3、写入签名:将签名信息写入原始数据的签名区块内。

**校验过程**:

1、首先用同样的 Hash 算法从接收到的数据中提取出摘要。

2、解密签名:使用发送方的公钥对数字签名进行解密，解密出原始摘要。

3、比较摘要:如果解密后的数据和提取的摘要一致，则校验通过;否则不通过。

**V1和V2的区别**

V1：应该是通过ZIP条目进行验证，这样APK 签署后可进行许多修改 - 可以移动甚至重新压缩文件。

V2：验证压缩文件的所有字节，而不是单个 ZIP 条目，因此，在签名后无法再更改\(包括 zipalign\)。正因如此，现在在编译过程中，我们将压缩、调整和签署合并成一步完成。好处显而易见，更安全而且新的签名可缩短在设备上进行验证的时间（不需要费时地解压缩然后验证），从而加快应用安装速度。

### 16. Glide

**Glide 的三层缓存机制:**

Glide 缓存机制大致分为三层:内存缓存、弱引用缓存、磁盘缓存。

取的顺序是:内存、弱引用、磁盘。

存的顺序是:弱引用、内存、磁盘。

通过Glide来加载图片的时候会优先从内存中去，如果内存中没有，就从一个弱引用的map activeResources中找，如果也没有就从Lrucache。如果都没有，就会发起请求，请求成功后，资源放到 diskLrucache 和 activeResources 中。

**LruCache** 原理

内存缓存基于 LruCache 实现，磁盘缓存基于 DiskLruCache 实现。这两个类都基于 Lru 算法和 LinkedHashMap 来实现。

LRU 是 Least Recently Used 的缩写，最近最少使用算法。LruCache 的原理就是利用 LinkedHashMap 持有对象的强引用，按照 Lru 算法进行对象淘汰。具体说来假设我们从表尾访问数据，在表头删除数据，当访问的数据项在链表中存在时，则将该数据项移动到表尾，否则在表尾新建一个数据项。当链表容量超过一定阈值，则移除表头的数据。

Lrucache 需要移除一个缓存时，会调用 resource.recycle\(\)方法。注意到该方法将不用的Bitmap放到BitmapPool中。

**LinkedHashMap的原理**

LinkedHashMap 几乎和 HashMap 一样\(他是HashMap的子类\):从技术上来说，不同的是它定义了一个 Entry header，这个 header 不是放在 Table 里，它是额外独立出来的。LinkedHashMap 通过继承 hashMap 中的 Entry,并添加两个属性Entry before,after,和 header 结合起来组成一个双向链表，来实现按插入顺序或访问顺序排序。

### 17. Leakcanary的原理

主要分为如下7个步骤：

* 1、RefWatcher.watch\(\)创建了一个KeyedWeakReference用于去观察对象。
* 2、然后，在后台线程中，它会检测引用是否被清除了，并且是否没有触发GC。
* 3、如果引用仍然没有被清除，那么它将会把堆栈信息保存在文件系统中的.hprof文件里。
* 4、HeapAnalyzerService被开启在一个独立的进程中，并且HeapAnalyzer使用了HAHA开源库解析了指定时刻的堆栈快照文件heap dump。
* 5、从heap dump中，HeapAnalyzer根据一个独特的引用key找到了KeyedWeakReference，并且定位了泄露的引用。
* 6、HeapAnalyzer为了确定是否有泄露，计算了到GC Roots的最短强引用路径，然后建立了导致泄露的链式引用。
* 7、这个结果被传回到app进程中的DisplayLeakService，然后一个泄露通知便展现出来了。

官方的原理简单来解释就是这样的：**在一个Activity执行完onDestroy\(\)之后，将它放入WeakReference中，然后将这个WeakReference类型的Activity对象与ReferenceQueque关联。这时再从ReferenceQueque中查看是否有没有该对象，如果没有，执行gc，再次查看，还是没有的话则判断发生内存泄露了。最后用HAHA这个开源库去分析dump之后的heap内存**

### 18. Android中的热修复，插件化

1. BootClassLoader：用于加载Framework层的class
2. PathClassLoader：用于加载已经安装到系统中的apk中的class
3. DexClassLoader：加载指定目录中的class文件
4. BaseDexClassLoader: 是 PathClassLoader 和 DexClassLoader 的父类。

**Tinker如何实现热修复**

1. 原理就是将新的Dex插入到Runtime老的dex前面
2. 重点就是基于dexDiff的差分算法（双指针同步处理，new和old 两个apk文件）
3. 利用 PathClassLoader 和 DexClassLoader 去加载与 bug 类同名的类。
4. 资源的修复参考上面的章节AssetsMangager
5. 同时注意异常监控的，做了很好的闭环，以及良好的注释

**资源修复**：

很多热修复框架的资源修复参考了 Instant Run 的资源修复的原理。AssetManager是加载资源的核心，并且AssetPath本身就支持多个资源

1、创建新的 AssetManager，通过反射调用 addAssetPath 方法加载外部的资源，这样新建的AssetManager就含有了外部的资源。

2、将 AssetManager 类型的 mAssets 字段的引用全部替换为新创建的新的AssetManager

**插件化的原理**

1. 占坑：预先在在AndroidManifest中注册Activity来占坑。
2. 通过hook的方式，拦截startActivity的方法，判断是启动插件还是内部的Activity
3. 将上一步中的参数，重新封装一个 subIntent 用来启动 StubActivity，并将前面得到的 TargetActivity保存到intent中
4. 这时候启动目标就变成了StubActivity，绕过AMS的校验
5. ActivityThread中重写mH中handleMessage对于launch activity的处理，将subActivity还原成原来的targetActivity，并且将intent还原。

**插件 Activity 的生命周期:** AMS 和 ActivityThread 之间的通信采用了 token 来对 Activity 进行标识，并且此后的 Activity 的生命周期处理也是根据 token 来对 Activity 进行标识，performLaunchActivity的r.token就是TargetActivity。所以TargetActivity是具有生命周期的。

**资源插件化:**

资源的插件化方案主要有两种:

* 合并资源方案，将插件的资源全部添加到宿主的 Resources 中，这种方案可以访问宿主的资源
* 构建插件资源方案，每个插件都构造出独立的 Resources，这种方案插件不能访问宿主的资源

### 19. **ListView**、**RecyclerView** 区别?

用法上差不多，RecyclerView 相比 ListView 在基础使用上的区别主要有如下几点:

* ViewHolder 的编写规范化了
* RecyclerView 复用 Item 的工作 Google 全帮你搞定，不再需要像ListView那样自己设置tag了
* RecyclerView 需要多出一步 LayoutManager 的设置工作
* RecyclerView 支持 线性布局、网格布局、瀑布流布局 三种
* RecyclerView支持单个Item的更新

ListView 的有点比较好，默认支持顶部和底部布局。

**两者的缓存机制也不相同**

ListView\(两级缓存\)，RecyclerView\(四级缓存\)：RecyclerView比ListView多两级缓存，支持多个离屏ItemView缓存，支持开发者自定义缓存处理逻辑，支持所有RecyclerView共用同一个RecyclerViewPool\(缓存池\)。

ListView和RecyclerView缓存机制基本一致：缓存离开屏幕的Item，给进入的item使用。

1\). RecyclerView缓存RecyclerView.ViewHolder，抽象可理解为：View + ViewHolder\(避免每次createView时调用findViewById\) + flag\(标识状态\)；

2\). ListView缓存View。

**RecyclerView的局部刷新**

以RecyclerView中notifyItemRemoved\(1\)为例，最终会调用requestLayout\(\)，使整个RecyclerView重新绘制，过程为：onMeasure\(\)--&gt;onLayout\(\)--&gt;onDraw\(\)

其中，onLayout\(\)为重点，分为三步：

1. dispathLayoutStep1\(\)：记录RecyclerView刷新前列表项ItemView的各种信息，如Top,Left,Bottom,Right，用于动画的相关计算；
2. dispathLayoutStep2\(\)：真正测量布局大小，位置，核心函数为layoutChildren\(\)；
3. dispathLayoutStep3\(\)：计算布局前后各个ItemView的状态，如Remove，Add，Move，Update等，如有必要执行相应的动画.

需要指出，ListView和RecyclerView最大的区别在于数据源改变时的缓存的处理逻辑，ListView是"一锅端"，将所有的mActiveViews都移入了二级缓存mScrapViews，而RecyclerView则是更加灵活地对每个View修改标志位，区分是否重新bindView。

### 20. OKHTTP的实现原理

**这个库的核心实现原理是什么?如果让你实现这个库的某些核心功能，你会考虑怎么去实现?**

1. OkHttp 内部的请求流程:使用 OkHttp 会在请求的时候初始化一个 Call 的实例，然后执行它的execute\(\)方法或enqueue\(\)方法，
2. 内部最后都会执行到getResponseWithInterceptorChain\(\)方法。这个方法里面通过拦截器组成的责任链，依次经过用户自定义普通拦截器、重试拦截器、桥接拦截器、缓存拦截器、连接拦截器和用户自定义网络拦截器以及访问服务器拦截器等拦截处理过程，来获取到一个响应并交给用户其中。

除了OKHttp的内部请求流程这点之外，缓存和连接这两部分内容也是两个很重要的点

**ConnectionPool 连接池**：

1、判断连接是否可用，不可用则从 ConnectionPool 获取连接，ConnectionPool，无连接，创建新连接，握手，放入 ConnectionPool。

2、它是一个 Deque，add 添加 Connection，使用线程池负责定时清理缓存。

3、使用连接复用省去了进行 TCP 和 TLS 握手的一个过程。

## 3. Java相关问题的补充

### 1.谈谈对 java 多态的理解?

多态的定义:指允许不同类的对象对同一消息做出响应。即同一消息可以根据发 送对象的不同而采用多种不同的行为方式。

多态的三个必要条件:

* 继承父类。
* 重写父类的方法。
* 父类的引用指向子类对象。

可以实现动态加载技术，因为他消除了类型间的耦合关系。

### 2. Java中集合框架

![](https://pic3.zhimg.com/80/v2-76c3c04de2e8609c488fa0081fb99c26_1440w.png)_\*\*_

总体汇聚成上面的图

List：有序、可重复;索引查询速度快;插入、删除伴随数据移动，速度慢;

Set：无序，不可重复;

Queue：一种数据结构吧FIFO

剩下问题如Vector和ArrayList和Linkedlist的差别，基本就是数据结构的区别。

Vector 作为动态数组，是有能力在数组中的任何位置添加或者删除元素的。因此，Stack 继承了 Vector，Stack 也有这样的能力,

所以java官网推荐Stack使用是

```java
   Deque<Integer> stack = new ArrayDeque<Integer>();
```

### 3. Java反射

反射就是在运行状态中,对于任意一个类,都能够知道这个类的所有属性和方法;对于任意一个对象,都能够调用它的任意方法和属性;并且能改变它的属性。是Java被视为动态语言的关键。

反射的核心是 JVM 在运行时才动态加载类或调用方法/访问属性，它不需要事先（写代码的时候或编译期）知道运行对象是谁。

### 4. 泛型

**4.1 简单介绍一下 java 中的泛型，泛型擦除以及相关的概念**

泛型的基本原理就是Java中的类型擦除，类型擦除是指，使用泛型时加上的类型参数，在编译时会去掉，最后得到编译后的产物（字节码）是不包含泛型参数的。

**类型擦除也会带来一些问题**

* 类型擦除后，如何保证只使用指定类型，编译器做了优化，先检查，再编译
* 基本类型不能作为泛型实参，这就意味过程中会有装箱和拆箱的开销
* 泛型类型无法作为方法重载（类型擦除后，参数类型是object），但是可以重写。编译器会自己生成两个桥方法，方法的参数类型就是object，桥方法的内部实现就是调用我们自己的重写方法。
* 泛型无法作为真实的类型使用 比如数new一个对象等，这也是为什么Gson.fromJson需要传入Class的原因
* 静态方法无法引用类泛型参数

  泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数。

* 泛型在类型强转时会增加运行时开销

类型虽然被擦除了，但是被擦除的信息还是会以元素附加的签名信息，存储下来。

#### 4.2 方法分派

分派也就是java的多态的体现，他分为静态分派和动态分派

首先需要理解变量类型，比如：`Object obj = new String("");`

* 静态类型

  定义变量时声明的类型。这里的obj的静态类型就是Object，静态类型在编译期就确定了

* 实际类型

  obj的实际类型就是String，实际类型时在运行时确定的

然后我们在看静态分配和动态分配

* **静态分配 ——方法的重载**

  根据变量的静态类型匹配调用方法的过程，他的字节码中显示，他是按照静态分派选择匹配静态类型的重载方法，而不是按照实际类型

* **动态分配——方法的重写**

  根据变量的实际类型匹配调用方法的过程，就是动态分配。

  Java虚拟机在方法区中建立一个虚方法表（Virtual Method Table），通过使用方法表的索引来代替元数据查找以提高性能。虚方法表中存放着各个方法的实际入口地址（由于Java虚拟机自己建立并维护的方法表，所以没有必要使用符号引用，那不是跟自己过不去嘛），如果子类没有覆盖父类的方法，那么子类的虚方法表里面的地址入口与父类是一致的；如果重写父类的方法，那么子类的方法表的地址将会替换为子类实现版本的地址。

  方法表是在类加载的连接阶段（验证、准备、解析）进行初始化，准备了子类的初始化值后，虚拟机会把该类的虚方法表也进行初始化。

### 5. 1、Java 的 char 是两个字节，是怎么存 Utf-8 的字符的?

1. java中的char不存utf-8它存储的是UTF-16
2. UTF-8是Unicode的一种实现方式，他是一种可变长度的编码方式，对于英文字母的UTF-8编码和ASCII码是相同的
3. UTF-16 具体定义了 Unicode 字符在计算机中存取方法，用两个字节来表示 Unicode 转化格式，这个是定长的表示方法，不论什么字符都能用两个字节
4. Unicode 是字符集（存储了所有的符号），他的作用是将字符转换成码点，也就是从人类的认知翻译成计算机可识别的代码

   人类认知的字符=&gt;字符集=&gt;计算机存储

5. java 中string 的length不是字符数而是char数组的长度，所以有时候会出现他的长度和字符数不想等的情况
6. 另外需要注意的是中文字符“中”获取UTF-16的字符长度为4个字节，比预计多出两个字节的，额外的两个字节是字符序

### 6. Java 的string 能有多长

1. 当String 存储在栈中
   * 受到字节码中CONSTANT\_UTF8\_info的限制，最终通过MUTF-8编码格式的字节数不能超过65535
   * 拉丁字符受到javac限制，最多是65534
   * 如果运行方法去较小，也会受到方法区大小的限制
2. 当String 存储在堆中
   * 受到虚拟机的指令限制，理论上最大值Integer.MAX\_VALUE
   * 实际可能小于理论上的值，虚拟机本身保留空间给头信息
   * 堆内存很小的话也会限制长度本身

### 7. Java的匿名内部类有哪些限制

1. 匿名内部类就是没有名字的内部类，只能继承一个父类或一个接口

   虚拟机中编译的匿名内部类其实是由名字的，比如外部类的路径$1

2. 匿名内部类分为静态和非静态
   * 静态匿名内部类

     静态匿名内部类不会持有外部引用

   * 非静态匿名内部类

     需要父类的外部实例来进行初始化
3. 匿名内部类的构造方法

   匿名内部类的构造方法是由编译器自动生成，当他是非静态的时候，默认的构造方法参数中会有外部的实例，以及自己外部类的实例

4. 只能捕获外部作用域内的final变量
5. java的中的匿名内部类不能被继承，kotlin 是可以的。

### 8. Java中对异常如何分类

1. Java 异常结构中定义有 Throwable 类。 Exception 和 Error 为其子类。
2. Error 是程序无法处理的错误，比如 OutOfMemoryError、StackOverflowError。发生时会终止程序
3. Exception 是程序本身可以处理的异常，这种异常分两大类运行时异常和非运行异常，程序应当去处理
4. 运行时异常都是 RuntimeException 类及其子类异常，如 NullPointerException等，这些异常不是编译时异常，如果遇到要尽量处理。

**NoClassDefFoundError** 和 **ClassNotFoundException** 有什么区别?

**ClassNotFoundException** 的产生原因主要是: Java 支持使用反射方式在运行时动态加载类，如调用反射的时候，但是在对应的路径没有找到这个类，就会抛出这个异常。

**NoClassDefFoundError** 产生的原因在于: 如果 JVM 或者 ClassLoader 实例尝试加载类的时候，找不到类的定义，编译时存在，但是运行时却找不到。

### 9. String 为什么要设计成不可变的?

String 本质上是个 final 的 char\[\]数组，不可变可以线程安全和字符串常量池的实现 。

### 10. Java 里的幂等性了解吗?

幂等性：对同一个系统，使用同样的条件，一次请求和重复的多次请求对系统资源的影响是一致的。

类似token的机制，或者订单id，通过一个标志表示订单状态，可以避免重复操作。

### 11. String，StringBuffer，StringBuilder 有哪些不同?

三者在执行速度方面的比较:StringBuilder &gt; StringBuffer &gt; String

String 每次变化一个值就会开辟一个新的内存空间

StringBuilder:线程非安全的

StringBuffer:线程安全的

单线程情况下使用StringBuilder，少的直接用string，大量数据处理需要线程安全则使用StringBuffer

### 12. 静态属性和静态方法是否可以被继承?是否可以被重写?以及原因?

结论:java 中静态属性和静态方法可以被继承，但是不可以被重写而是被隐藏。

1. 静态方法和属性是属于类的，调用的时候直接通过类名.方法名完成，不需要继承机制就能调用。
2. 多态之所以能够实现依赖于继承、接口和重写、重载\(继承和重写最为关键\)。有了继承和重写就可以实现父类的引用指向不同子类的对象。重写的功能是:"重写"后子类的优先级要高于父类的优先级，但是“隐藏”是没有这个优先级之分。
3. 静态属性、静态方法和非静态的属性都可以被继承和隐藏而不能被重写，因此不能实现多态，不能实现父类的引用可以指向不同子类的对象。非静态方法可以被继承和重写，因此可以实现多态。

### 13. Java 中对象的生命周期

在 Java 中，对象的生命周期包括以下几个阶段:

1. 创建阶段\(Created\)：JVM加载这个类
2. 应用阶段\(In Use\)：至少有一个强引用关系
3. 不可见阶段\(Invisible\)：程序本身不持有这个对象强引用
4. 不可达阶段\(Unreachable\)：该对象不再具有任何强引用了（和不可见相比，不可见可能仍然被其他程序锁持有）；
5. 回收阶段：垃圾收集器对对象空间进行回收，并进行空间再分配

### 14. java 中==和 equals 和 hashCode 的区别?

默认情况下也就是从超类 Object 继承而来的 equals 方法与‘==’是完全等价的，比较的都是对象的内存地址，但我们可以重写 equals 方法，使其按照我们的需求的方式进行比较，如 String 类重写了 equals 方法，使其比较的是字符的序列，而不再是内存地址。在 java 的集合中，判断两个对象是否相等的规则是:

1. 判断两个对象的 hashCode 是否相等。
2. 判断两个对象用 equals 运算是否相等。

此外还有一条，相同的对象hashCode一定相等，hashCode相同的对象不一样相等。

### 15. 类加载的过程，如Person p = new Person\(\)

1. 因为 new 用到了 Person.class，所以会先找到 Person.class 文件，并加载到内存中
2. 执行static方法，如果有的话，给Person类进行初始化操作
3. 在堆内存中开辟空间分配内存地址;
4. 在堆内存中建立对象的特有属性，并进行默认初始化;
5. 对属性进行显示初始化;
6. 对对象进行构造代码块初始化;
7. 对对象进行与之对应的构造函数进行初始化;
8. 将内存地址付给栈内存中的 p 变量。

### 16. 深拷贝和浅拷贝的区别

**浅拷贝**：浅拷贝是创建一个新对象，这个对象有着原始对象属性值的一份精确拷贝。如果属性是基本类型，拷贝的就是基本类型的值，如果属性是引用类型，拷贝的就是内存地址 ，所以**如果其中一个对象改变了这个地址，就会影响到另一个对象**。

**深拷贝**：深拷贝是将一个对象从内存中完整的拷贝一份出来,从堆内存中开辟一个新的区域存放新对象,且**修改新对象不会影响原对象**。

## 网络部分面试题

### 1. HTTP和HTTPS有啥区别？

* `HTTP`是超文本传输协议，信息是明文传输，`HTTPS`则是在HTTP层下加了一层具有安全性的SSL/TLS加密传输协议，要用到CA证书。
* `HTTP`是没有身份认证的，客户端无法知道对方的真实身份。`HTTPS`加入了CA证书，可以确认对方信息。
* `HTTP`默认端口为80，`HTTPS`为443。
* `HTTP`因为明文传输，容易被攻击或者流量劫持。

### 2. Http 1.0，1.1和2.0有哪些区别

#### 2.1 http 1.0和1.1的区别

**1. 缓存处理**，在HTTP1.0中主要使用header里的If-Modified-Since,Expires来做为缓存判断的标准，HTTP1.1则引入了更多的缓存控制策略例如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。

**2. 带宽优化及网络连接的使用**，HTTP1.0中，存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能，HTTP1.1则在请求头引入了range头域，它允许只请求资源的某个部分，即返回码是206（Partial Content），这样就方便了开发者自由的选择以便于充分利用带宽和连接。

**3. 错误通知的管理**，在HTTP1.1中新增了24个错误状态响应码，如409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。

**4. Host头处理**，在HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此，请求消息中的URL并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误（400 Bad Request）。

**5. 长连接**，HTTP 1.1支持长连接（PersistentConnection）和请求的流水线（Pipelining）处理，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟，在HTTP1.1中默认开启Connection： keep-alive，一定程度上弥补了HTTP1.0每次请求都要创建连接的缺点。

#### 2.2 SPDY：HTTP1.x的优化

2012 年 google 如一声惊雷提出了 SPDY 的方案，优化了 HTTP1.X 的请求延迟，解决了HTTP1.X的安全性：

1. **降低延迟**，针对HTTP高延迟的问题，SPDY优雅的采取了多路复用（multiplexing）。多路复用通过多个请求stream共享一个tcp连接的方式，解决了HOL blocking的问题，降低了延迟同时提高了带宽的利用率。
2. **请求优先级**（request prioritization）。多路复用带来一个新的问题是，在连接共享的基础之上有可能会导致关键请求被阻塞。SPDY允许给每个request设置优先级，这样重要的请求就会优先得到响应。比如浏览器加载首页，首页的html内容应该优先展示，之后才是各种静态资源文件，脚本文件等加载，这样可以保证用户能第一时间看到网页内容。
3. **header压缩。**前面提到HTTP1.x的header很多时候都是重复多余的。选择合适的压缩算法可以减小包的大小和数量。
4. **基于HTTPS的加密协议传输**，大大提高了传输数据的可靠性。
5. **服务端推送**（server push），采用了SPDY的网页，例如我的网页有一个sytle.css的请求，在客户端收到sytle.css数据的同时，服务端会将sytle.js的文件推送给客户端，当客户端再次尝试获取sytle.js时就可以直接从缓存中获取到，不用再发请求了。SPDY构成图：

![img](https://user-gold-cdn.xitu.io/2017/8/3/1706626a996d717c9d424646578813c2)

SPDY位于HTTP之下，TCP和SSL之上，这样可以轻松兼容老版本的HTTP协议\(将HTTP1.x的内容封装成一种新的frame格式\)，同时可以使用已有的SSL功能。

#### 2.3 HTTP2.0性能惊人：SPDY的升级版

**新的二进制格式**（Binary Format），HTTP1.x的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合。基于这种考虑HTTP2.0的协议解析决定采用二进制格式，实现方便且健壮。

**多路复用**（MultiPlexing），即连接共享，即每一个request都是是用作连接共享机制的。一个request对应一个id，这样一个连接上可以有多个request，每个连接的request可以随机的混杂在一起，接收方可以根据request的 id将request再归属到各自不同的服务端请求里面。

**header压缩**，如上文中所言，对前面提到过HTTP1.x的header带有大量信息，而且每次都要重复发送，HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一份header fields表，既避免了重复header的传输，又减小了需要传输的大小。

**服务端推送**（server push），同SPDY一样，HTTP2.0也具有server push功能。

### 3. Https 请求慢的解决办法

1. 不通过DNS解析，使用IP直连
2. 解决链路复用的问题，比如使用长连接

### 4. Http 的 request 和 response 的协议组成

#### 1. Request 组成

* 第一部分:请求行，第一行明了是 post 请求，以及 http1.1 版本。
* 第二部分:请求头部
* 第三部分:空行
* 第四部分:请求数据

#### 2. Response的组成

* 第一部分:状态行，由 HTTP 协议版本号， 状态码， 状态消息 三部分组成。
* 第二部分:消息报头，用来说明客户端要使用的一些附加信息
* 第三部分:空行，消息报头后面的空行是必须的
* 第四部分：响应正文

### 5. 说说http缓存

强制缓存:需要服务端参与判断是否继续使用缓存，当客户端第一次请求数据时，服务端返 回了缓存的过期时间\(Expires 与 Cache-Control\)，没有过期就可以继续使用缓存，否则则 不适用，无需再向服务端询问。

对比缓存:需要服务端参与判断是否继续使用缓存，当客户端第一次请求数据时，服务端会将缓存标识\(Last-Modified/If-Modified-Since 与 Etag/If-None-Match\)与数据一起返回给客户端，客户端将两者都备份到缓存中 ，再次请求数据时，客户端将上次备份的缓存 标识发送给服务端，服务端根据缓存标识进行判断， 如果返回 304，则表示通知客户端可以继续使用缓存。 强制缓存优先于对比缓存。

### 6. 长连接。

长连接一般是指TCP长连接，就是保持连接状态，并且可以复用的TCP（三次握手）建立的通道，避免连接解开连接的开销。

下面补充一个面试常见问题：

**socket断线重连，怎么实现，心跳机制又是如何实现的**？

套接字\(socket\)是通信的基石，是支持 TCP/IP 协议的网络通信的基本操作单元。TCP/IP协议提供了socket接口，应用层和传输层，通过socket区分来自不同应用程序或网络连接的通信，实现数据传输。

正常情况下Socket断开需要发一个fin包给服务端，服务端收到fin包之后，才会知道连接断开。

这时候需要重新发起socket连接。（通常情况下可以认为socket连接就是tcp连接）。

**如何确定长连接是否断开？**

1. 客户端向服务端发送心跳包
2. 服务端在一定时间内未收到心跳包，就反向向客户端发送心跳包，确认是否断开。

### 7. Https的加密

https 加密一般分为两种，对称加密和非对称加密（也可以加一个hash算法）。

* 对称加密：加密解密用的是同一个秘钥
* 非对称加密：加密解密用的是公钥和私钥，比如RSA加密算法

**7.1 客户端如何校验 CA 证书?**

1. 客户端拿到CA证书后，利用证书中的公钥解密CA证书中的hash值（这个值是私钥加密后的值），得到Hash-a；
   1. 利用证书内的签名hash算法生成一个hash-b
   2. 比较hash-a和hash-b是否相等，如果相等，表示证书是对的，不相等就是证书不匹配。、
   3. 也会比较一些CA证书的有效期和域名匹配等。

**7.2 HTTPS 中的 SSL 握手建立过程**

1. 客户端和服务端建立 SSL 握手，客户端通过 CA 证书来确认服务端的身份;
2. 互相传递三个随机数，之后通过这随机数来生成一个密钥;
3. 互相确认秘钥然后握手结束
4. 数据通信开始，都使用同一个秘钥来进行加解密。

### 8. 为什么 tcp 要经过三次握手，四次挥手?

![](https://user-gold-cdn.xitu.io/2019/10/8/16da9fd28a45bd19)

四次挥手：

![](https://user-gold-cdn.xitu.io/2019/10/8/16da9fd28b49f652)

### 9. TCP 可靠传输原理实现\(滑动窗口\)

TCP协议作为一个可靠的面向流的传输协议，其可靠性和流量控制由滑动窗口协议保证，而拥塞控制则由控制窗口结合一系列的控制算法实现。

* 确认和重传:接收方收到报文后就会进行确认，发送方一段时间没有收到确认就会重传。 数据校验。
* 数据合理分片与排序：TCP 会对数据进行分片，接收方会缓存为按序到达的数据，重新排序 后再提交给应用层。
* 流程控制:当接收方来不及接收发送的数据时，则会提示发送方降低发送的速度，防止包丢失。
* 拥塞控制:当网络发生拥塞时，减少数据的发送

### 10. Tcp 和 Udp 的区别?

|  | UDP | TCP |
| :--- | :--- | :--- |
| 是否基于连接 | 无连接 | 面向连接 |
| 是否可靠 | 不可靠传输，不使用流量控制和拥塞控制 | 可靠传输，使用流量控制和拥塞控制 |
| 连接对象个数 | 支持一对一，一对多，多对一和多对多交互通信 | 只能是一对一通信 |
| 传输方式 | 面向报文 | 面向字节流 |
| 是否保证顺序 | 不保证 | 保证数据顺序 |
| 适用场景 | 适用于实时应用（IP电话、视频会议、直播等） | 适用于要求可靠传输的应用，例如文件传输 |

**如何设计在 UDP 上层保证 UDP 的可靠性传输?**

* 添加 seq/ack 机制，确保数据发送到对端
* 添加发送和接收缓冲区，主要是用户超时重传。 
* 添加超时重传机制。

### 10. 浏览器输入地址到返回结果发生了什么?

1. DNS解析
2. TCP连接
3. 发送HTTP请求
4. 服务器处理请求并返回HTTP报文
5. 浏览器解析渲染页面
6. 连接结束

