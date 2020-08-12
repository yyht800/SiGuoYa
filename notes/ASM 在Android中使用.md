# Android 中ASM怎么使用
### 前情提要
最近准备做一个APM，调阅资料发现居然使用的是插桩的骚操作。一直想玩但是没有玩过，记录下操作的过程和坑点。
本文可能涉及的点，Android中的自定义gradle，ASM的使用等。

### 一、知识点预热

#### 1.1 什么是字节码插桩
[Android官网编译的过程](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process)中如图1-1，没有老版本的清晰如图1-2所示，先将xxx.java变成.class,然后再变成.dex文件，插桩就是对class文件的进行编辑。字节码插桩一般常用的有AspectJ，ASM，Redex（[Redex方案](https://github.com/facebook/redex)是Facebook的优化方案直接对dex进行操作，所见即所得）

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://developer.android.com/images/tools/studio/build-process_2x.png?hl=zh-cn">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">1-1 新版打包流程</div>
</center>


<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://user-gold-cdn.xitu.io/2017/3/2/35a4d886bc51ec6be29456eadd4b1fd2.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">1-2 老版打包流程</div>
</center>

#### 1.2 什么是Transform
Transform是Android gradle plugin 1.5开始引入的概念。
我们先从如何引入Transform依赖说起，首先我们需要编写一个自定义插件，然后在插件中注册一个自定义Transform。这其中我们需要先通过gradle引入Transform的依赖，这里有一个坑，Transform的库最开始是独立的，后来从2.0.0版本开始，被归入了Android编译系统依赖的gradle-api中。
它的原理如下图1-3所示。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://upload-images.jianshu.io/upload_images/13278007-e3181a3e9742296f?imageMogr2/auto-orient/strip|imageView2/2/w/900/format/webp">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">1-3 Transform的原理</div>
</center>


每个Transform其实都是一个gradle task，Android编译器中的TaskManager将每个Transform串连起来，第一个Transform接收来自javac编译的结果，以及已经拉取到在本地的第三方依赖（jar. aar），还有resource资源，注意，这里的resource并非android项目中的res资源，而是asset目录下的资源。这些编译的中间产物，在Transform组成的链条上流动，每个Transform节点可以对class进行处理再传递给下一个Transform。我们常见的混淆，Desugar等逻辑，它们的实现如今都是封装在一个个Transform中，而我们自定义的Transform，会插入到这个Transform链条的最前面。

但其实，上面这幅图，只是展示Transform的其中一种情况。而Transform其实可以有两种输入，一种是消费型的，当前Transform需要将消费型型输出给下一个Transform，另一种是引用型的，当前Transform可以读取这些输入，而不需要输出给下一个Transform，比如Instant Run就是通过这种方式，检查两次编译之间的diff的。


### 二、思路整理
在动手之前先整理一下思路,已知我们需要在编译期 修改目标class文件

关于混淆的
如果引用关系一直显示找不到或者报错的话，看下pack是否成功引入，没有的话手动添加一下
更新完代码记得build 在gradle 指令里面

### 参考文献
1. [在AndroidStudio中自定义Gradle插件](https://blog.csdn.net/huachao1001/article/details/51810328)
2. [一起玩转Android项目中的字节码](http://quinnchen.cn/2018/09/13/2018-09-13-asm-transform/)
3. [【Android】函数插桩（Gradle + ASM）](https://www.jianshu.com/p/16ed4d233fd1)
