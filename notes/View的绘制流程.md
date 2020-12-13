## View的绘制流程

### 1. 概述

View作为Activity的一个元素，它的绘制流程不能局限于内部的onMeasure() -> onLayout() -> onDraw()，而是要整体的来看流程的传递。

简单看下结构如下图所示。

<img src="https://upload-images.jianshu.io/upload_images/2397836-f1f6a200704884a2.png" style="zoom:60%;" />

-  **Activity**

  Activity 并不负责视图的控制，它只是控制生命周期和处理事件。真正控制视图的是Window。一个Activity对应了一个Window，而Window才是真正代表一个窗口。它更像一个控制器，统筹视图的添加显示等操作。关于Activity的创建就不展开了，可以从其他文章中获取相关的情报。

- **Window**

  Window是一个抽象类，表示Android中的窗口。在Android系统中，窗口是独占一个Surface实例的显示区域，每个窗口的Surface 由WindowManagerService分配。我们可以把Surface看作一块画布，应用可以通过Canvas或OpenGL在其上面作画。画好之后，通过SurfaceFlinger将多块Surface按照特定的顺序（即Z-order）进行混合，而后输出到FrameBuffer中，这样用户界面就得以显示。

  而PhoneWindow是Framework为我们提供的Android窗口的具体实现。我们平时调用setContentView()方法设置Activity的用户界面时，实际上就完成了对所关联的PhoneWindow的ViewTree的设置。

- **DecorView**

   继承自 FrameLayout，是Activity/Window中最顶级的View，一般情况下，内部是一个竖直方向的LinearLayout。与setContentView紧密相关。

###2.  从setContentView开始

我们需要明确：这个方法只是完成了Activity的ContentView的创建，而并没有执行View的绘制流程。调用的setContentView()方法是Activity类的，源码如下：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

getWindow()方法会返回Activity所关联的PhoneWindow，也就是说，实际上调用到了PhoneWindow的setContentView()方法，源码如下：

```java
@Override
public void setContentView(int layoutResID) {
  if (mContentParent == null) {
    // mContentParent即为上面提到的ContentView的父容器，若为空则调用installDecor()生成
    installDecor();
  } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
    // 具有FEATURE_CONTENT_TRANSITIONS特性表示开启了Transition
    // mContentParent不为null，则移除decorView的所有子View
    mContentParent.removeAllViews();
  }
  if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
    // 开启了Transition，做相应的处理，我们不讨论这种情况
    // 感兴趣的同学可以参考源码
    . . .
  } else {
    // 一般情况会来到这里，调用mLayoutInflater.inflate()方法来填充布局
    // 填充布局也就是把我们设置的ContentView加入到mContentParent中
    mLayoutInflater.inflate(layoutResID, mContentParent);
  }
  . . .
  // cb即为该Window所关联的Activity
  final Callback cb = getCallback();
  if (cb != null && !isDestroyed()) {
    // 调用onContentChanged()回调方法通知Activity窗口内容发生了改变
    cb.onContentChanged();
  }

  . . .
}
```

### 3. LayoutInflater.inflate()

