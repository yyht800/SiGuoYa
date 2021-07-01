# Android 中Handler机制的总结

## 1.Handler机制的基本原理

Android有大量的消息驱动方式来进行交互，比如Android的四剑客`Activity`, `Service`, `Broadcast`, `ContentProvider`的启动过程的交互，都离不开消息机制，Android某种意义上也可以说成是一个以消息驱动的系统。消息机制涉及MessageQueue/Message/Looper/Handler这4个类。

主要功能如下：

* **Message**：消息分为硬件产生的消息\(如按钮、触摸\)和软件生成的消息；
* **MessageQueue**：消息队列的主要功能向消息池投递消息\(`MessageQueue.enqueueMessage`\)和取走消息池的消息\(`MessageQueue.next`\)；
* **Handler**：消息辅助类，主要功能向消息池发送各种消息事件\(`Handler.sendMessage`\)和处理相应消息事件\(`Handler.handleMessage`\)；
* **Looper**：不断循环执行\(`Looper.loop`\)，按分发机制将消息分发给目标处理者。

![](https://user-gold-cdn.xitu.io/2019/2/26/16927e6099e1d48c)

## 2. Looper

### 2.1 prepare（）

prepare方法主要是创建一个Looper对象，对于无参的情况，默认调用`prepare(true)`，表示的是这个Looper允许退出，而对于false的情况则表示当前Looper不允许退出。

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

另外，与prepare\(\)相近功能的，还有一个`prepareMainLooper()`方法，该方法主要在ActivityThread类中使用。

```java
public static void prepareMainLooper() {
    prepare(false); //设置不允许退出的Looper
    synchronized (Looper.class) {
        //将当前的Looper保存为主Looper，每个线程只允许执行一次。
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

### 2.2 ThreadLocal

**ThreadLocal**： 线程本地存储区（Thread Local Storage，简称为TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。TLS常用的操作方法：

* `ThreadLocal.set(T value)`：将value存储到当前线程的TLS区域，源码如下：

```java
public void set(T value) {
    Thread currentThread = Thread.currentThread(); //获取当前线程
    Values values = values(currentThread); //查找当前线程的本地储存区
    if (values == null) {
        //当线程本地存储区，尚未存储该线程相关信息时，则创建Values对象
        values = initializeValues(currentThread);
    }
    //保存数据value到当前线程this
    values.put(this, value);
}
```

* `ThreadLocal.get()`：获取当前线程TLS区域的数据，源码如下：

```java
public T get() {
    Thread currentThread = Thread.currentThread(); //获取当前线程
    Values values = values(currentThread); //查找当前线程的本地储存区
    if (values != null) {
        Object[] table = values.table;
        int index = hash & values.mask;
        if (this.reference == table[index]) {
            return (T) table[index + 1]; //返回当前线程储存区中的数据
        }
    } else {
        //创建Values对象
        values = initializeValues(currentThread);
    }
    return (T) values.getAfterMiss(this); //从目标线程存储区没有查询是则返回null
}
```

ThreadLocal的get\(\)和set\(\)方法操作的类型都是泛型，接着回到前面提到的`sThreadLocal`变量，其定义如下：

```text
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>()
```

可见`sThreadLocal`的get\(\)和set\(\)操作的类型都是`Looper`类型。

### 2.3 loop（）

```java
public static void loop() {
    final Looper me = myLooper();  //获取TLS存储的Looper对象 
    final MessageQueue queue = me.mQueue;  //获取Looper对象中的消息队列

    Binder.clearCallingIdentity();
    //确保在权限检查时基于本地进程，而不是调用进程。
    final long ident = Binder.clearCallingIdentity();

    for (;;) { //进入loop的主循环方法
        Message msg = queue.next(); //可能会阻塞 
        if (msg == null) { //没有消息，则退出循环
            return;
        }

        //默认为null，可通过setMessageLogging()方法来指定输出，用于debug功能
        Printer logging = me.mLogging;  
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg); //用于分发Message 
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        //恢复调用者信息
        final long newIdent = Binder.clearCallingIdentity();
        msg.recycleUnchecked();  //将Message放入消息池 
    }
```

loop\(\)进入循环模式，不断重复下面的操作，直到没有消息时退出循环

* 读取MessageQueue的下一条Message；
* 把Message分发给相应的target；
* 再把分发后的Message回收到消息池，以便重复利用。

这是这个消息处理的核心部分。另外，上面代码中可以看到有logging方法，这是用于debug的，默认情况下`logging == null`，通过设置setMessageLogging\(\)用来开启debug工作。

### 2.4 quit（）

Looper.quit\(\)方法的实现最终调用的是MessageQueue.quit\(\)方法

```java
public void quit() {
    mQueue.quit(false); //消息移除
}

public void quitSafely() {
    mQueue.quit(true); //安全地消息移除
}
```

```java
void quit(boolean safe) {
        // 当mQuitAllowed为false，表示不运行退出，强行调用quit()会抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        synchronized (this) {
            if (mQuitting) { //防止多次执行退出操作
                return;
            }
            mQuitting = true;
            if (safe) {
                removeAllFutureMessagesLocked(); //移除尚未触发的所有消息
            } else {
                removeAllMessagesLocked(); //移除所有的消息
            }
            //mQuitting=false，那么认定为 mPtr != 0
            nativeWake(mPtr);
        }
    }
