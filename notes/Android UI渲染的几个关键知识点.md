## Android UI渲染的几个关键知识点

### 一、前言
Android中UI部分一直是自己知识的薄弱点，不成体系，逻辑上也不连贯，知其然而不知其所
以然，特此记录一下以便后续可以查漏补缺。

### 二、屏幕适配
#### 2.1 设备兼容性
其实这里还蕴含一个隐藏的属性--设备兼容性，由于Android是一个开源项目，你无法保证他会出现在哪里，不过我们能狗采购的到的设备，厂商已经对这个问题进行了处理，所以开发者并不需要去关心这个。

#### 2.1.2 支持不同屏幕尺寸和不同密度
这个其实是我们平时最常讨论的一个话题，屏幕尺寸和密度的适配，也就是我们常说的适配方案的选择。这个话题之前介绍一点基础知识。

- **px、dp、dpi、ppi、density的基本概念**
  
  ![](https://static001.geekbang.org/resource/image/e3/ce/e3094e900dccacb9d9e72063ca3084ce.png)
    px / density = dp，DPI / 160 = density，所以最终的公式是 px / (DPI / 160) = dp。 虽然官方推荐使用的是dp，他也能解决大部分的屏幕适配问题，但是同时也存在两个比较大的问题。
    1. **不一致性**。因为 dpi 与实际 ppi 的差异性，导致在相同分辨率的手机上，控件的实际大小会有所不同。（这是主要问题）
    2. **效率**。设计师的设计稿都是以 px 为单位的，开发人员为了 UI 适配，需要手动通过百分比估算出 dp 值。（当然现在一些工具已经帮我们做了这一步了）

- **主流的适配方案**
  1. [smallestWidth 限定符适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826381&idx=1&sn=5b71b7f1654b04a55fca25b0e90a4433&chksm=80b7b213b7c03b0598f6014bfa2f7de12e1f32ca9f7b7fc49a2cf0f96440e4a7897d45c788fb&scene=21#wechat_redirect)当系统识别到手机的smallestWidth值时，就会自动去寻找和目标数据最近的资源文件的尺寸。
  2. [今日头条适配方案](https://mp.weixin.qq.com/s/oSBUA7QKMWZURm1AHMyubA)通过反射修正系统的 density 值;

- **mipmap和drawable职责的区分**
  
  谷歌建议将启动图标放置在mipmap下，而将其他资源图片放置在drawable对应目录下，
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://github.com/yyht800/SiGuoYa/blob/master/res/drawable/drawable%E4%BB%8B%E7%BB%8D.png?raw=true">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">密度对应的表格</div>
</center>
 
### 三、CPU和GPU
除了屏幕，UI 渲染还依赖两个核心的硬件：CPU 与 GPU。UI 组件在绘制到屏幕之前，都需要经过 Rasterization（栅格化）操作，而栅格化操作又是一个非常耗时的操作。GPU（Graphic Processing Unit ）也就是图形处理器，它主要用于处理图形运算，可以帮助我们加快栅格化操作。
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://blog.yorek.xyz/assets/images/android/master/ui_1_2.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">软硬件绘制流程</div>
</center>

由上图可知，软件绘制使用的是 Skia 库，它是一款能在低端设备如手机上呈现高质量的 2D 跨平台图形框架，类似 Chrome、Flutter 内部使用的都是 Skia 库。

### 四、OpenGL 与 Vulkan

对于硬件绘制，我们通过调用 OpenGL ES 接口利用 GPU 完成绘制。OpenGL是一个跨平台的图形 API，它为 2D/3D 图形处理硬件指定了标准软件接口。而 OpenGL ES 是 OpenGL 的子集，专为嵌入式设备设计。

在官方[硬件加速的文档](https://developer.android.com/guide/topics/graphics/hardware-accel)中，可以看到很多 API 都有相应的 Android API level 限制。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://blog.yorek.xyz/assets/images/android/master/ui_1_3.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;"></div>
</center>

这是为什么呢？其实这主要是受OpenGL ES版本与系统支持的限制，直到 Android P，有 3 个 API 是仍然没有支持。对于不支持的 API，我们需要使用软件绘制模式，渲染的性能将会大大降低。

Android 7.0 把 OpenGL ES 升级到最新的 3.2 版本同时，还添加了对Vulkan的支持。Vulkan 是用于高性能 3D 图形的低开销、跨平台 API。相比 OpenGL ES，Vulkan 在改善功耗、多核优化提升绘图调用上有着非常明显的优势。

在国内，“王者荣耀”是比较早适配 Vulkan 的游戏，虽然目前兼容性还有一些问题，但是 Vulkan 版本的王者荣耀在流畅性和帧数稳定性都有大幅度提升，即使是战况最激烈的团战阶段，也能够稳定保持在 55～60 帧。


### 五、Android 渲染的演进

Android 系统为了弥补跟 iOS 的差距，在每个版本都做了大量的优化。在了解 Android 的渲染之前，需要先了解一下 Android 图形系统的整体架构，以及它包含的主要组件。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://blog.yorek.xyz/assets/images/android/master/ui_1_3.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">来自官网--关键组件如何协同工作</div>
</center>

如果把应用程序图形渲染过程当作一次绘画过程，那么绘画过程中 Android 的各个图形组件的作用是：

- 画笔：Skia 或者 OpenGL。我们可以用 Skia 画笔绘制 2D 图形，也可以用 OpenGL 来绘制 2D/3D 图形。正如前面所说，前者使用 CPU 绘制，后者使用 GPU 绘制。
- 画纸：Surface。所有的元素都在 Surface 这张画纸上进行绘制和渲染。在 Android 中，Window 是 View 的容器，每个窗口都会关联一个 Surface。而 WindowManager 则负责管理这些窗口，并且把它们的数据传递给 SurfaceFlinger。
- 画板：Graphic Buffer。Graphic Buffer 缓冲用于应用程序图形的绘制，在 Android 4.1 之前使用的是双缓冲机制；在 Android 4.1 之后，使用的是三缓冲机制。
- 显示：SurfaceFlinger。它将 WindowManager 提供的所有 Surface，通过硬件合成器 Hardware Composer 合成并输出到显示屏。

#### 5.1 Android 4.0：开启硬件加速
所以从 Androd 3.0 开始，Android 开始支持硬件加速，到 Android 4.0 时，默认开启硬件加速。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://blog.yorek.xyz/assets/images/android/master/ui_1_7.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">硬件加速绘制</div>
</center>


### 参考文献
1. [Android开发高手课 20-21 ui 优化](https://time.geekbang.org/column/article/80921)
2. [Google官方-图形部分文档](https://source.android.com/devices/graphics)
3. [Google官方开发者-设备兼容性](https://developer.android.com/guide/practices/compatibility?hl=zh-cn)
4. [smallestWidth 限定符适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826381&idx=1&sn=5b71b7f1654b04a55fca25b0e90a4433&chksm=80b7b213b7c03b0598f6014bfa2f7de12e1f32ca9f7b7fc49a2cf0f96440e4a7897d45c788fb&scene=21#wechat_redirect)