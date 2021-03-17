## Android进阶解密第七、八章——理解Window相关

### 1. Window、WindowManager和WMS 概述

WMS和AMS一样是Android中非常在重要的两个机制，首先来看Window，他是一个抽象类，实现类是PhoneWindow，它对View进行管理。

WindowManager是继承自ViewManager，他是一个接口类。是用来管理Window的，他的实现类是WindowManagerImpl，如果我们想对Window（View）进行添加、更新、删除操作，就可以交给WindowManager，他会将具体的工作交给WMS来进行处理，这里的WindowManager和WMS会通过Binder进行通信。

### 2.Window类的属性

 Window 的属’性 有很多种，与应用开发最密切的有 3 种，它们分别是 Type (Window 的类型)、 Flag (Window 的标志)和 SoftlnputMode (软键盘相关模式)，下面分别介绍这 3 种 Window 的属性。

#### 2.1 Window的类型

window也是一样按照高度范围进行分类，他也有一个变量Z-Order，Z-Order越大，也就是高度越高，离用户也就越近。window一共可分为三类：

-  应用程序级别窗口

  应用程序窗口一般位于最底层，Z-Order在1-99，

- 子窗口

  子窗口一般是显示在应用窗口之上，Z-Order在1000-1999

- 系统级别窗口

  系统级窗口一般位于最顶层，不会被其他的window遮住，如Toast，Z-Order在2000-2999。**如果要弹出自定义系统级窗口需要动态申请权限**。

#### 2.2 Window的flags参数

flag标志控制window的东西比较多，很多资料的描述是“控制window的显示”，但我觉得不够准确。flag控制的范围包括了：各种情景下的显示逻辑（锁屏，游戏等）还有触控事件的处理逻辑。控制显示确实是他的很大部分功能，但是并不是全部。下面看一下一些常用的flag，就知道flag的功能了（以下常用参数介绍转自参考文献第一篇文章）：

```java
// 当 Window 可见时允许锁屏
public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON = 0x00000001;

// Window 后面的内容都变暗
public static final int FLAG_DIM_BEHIND = 0x00000002;

// Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的
// Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL
public static final int FLAG_NOT_FOCUSABLE = 0x00000008;

// 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件
// Window 之外的 view 也是可以响应 touch 事件。
public static final int FLAG_NOT_TOUCH_MODAL  = 0x00000020;

// 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。
public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;

// 只要 Window 可见时屏幕就会一直亮着
public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;

// 允许 Window 占满整个屏幕
public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;

// 允许 Window 超过屏幕之外
public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;

// 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示
public static final int FLAG_FULLSCREEN      = 0x00000400;

// 表示比FLAG_FULLSCREEN低一级，会显示状态栏
public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;

// 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
public static final int FLAG_IGNORE_CHEEK_PRESSES    = 0x00008000;

// 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件。
public static final int FLAG_WATCH_OUTSIDE_TOUCH = 0x00040000;

@Deprecated
// 窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替
public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;

// 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，
// 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。
public static final int FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS = 0x80000000;

// 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸
public static final int FLAG_SHOW_WALLPAPER = 0x00100000;

```



#### 2.3 软键盘相关模式

这一部分就是当软件盘弹起来的时候，window的处理逻辑，这在日常中也经常遇到，如：我们在微信聊天的时候，点击输入框，当软键盘弹起来的时候输入框也会被顶上去。如果你不想被顶上去，也可以设置为被软键盘覆盖。下面介绍一下常见的属性（以下常见属性介绍选自参考文献第一篇文章）：