```

消息退出的方式：

* 当safe =true时，只移除尚未触发的所有消息，对于正在触发的消息并不移除；
* 当safe =flase时，移除所有的消息

## 3. Handler

### 3.1 创建Handler

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    //匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
    mLooper = Looper.myLooper();  //从当前线程的TLS中获取Looper对象【见2.1】
    if (mLooper == null) {
        throw new RuntimeException("");
    }
    mQueue = mLooper.mQueue; //消息队列，来自Looper对象
    mCallback = callback;  //回调方法
    mAsynchronous = async; //设置消息是否为异步处理方式
}
```

上述的代码中可以看出，创建handler之前Loop必须要创建也就是Loop.perpare\(\)方法的执行。

### 3.2 消息分发机制

在Looper.loop\(\)中，当发现有消息时，调用消息的目标handler，执行dispatchMessage\(\)方法来分发消息。

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //当Message存在回调方法，回调msg.callback.run()方法；
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //当Handler存在Callback成员变量时，回调方法handleMessage()；
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //Handler自身的回调方法handleMessage()
        handleMessage(msg);
    }
}
```

**分发消息流程：**

1. 当`Message`的回调方法不为空时，则回调方法`msg.callback.run()`，其中callBack数据类型为Runnable,否则进入步骤2；
2. 当`Handler`的`mCallback`成员变量不为空时，则回调方法`mCallback.handleMessage(msg)`,否则进入步骤3；
3. 调用`Handler`自身的回调方法`handleMessage()`，该方法默认为空，Handler子类通过覆写该方法来完成具体的逻辑。

对于很多情况下，消息分发后的处理方法是第3种情况，即Handler.handleMessage\(\)，一般地往往通过覆写该方法从而实现自己的业务逻辑。

`Handler`是消息机制中非常重要的辅助类，更多的实现都是`MessageQueue`, `Message`中的方法，Handler的目的是为了更加方便的使用消息机制。

## 4. MessageQueue

MessageQueue是消息机制的Java层和C++层的连接纽带，大部分核心方法都交给native层来处理，其中MessageQueue类中涉及的native方法如下：

```java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

### 4.1 nativeInit（）

![](http://gityuan.com/images/handler/native_init.png)

下面来进一步来看看调用链的过程：

**【1】 new MessageQueue\(\)**

==&gt; MessageQueue.java

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();  //mPtr记录native消息队列的信息 【2】
}
```

**【2】android\_os\_MessageQueue\_nativeInit\(\)**

==&gt; android\_os\_MessageQueue.cpp

```java
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    //初始化native消息队列 【3】
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    nativeMessageQueue->incStrong(env); //增加引用计数
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

