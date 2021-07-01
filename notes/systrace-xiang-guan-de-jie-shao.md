# Systrace相关的介绍

## 1. 概述

Systrace 是 Android4.1 中新增的性能数据采样和分析工具。它可帮助开发者收集 Android 关键子系统（如 SurfaceFlinger/SystemServer/Kernel/Input/Display 等 Framework 部分关键模块、服务，View系统等）的运行信息，从而帮助开发者更直观的分析系统瓶颈，改进性能。

Systrace 的功能包括跟踪系统的 I/O 操作、内核工作队列、CPU 负载以及 Android 各个子系统的运行状况等。在 Android 平台中，它主要由3部分组成：

* **内核部分**：Systrace 利用了 Linux Kernel 中的 ftrace 功能。所以，如果要使用 Systrace 的话，必须开启 kernel 中和 ftrace 相关的模块。
* **数据采集部分**：Android 定义了一个 Trace 类。应用程序可利用该类把统计信息输出给ftrace。同时，Android 还有一个 atrace 程序，它可以从 ftrace 中读取统计信息然后交给数据分析工具来处理。
* **数据分析工具**：Android 提供一个 systrace.py（ python 脚本文件，位于 Android SDK目录/platform-tools/systrace 中，其内部将调用 atrace 程序）用来配置数据采集的方式（如采集数据的标签、输出文件名等）和收集 ftrace 统计数据并生成一个结果网页文件供用户查看。 从本质上说，Systrace 是对 Linux Kernel中 ftrace 的封装。应用进程需要利用 Android 提供的 Trace 类来使用 Systrace.

  关于 Systrace 的官方介绍和使用可以看这里：[Systrace](http://developer.android.com/tools/help/systrace.html)

## 2. Android中常说的60FPS

先梳理几个概念：

1. 60 fps 的意思是说，画面每秒更新60次
2. 这60次更新，是要均匀更新的，不是说一会快，一会慢，那样视觉上也会觉得不流畅
3. **屏幕刷新率** ：是针对硬件的概念，Android中大部分机型都是60 HZ，对于移动设备来说，刷新率的提高会带来功耗增大。

**小问题：我们看的电影在24帧左右，为什么不觉得卡顿**

可以参考下面的链接，大致是因为电影有一个动态模糊的效果（将位移信息，像素变化填充到一帧的中，即时在低帧率的情况下，大脑也会自己进行补充），并且电影还限制了观众的视角，以便动态模糊效果达到最佳状态。

而游戏中，需要一个实时反馈的机制，并且游戏玩家在游戏过程中，视角变化幅度很大，导致动态模糊效果并不能发挥正常作用，导致很多游戏低于40帧的时候基本不能玩。

[玩游戏为何要60帧才流畅，电影却只需24帧](https://www.youtube.com/watch?v=--OKrYxOb6Y)

## 3. Andorid中RenderThread

RenderThread也就是我们常说的渲染线程直接看总结了：

1. 将Main Thread维护的Display List同步到Render Thread维护的Display List去。这个同步过程由Render Thread执行，但是Main Thread会被阻塞住。
2. 如果能够完全地将Main Thread维护的Display List同步到Render Thread维护的Display List去，那么Main Thread就会被唤醒，此后Main Thread和Render Thread就互不干扰，各自操作各自内部维护的Display List；否则的话，Main Thread就会继续阻塞，直到Render Thread完成应用程序窗口当前帧的渲染为止。
3. Render Thread在渲染应用程序窗口的Root Render Node的Display List之前，首先将那些设置了Layer的子Render Node的Display List渲染在各自的一个FBO（Frame Buffer Object）上，接下来再一起将这些FBO以及那些没有设置Layer的子Render Node的Display List一起渲染在Frame Buffer之上，也就是渲染在从Surface Flinger请求回来的一个图形缓冲区上。这个图形缓冲区最终会被提交给Surface Flinger合并以及显示在屏幕上。

其中第二步中的DisplayList同步很关键，它使得Main Thread和Render Thread可以并行执行，这意味着Render Thread在渲染应用程序窗口当前帧的Display List的同时，Main Thread可以去准备应用程序窗口下一帧的Display List，这样就使得应用程序窗口的UI更流畅。

## 4. Vsync和Choreographer

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

## 5. SurfaceFlinger

SurfaceFlinger作为负责绘制应用UI的核心，从名字可以看出**其功能是将所有Surface合成工作**。 不论使用什么渲染API, 所有的东西最终都是渲染到”surface”. surface代表BufferQueue的生产者端, 并且 由SurfaceFlinger所消费, 这便是基本的生产者-消费者模式. Android平台所创建的Window都由surface所支持, 所有可见的surface渲染到显示设备都是通过SurfaceFlinger来完成的.

对于大多数的app来说都有3个layers: 状态栏,导航栏, 应用UI. 每一个layer都是独立更新的. 状态栏和导航栏是由系统进程负责渲染, app层是由app自己渲染,两者直接并没有协作.

SurfaceFlinger进程是由init进程创建的，运行在独立的SurfaceFlinger进程。Android应用程序 必须跟SurfaceFlinger进程交互，才能完成将应用UI绘制到frameBuffer\(帧缓冲区\)

## 6. 小结

下面是上述流程所对应的流程图， 简单地说， SurfaceFlinger 最主要的功能:**SurfaceFlinger 接受来自多个来源的数据缓冲区，对它们进行合成，然后发送到显示设备。**

![](https://www.androidperformance.com/images/15816781462135.jpg)

## 参考文章

1. [Systrace系列文章——高爷](https://www.androidperformance.com/2019/05/28/Android-Systrace-About/)
2. [Android开发高手课——UI部分](https://time.geekbang.org/column/article/80921)
3. [Android应用程序UI硬件加速渲染的Display List渲染过程分析](https://blog.csdn.net/luoshengyang/article/details/46281499)
4. [Android屏幕刷新机制—VSync、Choreographer 全面理解](https://juejin.cn/post/6863756420380196877)
5. [Android Performance Patterns: Understanding VSYNC](https://www.youtube.com/watch?v=1iaHxmfZGGc)

