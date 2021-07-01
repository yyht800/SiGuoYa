# 如果面试遇到这些问题该如何回答？

## 一、Android 基础

### 1.什么是 ANR 如何避免它?

A: ANR是指应用程序未响应，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR。ANR由消息处理机制保证，Android在系统层实现了一套精密的机制来发现ANR，核心原理是消息调度和超时处理。

然后我们来看什么样的情况会导致ANR异常，总结起来就是四大组件的响应超时：

（1）Service Timeout:Service在特定的时间内无法处理完成（20s均为前台） （2）BroadcastQueue Timeout：BroadcastReceiver在特定时间内无法处理完成（10s） （3）ContentProvider Timeout：内容提供者执行超时 （4）inputDispatching Timeout: 按键或触摸事件在特定时间内无响应。（5s）

具体的的场景如：

1. 应用在主线程上非常缓慢地执行涉及 I/O 的操作。
2. 应用在主线程上进行长时间的计算。
3. 主线程在对另一个进程进行同步 binder 调用，而后者需要很长时间才能返回。
4. 主线程处于阻塞状态，为发生在另一个线程上的长操作等待同步的块。
5. 主线程在进程中或通过 binder 调用与另一个线程之间发生死锁。主线程不只是在等待长操作执行完毕，而且处于死锁状态。如需更多信息，请参阅维基百科上的[死锁](https://en.wikipedia.org/wiki/Deadlock)。

**解决方式**：

将所有的耗时操作都放在子线程去处理，保证UI的顺畅。

### 2. Activity和fragment的生命周期？

A：Activity的生命周期耳熟能详，如下所示：

![](https://upload-images.jianshu.io/upload_images/2244681-1532340d63d59dc6.png)

Activity是四大组件之一，除了上述的生命周期之外额外需要注意的还有：

1. onNewIntent，栈内复用
2. onActivityResult 跳转到另一个Activity后，返回到之前的Activity会调用
3. onSaveInstanceState 和onRestoreInstanceState 当系统意外停止activity时，它会调用onSaveInstanceState\(\)存储一个Bundle 对象并将数据传递给onCreate和onRestoreInstanceState。由于该方法不一定会被调用，这里最多只能做一些临时性数据的存储，持久化操作还是建议放在必定会调用的生命周期中。

下面是Fragmet 的生命周期：

![](https://upload-images.jianshu.io/upload_images/2244681-3685a0866eb07d3a.png)

fragment的生命周期中需要注意的方法：

1. onAttach：onAttach\(\)回调将在Fragment与其Activity关联之后调用。
2. onCreate\(Bundle savedInstanceState\)：此时的Fragment的onCreat回调时，该fragmet还没有获得Activity的onCreate\(\)已完成的通知，所以不能将依赖于Activity视图层次结构存在性的代码放入此回调方法中。在onCreate\(\)回调方法中，我们应该尽量避免耗时操作。此时的bundle就可以获取到activity传来的参数
3. onCreateView\(LayoutInflater inflater, ViewGroup container,

   Bundle savedInstanceState\)： 其中的Bundle为状态包与上面的bundle不一样。

   **注意的是：不要将视图层次结构附加到传入的ViewGroup父元素中，该关联会自动完成。如果在此回调中将碎片的视图层次结构附加到父元素，很可能会出现异常。**

4. onActivityCreated：onActivityCreated\(\)回调会在Activity完成其onCreate\(\)回调之后调用。在调用onActivityCreated\(\)之前，Activity的视图层次结构已经准备好了，**这是在用户看到用户界面之前你可对用户界面执行的最后调整的地方。**
5. Fragment与Activity相同生命周期调用：接下来的onStart\(\)\onResume\(\)\onPause\(\)\onStop\(\)回调方法将和Activity的回调方法进行绑定，也就是说与Activity中对应的生命周期相同，因此不做过多介绍。
6. onDestroyView:该回调方法在视图层次结构与Fragment分离之后调用。
7. onDestroy：不再使用Fragment时调用。（备注：Fragment仍然附加到Activity并任然可以找到，但是不能执行其他操作）
8. onDetach：Fragme生命周期最后回调函数，调用后，Fragment不再与Activity绑定，释放资源。
9. setRetainInstance（true），是可以在Activity重新创建时可以不完全销毁Fragment，以便Fragment可以恢复

**总结**：关于Activity和Fragment的生命周期图，如下所示：

![](https://imgs.piasy.com/2018-03-23-2017010890963complete_android_fragment_lifecycle.png)

### 3.AsyncTask 的原理，以及它的缺陷和问题。

### 3.1 AsyncTask 的原理

AsyncTask 是由 两 个 线 程 池 \( SerialExecutor 和THREAD\_POOL\_EXECUTOR\) 和一个Handler\(InternalHandler\)，其中线程池 SerialExecutor 用于任务的排队，而线程池THREAD\_POOL\_EXECUTOR 用于真正地执行任务，InternalHandler 用于 将执行环境从线程池切换到主线程。

其中sHandler 是一个静态的 Handler 对象，为了能够将环境切换到主线程，这就要求 sHandler 这个对象必须在主线程创建。由于静态成员会在 加载类的时候进行初始化，因此这就变相要求 AsyncTask 的类必须在主线程中加载，否则同一个进程中的 AsyncTask 都将无法正常工作。

**总结**：AsyncTask是由两个线程池组成，其中一个用于任务的排队，另一个真正的执行任务，并通过InternalHandler将执行完的结果，切换回主线程。同时，他的创建需要在主线程中。

### 3.2 一些问题

**3.2.1 生命周期**

AsynTask在主线程创建后，不会随着Activity的销毁而自动销毁，当我们的Activity销毁后，如果AsynTask没有被销毁，可能会导致crash，如它所持有的view已经被销毁了。所以我们在Activity或者view销毁前，取消对应的任务。

**3.2.2 内存泄漏**

如果 AsyncTask 被声明为 Activity 的非静态内部类，那么 AsyncTask 会保留一个 对 Activity 的引用。如果Activity需要被销毁，但是AsyncTask仍在执行任务，那么它将持有一个Activity的引用，导致Activity无法被回收。

**3.2.3 结果丢失**

屏幕旋转或 Activity 在后台被系统杀掉等情况会导致 Activity 的重新创建，之前 运行的 AsyncTask 会持有一个之前 Activity 的引用，这个引用已经无效，这时调 用 onPostExecute\(\)再去更新界面将不再生效。

**3.2.4 并行还是串行**

在 Android1.6 之前的版本，AsyncTask 是串行的，在 1.6 之后的版本，采用线程池处理并行任务，但是从 Android 3.0 开始，为了避免 AsyncTask 所带来的并发错误，又采用一个线程来串行执行任务。可以使用 executeOnExecutor\(\)方法来 并行地执行任务。

### 4.android 中进程的优先级?

**1.** **前台进程**: 即与用户正在交互的 Activity 或者 Activity 用到的 Service 等，如果系统内存不足时前台进程是最晚被杀死的

**2.** **可见进程**:

可以是处于暂停状态\(onPause\)的 Activity 或者绑定在其上的 Service，即被用户 可见，但由于失了焦点而不能与用户交互

**3.** **服务进程**: 其中运行着使用 startService 方法启动的 Service，虽然不被用户可见，但是却是用户关心的，例如用户正在非音乐界面听的音乐或者正在非下载页面下载的文件 等;当系统要空间运行，前两者进程才会被终止

**4.** **后台进程**: 其中运行着执行 onStop 方法而停止的程序，但是却不是用户当前关心的，例如后台挂着的 QQ，这时的进程系统一旦没了有内存就首先被杀死

**5.** **空进程**

不包含任何应用程序的进程，这样的进程系统是一般不会让他存在的

### 5. Bundle 传递对象为什么需要序列化?Serialzable 和 Parcelable 的区别?

bundle在传递的过程中，只支持基础数据类型，所以在传递对象时需要将对象转换成可存储可传输的字节流。序列化的对象可以在组件间（IPC）、网络等情况下传输。

* **Serializable**

  Java自带的一个序列化工具，直接将对象进行序列化

* **Parcelable**

  专门为Android设计的一个序列化工具，将对象拆解，拆解后的每一部分都是Intent所支持的数据类型。

### 6. Android中的Context

## 本周总结

1. 基于复习计划的准备，其实执行的不够严格
2. 算法上的问题，过了一段时间之后思路会比较迟钝，需要更好的算法规划思路和刷题相结合
3. 文章的总结，推进和执行的不够彻底，导致关于学习笔记的进度，一直不是很快
4. 每周保持起码一个章节的梳理来推进学习计划
5. 同时推进的项目太多，不再拓展新的学习计划，贪多嚼不烂。

学习上的进度，不喜不忧，但是执行上可以更严格一些，学习计划线上需要进行收束。汇聚焦点