此处reinterpret\_cast是C++里的强制类型转换符

**【3】new NativeMessageQueue\(\)**

==&gt; android\_os\_MessageQueue.cpp

```java
NativeMessageQueue::NativeMessageQueue()
            : mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {

    mLooper = Looper::getForThread(); //获取TLS中的Looper对象
    if (mLooper == NULL) {
        mLooper = new Looper(false); //创建native层的Looper 【4】
        Looper::setForThread(mLooper); //保存native层的Looper到TLS
    }
}
```

* Looper::getForThread\(\)，功能类比于Java层的Looper.myLooper\(\);
* Looper::setForThread\(mLooper\)，功能类比于Java层的ThreadLocal.set\(\);

此处Native层的Looper与Java层的Looper没有任何的关系，只是在Native层重实现了一套类似功能的逻辑。

**【4】new Looper\(\)**

==&gt; Looper.cpp

```java
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK); //构造唤醒事件的fd
    AutoMutex _l(mLock);
    rebuildEpollLocked();  //重建Epoll事件【5】
}
```

**【5】epoll\_create/epoll\_ctl**

==&gt; Looper.cpp

```java
void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        close(mEpollFd); //关闭旧的epoll实例
    }
    mEpollFd = epoll_create(EPOLL_SIZE_HINT); //创建新的epoll实例，并注册wake管道
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); //把未使用的数据区域进行置0操作
    eventItem.events = EPOLLIN; //可读事件
    eventItem.data.fd = mWakeEventFd;
    //将唤醒事件(mWakeEventFd)添加到epoll实例(mEpollFd)
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        //将request队列的事件，分别添加到epoll实例
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
    }
}
```

关于epoll的原理以及为什么选择epoll的方式，可查看文章[select/poll/epoll对比分析](http://gityuan.com/2015/12/06/linux_epoll/)。

Looper对象中的mWakeEventFd添加到epoll监控，以及mRequests也添加到epoll的监控范围内。

### 4.2 next（）

```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) { //当消息循环已经退出，则直接返回
        return null;
    }
    int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //当消息的Handler为空时，则查询异步消息
            if (msg != null && msg.target == null) {
                //当查询到异步消息，则立刻退出循环
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    //设置消息的使用状态，即flags |= FLAG_IN_USE
                    msg.markInUse();
                    return msg;   //成功地获取MessageQueue中的下一条即将要执行的消息
                }
            } else {
                //没有消息
                nextPollTimeoutMillis = -1;
            }
            //消息正在退出，返回null
            if (mQuitting) {
                dispose();
                return null;
            }
            //当消息队列为空，或者是消息队列的第一个消息时
            if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //没有idle handlers 需要运行，则循环并等待。
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; //去掉handler的引用
            boolean keep = false;
            try {
                keep = idler.queueIdle();  //idle时执行的方法
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        //重置idle handler个数为0，以保证不会再次重复运行
        pendingIdleHandlerCount = 0;
        //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
        nextPollTimeoutMillis = 0;
    }
}
```

`nativePollOnce`是阻塞操作，其中`nextPollTimeoutMillis`代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。

当处于空闲时，往往会执行`IdleHandler`中的方法。当nativePollOnce\(\)返回后，next\(\)从`mMessages`中提取一个消息。

### 4.3 nativePollOnce（）

nativePollOnce用于提取消息队列中的消息，提取消息的调用链，如下：

![](http://gityuan.com/images/handler/poll_once.png)

下面来进一步来看看调用链的过程：

**【1】MessageQueue.next\(\)**

==&gt; MessageQueue.java

```text
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis); //阻塞操作 【2】
        ...
    }
```

**【2】android\_os\_MessageQueue\_nativePollOnce\(\)**

==&gt; android\_os\_MessageQueue.cpp

```text
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    //将Java层传递下来的mPtr转换为nativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis); 【3】
}
```

**【3】NativeMessageQueue::pollOnce\(\)**

==&gt; android\_os\_MessageQueue.cpp

```text
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis); 【4】
    mPollObj = NULL;
    mPollEnv = NULL;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

**【4】Looper::pollOnce\(\)**

==&gt; Looper.h

```text
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, NULL, NULL, NULL); 【5】
}
```

**【5】 Looper::pollOnce\(\)**

==&gt; Looper.cpp

```text
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // 先处理没有Callback方法的 Response事件
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) { //ident大于0，则表示没有callback, 因为POLL_CALLBACK = -2,
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        // 再处理内部轮询
        result = pollInner(timeoutMillis); 【6】
    }
}
```

参数说明：

* timeoutMillis：超时时长
* outFd：发生事件的文件描述符
* outEvents：当前outFd上发生的事件，包含以下4类事件
  * EVENT\_INPUT 可读
  * EVENT\_OUTPUT 可写
  * EVENT\_ERROR 错误
  * EVENT\_HANGUP 中断
* outData：上下文数据

**【6】Looper::pollInner\(\)**

==&gt; Looper.cpp

```text
int Looper::pollInner(int timeoutMillis) {
    ...
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;
    mPolling = true; //即将处于idle状态
    struct epoll_event eventItems[EPOLL_MAX_EVENTS]; //fd最大个数为16
    //等待事件发生或者超时，在nativeWake()方法，向管道写端写入字符，则该方法会返回；
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    mPolling = false; //不再处于idle状态
    mLock.lock();  //请求锁
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();  // epoll重建，直接跳转Done;
        goto Done;
    }
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = POLL_ERROR; // epoll事件个数小于0，发生错误，直接跳转Done;
        goto Done;
    }
    if (eventCount == 0) {  //epoll事件个数等于0，发生超时，直接跳转Done;
        result = POLL_TIMEOUT;
        goto Done;
    }

    //循环遍历，处理所有的事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken(); //已经唤醒了，则读取并清空管道数据【7】
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //处理request，生成对应的reponse对象，push到响应数组
                pushResponse(events, mRequests.valueAt(requestIndex));
            }
        }
    }