从上面的代码中可以看到，PhoneWindow的setContentView()方法中调用了LayoutInflater的inflate()方法来填充布局，这个方法的源码如下：

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        ...
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```

在PhoneWindow的setContentView()方法中传入了decorView作为LayoutInflater.inflate()的root参数，我们可以看到，通过层层调用，最终调用的是inflate(XmlPullParser, ViewGroup, boolean)方法来填充布局。这个方法的源码如下：

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
  synchronized (mConstructorArgs) {
    . . .
    final Context inflaterContext = mContext;
    final AttributeSet attrs = Xml.asAttributeSet(parser);
    Context lastContext = (Context) mConstructorArgs[0];
    mConstructorArgs[0] = inflaterContext;

    View result = root;

    try {
      // Look for the root node.
      int type;
      // 一直读取xml文件，直到遇到开始标记
      while ((type = parser.next()) != XmlPullParser.START_TAG &&
          type != XmlPullParser.END_DOCUMENT) {
        // Empty
       }
      // 最先遇到的不是开始标记，报错
      if (type != XmlPullParser.START_TAG) {
        throw new InflateException(parser.getPositionDescription()
+ ": No start tag found!");
      }

      final String name = parser.getName();
      . . .
      // 单独处理<merge>标签，不熟悉的同学请参考官方文档的说明
      if (TAG_MERGE.equals(name)) {
        // 若包含<merge>标签，父容器（即root参数）不可为空且attachRoot须为true，否则报错
        if (root == null || !attachToRoot) {
          throw new InflateException("<merge /> can be used only with a valid "
+ "ViewGroup root and attachToRoot=true");
        }
        
        // 递归地填充布局
        rInflate(parser, root, inflaterContext, attrs, false);
     } else {
        // temp为xml布局文件的根View
        final View temp = createViewFromTag(root, name, inflaterContext, attrs); 
        ViewGroup.LayoutParams params = null;
        if (root != null) {
          . . .
          // 获取父容器的布局参数（LayoutParams）
          params = root.generateLayoutParams(attrs);
          if (!attachToRoot) {
            // 若attachToRoot参数为false，则我们只会将父容器的布局参数设置给根View
            temp.setLayoutParams(params);
          }

        }

        // 递归加载根View的所有子View
        rInflateChildren(parser, temp, attrs, true);
        . . .

        if (root != null && attachToRoot) {
          // 若父容器不为空且attachToRoot为true，则将父容器作为根View的父View包裹上来
          root.addView(temp, params);
        }
      
        // 若root为空或是attachToRoot为false，则以根View作为返回值
        if (root == null || !attachToRoot) {
           result = temp;
        }
      }

    } catch (XmlPullParserException e) {
      . . . 
    } catch (Exception e) {
      . . . 
    } finally {

      . . .
    }
    return result;
  }
}
```

在上面的源码中，首先对于布局文件中的<merge>标签进行单独处理，调用rInflate()方法来递归填充布局。这个方法的源码如下：

```java
void rInflate(XmlPullParser parser, View parent, Context context,
    AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
    // 获取当前标记的深度，根标记的深度为0,可以理解成viewTree的层级
    final int depth = parser.getDepth();
    int type;
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
        parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
      // 不是开始标记则继续下一次迭代
      if (type != XmlPullParser.START_TAG) {
        continue;
      }
      final String name = parser.getName();
      // 对一些特殊标记做单独处理
      if (TAG_REQUEST_FOCUS.equals(name)) {
        parseRequestFocus(parser, parent);
      } else if (TAG_TAG.equals(name)) {
        parseViewTag(parser, parent, attrs);
      } else if (TAG_INCLUDE.equals(name)) {
        if (parser.getDepth() == 0) {
          throw new InflateException("<include /> cannot be the root element");
        }
        // 对<include>做处理
        parseInclude(parser, context, parent, attrs);
      } else if (TAG_MERGE.equals(name)) {
        throw new InflateException("<merge /> must be the root element");
      } else {
        // 对一般标记的处理
        final View view = createViewFromTag(parent, name, context, attrs);
        final ViewGroup viewGroup = (ViewGroup) parent;
        final ViewGroup.LayoutParams params=viewGroup.generateLayoutParams(attrs);
        // 递归地加载子View
        rInflateChildren(parser, view, attrs, true);
        viewGroup.addView(view, params);
      }
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```

我们可以看到，上面的inflate()和rInflate()方法中都调用了rInflateChildren()方法，这个方法的源码如下：

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

上面的流程可以看到，LayoutInflate.inflate 最终是调用 createViewFromTag 从 xml 生成 View 的，其实这里才是关键。

```java
 View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
   			/** 如果是 view 标签的话，就取其 class 属性作为 name
        * 比如
        * <view class="LinearLayout"/>
        * 最终生成的会是一个 LinearLayout
        * 是不是又学会了一种 view 的写法 ^_^
        */
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }
				...
        //处理blink标签
        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
          // 通过 mFactory2、mFactory、mPrivateFactory 创建 View
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }
						// 没有设置 Factory，走默认的创建 View 的流程
            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
         
        } 
    }
```

这里我们需要了解一下上文中提到的mFactory、mFactory2、mPrivateFactory 都是什么？

1. mFactory、mFactory2

   分别对应的是 Factory 和 Factory2 的 onCreateView 方法，Factory.onCreateView 没有传入 parent 参数，Factory2.onCreateView 传入了 parent 参数。如果已经有 mFactory 的值，则生成一个 FactoryMerger，这个也是继承了 Factory2，用来控制一下调用顺序。