```java
// 没有指定状态，系统会选择一个合适的状态或者依赖于主题的配置
public static final int SOFT_INPUT_STATE_UNCHANGED = 1;

// 当用户进入该窗口时，隐藏软键盘
public static final int SOFT_INPUT_STATE_HIDDEN = 2;

// 当窗口获取焦点时，隐藏软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_HIDDEN = 3;

// 当用户进入窗口时，显示软键盘
public static final int SOFT_INPUT_STATE_VISIBLE = 4;

// 当窗口获取焦点时，显示软键盘
public static final int SOFT_INPUT_STATE_ALWAYS_VISIBLE = 5;

// window会调整大小以适应软键盘窗口
public static final int SOFT_INPUT_MASK_ADJUST = 0xf0;

// 没有指定状态,系统会选择一个合适的状态或依赖于主题的设置
public static final int SOFT_INPUT_ADJUST_UNSPECIFIED = 0x00;

// 当软键盘弹出时，窗口会调整大小,例如点击一个EditView，整个layout都将平移可见且处于软件盘的上方
// 同样的该模式不能与SOFT_INPUT_ADJUST_PAN结合使用；
// 如果窗口的布局参数标志包含FLAG_FULLSCREEN，则将忽略这个值，窗口不会调整大小，但会保持全屏。
public static final int SOFT_INPUT_ADJUST_RESIZE = 0x10;

// 当软键盘弹出时，窗口不需要调整大小, 要确保输入焦点是可见的,
// 例如有两个EditView的输入框，一个为Ev1，一个为Ev2，当你点击Ev1想要输入数据时，当前的Ev1的输入框会移到软键盘上方
// 该模式不能与SOFT_INPUT_ADJUST_RESIZE结合使用
public static final int SOFT_INPUT_ADJUST_PAN = 0x20;

// 将不会调整大小，直接覆盖在window上
public static final int SOFT_INPUT_ADJUST_NOTHING = 0x30;

```



###  3. Window的添加过程

添加过程核心就是WindowManager（即实现类WindowManagerImpl）的addView的方法，代码如下：

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}

//WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // 首先判断参数是否合法
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }
    
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    // 如果不是子窗口，会对其做参数的调整
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
    
	synchronized (mLock) {
        ...
        // 这里新建了一个viewRootImpl，并设置参数
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);

        // 添加到windowManagerGlobal的三个重要list中，后面会讲到
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // 最后通过viewRootImpl来添加window
        try {
            root.setView(view, wparams, panelParentView);
        } 
        ...
    }  
}

```

WindowManagerGlobal，是一个全局单例，是WindowManager接口的具体逻辑实现。这里运用的是桥接模式，可以看下WindowManagerGlobal中addView的方法，

- 首先对参数的合法性进行检查

- 然后判断该窗口是不是子窗口，如果是的话需要对窗口进行调整，这个好理解，子窗口要跟随父窗口的特性。
- 接着新建viewRootImpl对象，并把view、viewRootImpl、params三个对象添加到三个list中进行保存

- 最后通过viewRootImpl来进行添加

可以看到添加的window的逻辑就交给ViewRootImpl了。viewRootImpl是window和view之间的桥梁，viewRootImpl可以处理两边的对象，然后联结起来。下面看一下viewRootImpl怎么处理：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        ...
        try {
            mOrigWindowType = mWindowAttributes.type;
            mAttachInfo.mRecomputeGlobalAttributes = true;
            collectViewAttributes();
            // 这里调用了windowSession的方法，调用wms的方法，把添加window的逻辑交给wms
            res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                    mTempInsets);
            setFrame(mTmpFrame);
        } 
        ...
    }
}
```

这里主要就是调用了mWindowSession.addToDisplay，mWindowSession是IWindowSession类型的，他是一个Binder对象，最后调用的是WMS，后面的逻辑就交给WMS去处理了，WMS就会创建window，然后结合参数计算window的高度等等，最后使用viewRootImpl进行绘制。

### 4.window的机制

WMS也是在SystemServer的run方法中创建的，感觉书里面也没有讲得特别清楚，大致框架应该是

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/768cd62019764d129e24d432792e3638~tplv-k3u1fbpfcp-zoom-1.image)

我们知道WMS是window的最终管理者，在WMS中为每一个应用持有一个session，关于session前面我们讲过，每个应用都是全局单例，负责和WMS通信的binder对象。WMS为每个window都建立了一个windowStatus对象，同一个应用的Window使用同一个session进行通信，每一个windowStatus对应一个viewRootImpl，WMS通过viewRootImpl来控制view。这也就是window机制的管理结构。

当我们需要添加window的时候，最终的逻辑实现是WindowManagerGlobal，他的内部使用自己的session创建一个viewRootImpl，然后向WMS申请添加window，结构图大概如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5dcbb1f264646539f78652b08ce0fc6~tplv-k3u1fbpfcp-zoom-1.image)

windowManagerGlobal使用自己的IWindowSession创建viewRootImpl，这个IWindowSession是全局单例。viewRootImpl和WMS申请创建window，然后WMS允许之后，再通知viewRootImpl绘制view，同时WMS通过windowStatus存储了viewRootImpl的相关信息，这样如果WMS需要修改view，直接通过viewRootImpl就可以修改view了。

### 参考文章

1. [Android全面解析之Window机制](https://juejin.cn/post/6888688477714841608#heading-3)