Done: ;
    //再处理Native的Message，调用相应回调方法
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            {
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();  //释放锁
                handler->handleMessage(message);  // 处理消息事件
            }
            mLock.lock();  //请求锁
            mSendingMessage = false;
            result = POLL_CALLBACK; // 发生回调
        } else {
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    mLock.unlock(); //释放锁

    //处理带有Callback()方法的Response事件，执行Reponse相应的回调方法
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            // 处理请求的回调方法
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq); //移除fd
            }
            response.request.callback.clear(); //清除reponse引用的回调方法
            result = POLL_CALLBACK;  // 发生回调
        }
    }
    return result;
}
```

pollOnce返回值说明：

* POLL\_WAKE： 表示由wake\(\)触发，即pipe写端的write事件触发；
* POLL\_CALLBACK： 表示某个被监听fd被触发。
* POLL\_TIMEOUT： 表示等待超时；
* POLL\_ERROR：表示等待期间发生错误；

**【7】Looper::awoken\(\)**

```text
void Looper::awoken() {
    uint64_t counter;
    //不断读取管道数据，目的就是为了清空管道内容
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}
```

**poll小结**

pollInner\(\)方法的处理流程：

1. 先调用epoll\_wait\(\)，这是阻塞方法，用于等待事件发生或者超时；
2. 对于epoll\_wait\(\)返回，当且仅当以下3种情况出现：
   * POLL\_ERROR，发生错误，直接跳转到Done；
   * POLL\_TIMEOUT，发生超时，直接跳转到Done；
   * 检测到管道有事件发生，则再根据情况做相应处理：
     * 如果是管道读端产生事件，则直接读取管道的数据；
     * 如果是其他事件，则处理request，生成对应的reponse对象，push到reponse数组；
3. 进入Done标记位的代码段：
   * 先处理Native的Message，调用Native 的Handler来处理该Message;
   * 再处理Response数组，POLL\_CALLBACK类型的事件；

从上面的流程，可以发现对于Request先收集，一并放入reponse数组，而不是马上执行。真正在Done开始执行的时候，是先处理native Message，再处理Request，说明native Message的优先级高于Request请求的优先级。

另外pollOnce\(\)方法中，先处理Response数组中不带Callback的事件，再调用了pollInner\(\)方法。

### 4.4 enqueueMessage（）

添加一条消息到消息队列

```java
boolean enqueueMessage(Message msg, long when) {
    // 每一个普通Message必须有一个target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {  //正在退出时，回收msg，加入到消息池
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; //当阻塞时需要唤醒
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
            //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p;
            prev.next = msg;
        }
        //消息没有退出，我们认为此时mPtr != 0
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