2. mPrivateFactory

   看名称就知道是系统的隐藏方法。调用时机是在 Activity.attach 中，Activity 其实是实现了 Factory2 的 onCreateView 方法，其中对 fragment 做了处理，如果是 fragment 标签，就调用 fragment 的 onCreateView，这里就不详细往下面看了，如果是非 fragment 的标签，就返回 null，走默认的创建 View 的方法。

创建View之后则LayoutInflate.inflate 的整个流程结束，到这里，setContentView()的整体执行流程我们就分析完了，至此我们已经完成了Activity的ContentView的创建与设置工作。

### 3. View开始绘制

####  3.1 添加DecorView，ViewRootImpl并建立关联关系

View 的绘制流程，那整个流程是什么时候触发的呢？答案是在 ActivityThread.handleResumeActivity 里触发的。

```java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    // 调用Activity的onResume方法
    ActivityClientRecord r = performResumeActivity(token, clearHide);
    if (r != null) {
        final Activity a = r.activity;
        ...
        if (r.window == null &&& !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            // 得到DecorView
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            // 得到了WindowManager，WindowManager是一个接口
            // 并且继承了接口ViewManager
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                // WindowManager的实现类是WindowManagerImpl，
                // 所以实际调用的是WindowManagerImpl的addView方法
                wm.addView(decor, l);
            }
        }
    }
}

public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    ...

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }
    ...
}
```

这里补充说明一下，ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联，相关源码如下所示：

```java
// WindowManagerGlobal的addView方法
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    View pannelParentView = null;
    synchronized (mLock) {
        ...
        // 创建ViewRootImpl实例
        root = new ViewRootImpl(view..getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
    try {
        // 把DecorView加载到Window中
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        synchronized (mLock) {
            final int index = findViewLocked(view, false);
            if (index >= 0) {
                removeViewLocked(index, true);
            }
        }
        throw e;
    }
}
```

最终通过 WindowManagerImpl.addView -> WindowManagerGlobal.addView -> ViewRootImpl.setView -> ViewRootImpl.requestLayout 就触发了第一次 View 的绘制。

#### 3.2 MeasureSpec的理解

很多博客包括官方文档对他的说明基本都是“一个MeasureSpec封装了从父容器传递给子容器的布局要求”,这个MeasureSpec 封装的是父容器传递给子容器的布局要求，而不是父容器对子容器的布局要求，“传递” 两个字很重要，更精确的说法应该这个MeasureSpec是由父View的MeasureSpec和子View的LayoutParams通过简单的计算得出一个针对子View的测量要求，这个测量要求就是MeasureSpec。

MeasureSpec是一个大小跟模式的组合值,MeasureSpec中的值是一个整型（32位）将size和mode打包成一个Int型，其中高两位是mode，后面30位存的是size，是为了减少对象的分配开支。MeasureSpec 类似于下图，只不过这边用的是十进制的数，而MeasureSpec 是二进制存储的。