`MessageQueue`是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

### 4.5 nativeWake（）

nativeWake用于唤醒功能，在添加消息到消息队列`enqueueMessage()`, 或者把消息从消息队列中全部移除`quit()`，再有需要时都会调用 `nativeWake`方法。包含唤醒过程的添加消息的调用链，如下：

![native\_wake](http://gityuan.com/images/handler/native_wake.png)

下面来进一步来看看调用链的过程：

**【1】MessageQueue.enqueueMessage\(\)**

==&gt; MessageQueue.java

```text
boolean enqueueMessage(Message msg, long when) {
    ... //将Message按时间顺序插入MessageQueue
    if (needWake) {
            nativeWake(mPtr); 【2】
        }
}
```

往消息队列添加Message时，需要根据mBlocked情况来决定是否需要调用nativeWake。

**【2】android\_os\_MessageQueue\_nativeWake\(\)**

==&gt; android\_os\_MessageQueue.cpp

```text
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake(); 【3】
}
```

**【3】NativeMessageQueue::wake\(\)**

==&gt; android\_os\_MessageQueue.cpp

```text
void NativeMessageQueue::wake() {
    mLooper->wake();  【4】
}
```

**【4】Looper::wake\(\)**

==&gt; Looper.cpp

```text
void Looper::wake() {
    uint64_t inc = 1;
    // 向管道mWakeEventFd写入字符1
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

其中`TEMP_FAILURE_RETRY` 是一个宏定义， 当执行`write`失败后，会不断重复执行，直到执行成功为止。‘

## 5. 常见面试题

### **5.1 View.post 和 Handler.post 的区别**

我们最常用的 Handler 功能就是 Handler.post，除此之外，还有 View.post 也经常会用到，那么这两个有什么区别呢

我们先看下 View.post 的代码。

```java
// View.java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }

    // Postpone the runnable until we know on which thread it needs to run.
    // Assume that the runnable will be successfully placed after attach.
    getRunQueue().post(action);
    return true;
}
```

通过代码来看，如果 AttachInfo 不为空，则通过 handler 去执行，如果 handler 为空，则通过 RunQueue 去执行。

那我们先看看这里的 AttachInfo 是什么。

这个就需要追溯到 ViewRootImpl 的流程里了，我们先看下面这段代码。

```java
// ViewRootImpl.java
final ViewRootHandler mHandler = new ViewRootHandler();

public ViewRootImpl(Context context, Display display) {
    // ...
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
            context);
}