![](https://upload-images.jianshu.io/upload_images/966283-c330852c971b02a8.png)

MeasureSpec一共有三种模式

- **UPSPECIFIED** : 父容器对于子容器没有任何限制,子容器想要多大就多大
- **EXACTLY**: 父容器已经为子容器设置了尺寸,子容器应当服从这些边界,不论子容器想要多大的空间。
-  **AT_MOST**：子容器可以是声明大小内的任意大小



父View的measure的过程会先测量子View，等子View测量结果出来后，再来测量自己，上面的measureChildWithMargins就是用来测量某个子View的，我们来分析是怎样测量的，具体看注释：

```java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) { 

// 子View的LayoutParams，你在xml的layout_width和layout_height,
// layout_xxx的值最后都会封装到这个个LayoutParams。
final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();   

//根据父View的测量规格和父View自己的Padding，
//还有子View的Margin和已经用掉的空间大小（widthUsed），就能算出子View的MeasureSpec,具体计算过程看getChildMeasureSpec方法。
final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,            
mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);    

final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,           
mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin  + heightUsed, lp.height);  

//通过父View的MeasureSpec和子View的自己LayoutParams的计算，算出子View的MeasureSpec，然后父容器传递给子容器的
// 然后让子View用这个MeasureSpec（一个测量要求，比如不能超过多大）去测量自己，如果子View是ViewGroup 那还会递归往下测量。
child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

}

// spec参数   表示父View的MeasureSpec 
// padding参数    父View的Padding+子View的Margin，父View的大小减去这些边距，才能精确算出
//               子View的MeasureSpec的size
// childDimension参数  表示该子View内部LayoutParams属性的值（lp.width或者lp.height）
//                    可以是wrap_content、match_parent、一个精确指(an exactly size),  
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {  
    int specMode = MeasureSpec.getMode(spec);  //获得父View的mode  
    int specSize = MeasureSpec.getSize(spec);  //获得父View的大小  

   //父View的大小-自己的Padding+子View的Margin，得到值才是子View的大小。
    int size = Math.max(0, specSize - padding);   
  
    int resultSize = 0;    //初始化值，最后通过这个两个值生成子View的MeasureSpec
    int resultMode = 0;    //初始化值，最后通过这个两个值生成子View的MeasureSpec
  
    switch (specMode) {  
    // Parent has imposed an exact size on us  
    //1、父View是EXACTLY的 ！  
    case MeasureSpec.EXACTLY:   
        //1.1、子View的width或height是个精确值 (an exactly size)  
        if (childDimension >= 0) {            
            resultSize = childDimension;         //size为精确值  
            resultMode = MeasureSpec.EXACTLY;    //mode为 EXACTLY 。  
        }   
        //1.2、子View的width或height为 MATCH_PARENT/FILL_PARENT   
        else if (childDimension == LayoutParams.MATCH_PARENT) {  
            // Child wants to be our size. So be it.  
            resultSize = size;                   //size为父视图大小  
            resultMode = MeasureSpec.EXACTLY;    //mode为 EXACTLY 。  
        }   
        //1.3、子View的width或height为 WRAP_CONTENT  
        else if (childDimension == LayoutParams.WRAP_CONTENT) {  
            // Child wants to determine its own size. It can't be  
            // bigger than us.  
            resultSize = size;                   //size为父视图大小  
            resultMode = MeasureSpec.AT_MOST;    //mode为AT_MOST 。  
        }  
        break;  
  
    // Parent has imposed a maximum size on us  
    //2、父View是AT_MOST的 ！      
    case MeasureSpec.AT_MOST:  
        //2.1、子View的width或height是个精确值 (an exactly size)  
        if (childDimension >= 0) {  
            // Child wants a specific size... so be it  
            resultSize = childDimension;        //size为精确值  
            resultMode = MeasureSpec.EXACTLY;   //mode为 EXACTLY 。  
        }  
        //2.2、子View的width或height为 MATCH_PARENT/FILL_PARENT  
        else if (childDimension == LayoutParams.MATCH_PARENT) {  
            // Child wants to be our size, but our size is not fixed.  
            // Constrain child to not be bigger than us.  
            resultSize = size;                  //size为父视图大小  
            resultMode = MeasureSpec.AT_MOST;   //mode为AT_MOST  
        }  
        //2.3、子View的width或height为 WRAP_CONTENT  
        else if (childDimension == LayoutParams.WRAP_CONTENT) {  
            // Child wants to determine its own size. It can't be  
            // bigger than us.  
            resultSize = size;                  //size为父视图大小  
            resultMode = MeasureSpec.AT_MOST;   //mode为AT_MOST  
        }  
        break;  
  
    // Parent asked to see how big we want to be  
    //3、父View是UNSPECIFIED的 ！  
    case MeasureSpec.UNSPECIFIED:  
        //3.1、子View的width或height是个精确值 (an exactly size)  
        if (childDimension >= 0) {  
            // Child wants a specific size... let him have it  
            resultSize = childDimension;        //size为精确值  
            resultMode = MeasureSpec.EXACTLY;   //mode为 EXACTLY  
        }  
        //3.2、子View的width或height为 MATCH_PARENT/FILL_PARENT  
        else if (childDimension == LayoutParams.MATCH_PARENT) {  
            // Child wants to be our size... find out how big it should  
            // be  
            resultSize = 0;                        //size为0！ ,其值未定  
            resultMode = MeasureSpec.UNSPECIFIED;  //mode为 UNSPECIFIED  
        }   
        //3.3、子View的width或height为 WRAP_CONTENT  
        else if (childDimension == LayoutParams.WRAP_CONTENT) {  
            // Child wants to determine its own size.... find out how  
            // big it should be  
            resultSize = 0;                        //size为0! ，其值未定  
            resultMode = MeasureSpec.UNSPECIFIED;  //mode为 UNSPECIFIED  
        }  
        break;  
    }  
    //根据上面逻辑条件获取的mode和size构建MeasureSpec对象。  
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);  
}    
```

参照上面的注释可以这样去理解三种模式：

- 如果父View的MeasureSpec 是EXACTLY，说明父View的大小是确切的，确切的意思很好理解，如果一个View的MeasureSpec 是EXACTLY，那么它的size 是多大，最后展示到屏幕就一定是那么大，简单理解成我们在布局中写的固定值。
- 如果父View的MeasureSpec 是AT_MOST，说明父View的大小是不确定，但是最大的size限定的就是是MeasureSpec 的值，不能超过这个值，子view最大的大小在限定的最大MeasureSpec之内不受限制。
- 如果父View的MeasureSpec 是UNSPECIFIED(未指定),表示没有任何束缚和约束，不像AT_MOST表示最大只能多大，不也像EXACTLY表示父View确定的大小，子View可以得到任意想要的大小，不受约束

#### 3.2 view 的measure

在3.1中我们知道了，view 的第一次绘制是在ViewRootImpl.requestLayout触发的，但是实际调用 scheduleTraversals -> doTraversal -> performTraversals 才开启绘制流程，ViewRootImpl.requestLayout是触发节点，开始绘制的节点是performTraversals。

```java
private void performTraversals() { 
...... 
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width); 
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height); 
...... 
mView.measure(childWidthMeasureSpec, childHeightMeasureSpec); 
......
mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
...... 
mView.draw(canvas); 
......
}
```

在 performTraversals 里，就是熟悉的 performMeasure -> performLayout -> performDraw 三个流程了。

performMeasure 实际对应的就是view的onMeasure方法，而ViewGroup 类并没有实现onMeasure，我们知道测量过程其实都是在onMeasure方法里面做的，我们来看下FrameLayout 的onMeasure 方法,具体分析看注释哦。

```java
//FrameLayout 的测量
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
....
int maxHeight = 0;
int maxWidth = 0;
int childState = 0;
for (int i = 0; i < count; i++) {    
   final View child = getChildAt(i);    
   if (mMeasureAllChildren || child.getVisibility() != GONE) {   
    // 遍历自己的子View，只要不是GONE的都会参与测量，measureChildWithMargins方法在最上面
    // 的源码已经讲过了，如果忘了回头去看看，基本思想就是父View把自己的MeasureSpec 
    // 传给子View结合子View自己的LayoutParams 算出子View 的MeasureSpec，然后继续往下传，
    // 传递叶子节点，叶子节点没有子View，根据传下来的这个MeasureSpec测量自己就好了。
     measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);       
     final LayoutParams lp = (LayoutParams) child.getLayoutParams(); 
     maxWidth = Math.max(maxWidth, child.getMeasuredWidth() +  lp.leftMargin + lp.rightMargin);        
     maxHeight = Math.max(maxHeight, child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);  
     ....
     ....
   }
}
.....
.....
//所有的孩子测量之后，经过一系类的计算之后通过setMeasuredDimension设置自己的宽高，
//对于FrameLayout 可能用最大的字View的大小，对于LinearLayout，可能是高度的累加，
//具体测量的原理去看看源码。总的来说，父View是等所有的子View测量结束之后，再来测量自己。
setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),        
resolveSizeAndState(maxHeight, heightMeasureSpec, childState << MEASURED_HEIGHT_STATE_SHIFT));
....
}  
```

#### 3.3 View的layout

在View的measure的过程结束后就开始了layout的工作，这里layout的主要作用 ：根据子视图的大小以及布局参数将View树放到合适的位置上。

先看下ViewGroup的layout方法：

```java
public final void layout(int l, int t, int r, int b) {    
   if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {        
    if (mTransition != null) {            
       mTransition.layoutChange(this);        
    }       
    super.layout(l, t, r, b);    
    } else {        
    // record the fact that we noop'd it; request layout when transition finishes        
      mLayoutCalledWhileSuppressed = true;    
   }
}
```

代码可以看个大概，LayoutTransition是用于处理ViewGroup增加和删除子视图的动画效果，也就是说如果当前ViewGroup未添加LayoutTransition动画，或者LayoutTransition动画此刻并未运行，那么调用super.layout(l, t, r, b)，继而调用到ViewGroup中的onLayout，否则将mLayoutSuppressed设置为true，等待动画完成时再调用requestLayout()。
这个函数是final 不能重写，所以ViewGroup的子类都会调用这个函数，layout 的具体实现是在super.layout(l, t, r, b)里面做的，那么我接下来看一下View类的layout函数：

```java
 public final void layout(int l, int t, int r, int b) {
       .....
      //设置View位于父视图的坐标轴
       boolean changed = setFrame(l, t, r, b); 
       //判断View的位置是否发生过变化，看有必要进行重新layout吗
       if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {
           if (ViewDebug.TRACE_HIERARCHY) {
               ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_LAYOUT);
           }
           //调用onLayout(changed, l, t, r, b); 函数
           onLayout(changed, l, t, r, b);
           mPrivateFlags &= ~LAYOUT_REQUIRED;
       }
       mPrivateFlags &= ~FORCE_LAYOUT;
       .....
   }
```

1、setFrame(l, t, r, b) 可以理解为给mLeft 、mTop、mRight、mBottom赋值，然后基本就能确定View自己在父视图的位置了，这几个值构成的矩形区域就是该View显示的位置，这里的具体位置都是相对与父视图的位置。

2、回调onLayout，对于View来说，onLayout只是一个空实现，一般情况下我们也不需要重载该函数,：

```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {  

    }  
```

对于ViewGroup 来说，唯一的差别就是ViewGroup中多了关键字abstract的修饰，要求其子类必须重载onLayout函数。

```java
@Override  
protected abstract void onLayout(boolean changed,  
        int l, int t, int r, int b); 
```

而重载onLayout的目的就是安排其children在父视图的具体位置，那么如何安排子View的具体位置呢？

```java
 int childCount = getChildCount() ; 
  for(int i=0 ;i<childCount ;i++){
       View child = getChildAt(i) ;
       //整个layout()过程就是个递归过程
       child.layout(l, t, r, b) ;
    }
```

代码很简单，就是遍历自己的孩子，然后调用  child.layout(l, t, r, b) ，给子view 通过setFrame(l, t, r, b) 确定位置，而重点是(l, t, r, b) 怎么计算出来的呢。还记得我们之前测量过程，测量出来的MeasuredWidth和MeasuredHeight吗？还记得你在xml 设置的Gravity吗？还有RelativeLayout 的其他参数吗，没错，就是这些参数和MeasuredHeight、MeasuredWidth 一起来确定子View在父视图的具体位置的。具体的计算过程大家可以看下最简单FrameLayout 的onLayout 函数的源码，每个不同的ViewGroup  的实现都不一样，这边不做具体分析了吧。

#### 3.4 View 的draw

 View的draw 方法一般不去重写，官网文档也建议不要去重写draw 方法，所以下一步执行就是View.java的draw 方法，我们来看下源码：

```java
// 绘制基本上可以分为六个步骤
public void draw(Canvas canvas) {
    ...
    // 步骤一：绘制View的背景
    drawBackground(canvas);

    ...
    // 步骤二：如果需要的话，保持canvas的图层，为fading做准备
    saveCount = canvas.getSaveCount();
    ...
    canvas.saveLayer(left, top, right, top + length, null, flags);

    ...
    // 步骤三：绘制View的内容
    onDraw(canvas);

    ...
    // 步骤四：绘制View的子View
    dispatchDraw(canvas);

    ...
    // 步骤五：如果需要的话，绘制View的fading边缘并恢复图层
    canvas.drawRect(left, top, right, top + length, p);
    ...
    canvas.restoreToCount(saveCount);

    ...
    // 步骤六：绘制View的装饰(例如滚动条等等)
    onDrawForeground(canvas)
}
```

看注释就可以了





### 参考文献

1. [深入理解Android之View的绘制流程](https://www.jianshu.com/p/060b5f68da79)
2. [Android View的绘制流程 ---jsonchao](https://jsonchao.github.io/2018/10/28/Android%20View%E7%9A%84%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B/)
3. [面试官带你学安卓 - 从 View 的绘制流程说起](https://mp.weixin.qq.com/s?__biz=MzI3OTcyNjQ5MQ==&mid=2247483839&idx=1&sn=de79bab8aff5210b076527190dd72d6a&chksm=eb42114bdc35985d64e88c7acad0e4f690c23e40d9320eab0af5d3048c11c1dc2dfd22f8c0cf#rd)
4. [Android View的绘制流程-kelin](https://www.jianshu.com/p/5a71014e7b1b)
5. Android 系统源码