private void performTraversals() {
    final View host = mView;
    // ...
    if (mFirst) {
        host.dispatchAttachedToWindow(mAttachInfo, 0);
        mFirst = false;
    }
    // ...
}
```

代码写了一些关键部分，在 ViewRootImpl 构造函数里，创建了 mAttachInfo，然后在 performTraversals 里，如果 mFirst 为 true，则调用 host.dispatchAttachedToWindow。

这里的 host 就是 DecorView，如果有读者朋友对这里不太清楚，可以看看前面【面试官带你学安卓-从View的绘制流程】说起这篇文章复习一下。

这里还有一个知识点就是 mAttachInfo 中的 mHandler 其实是 ViewRootImpl 内部的 ViewRootHandler。

然后就调用到了 DecorView.dispatchAttachedToWindow，其实就是 ViewGroup 的 dispatchAttachedToWindow。

一般 ViewGroup 中相关的方法，都是去依次调用 child 的对应方法。

这个也不例外，依次调用子 View 的 dispatchAttachedToWindow，把 AttachInfo 传进去，在 子 View 中给 mAttachInfo 赋值。

```java
// ViewGroup
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mGroupFlags |= FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
    super.dispatchAttachedToWindow(info, visibility);
    mGroupFlags &= ~FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;

    final int count = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < count; i++) {
        final View child = children[i];
        child.dispatchAttachedToWindow(info,
                combineVisibility(visibility, child.getVisibility()));
    }
    final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
    for (int i = 0; i < transientCount; ++i) {
        View view = mTransientViews.get(i);
        view.dispatchAttachedToWindow(info,
                combineVisibility(visibility, view.getVisibility()));
    }
}

// View
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    // ...
}
```

看到这里，大家可能忘记我们开始刚刚要做什么了。

我们是在看 View.post 的流程，再回顾一下 View.post 的代码：

```java
// View.java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }

    getRunQueue().post(action);
    return true;
}
```

现在我们知道 attachInfo 是什么了，是 ViewRootImpl 首次触发 performTraversals 传进来的，也就是触发 performTraversals 之后，View.post 都是通过 ViewRootImpl 内部的 Handler 进行处理的。

如果在 performTraversals 之前或者 mAttachInfo 置为空以后进行执行，则通过 RunQueue 进行处理。

那我们再看看 getRunQueue\(\).post\(action\); 做了些什么事情。这里的 RunQueue 其实是 HandlerActionQueue。

HandlerActionQueue 的代码看一下。

```java
public class HandlerActionQueue {
    public void post(Runnable action) {
        postDelayed(action, 0);
    }

    public void postDelayed(Runnable action, long delayMillis) {
        final HandlerAction handlerAction = new HandlerAction(action, delayMillis);

        synchronized (this) {
            if (mActions == null) {
                mActions = new HandlerAction[4];
            }
            mActions = GrowingArrayUtils.append(mActions, mCount, handlerAction);
            mCount++;
        }
    }

    public void executeActions(Handler handler) {
        synchronized (this) {
            final HandlerAction[] actions = mActions;
            for (int i = 0, count = mCount; i < count; i++) {
                final HandlerAction handlerAction = actions[i];
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }

            mActions = null;
            mCount = 0;
        }
    }
}
```

通过上面的代码我们可以看到，执行 getRunQueue\(\).post\(action\); 其实是将代码添加到 mActions 进行保存，然后在 executeActions 的时候进行执行。

executeActions 执行的时机只有一个，就是在 dispatchAttachedToWindow\(AttachInfo info, int visibility\) 里面调用的。

```java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    if (mRunQueue != null) {
        mRunQueue.executeActions(info.mHandler);
        mRunQueue = null;
    }
}
```

看到这里我们就知道了，View.post 和 Handler.post 的区别就是：

1. 如果在 performTraversals 前调用 View.post，则会将消息进行保存，之后在 dispatchAttachedToWindow 的时候通过 ViewRootImpl 中的 Handler 进行调用。
2. 如果在 performTraversals 以后调用 View.post，则直接通过 ViewRootImpl 中的 Handler 进行调用。

这里我们又可以回答一个问题了，就是为什么 View.post 里可以拿到 View 的宽高信息呢？

因为 View.post 的 Runnable 执行的时候，已经执行过 performTraversals 了，也就是 View 的 measure layout draw 方法都执行过了，自然可以获取到 View 的宽高信息了。

## 参考文章

1. [Handler 都没搞懂，拿什么去跳槽啊？](https://juejin.cn/post/6844903783139393550#heading-17)
2. [Android消息机制系列 1-3](http://gityuan.com/2015/12/26/handler-message-framework/)
3. 大厂资深面试官-带你破解Android高级面试第7章

