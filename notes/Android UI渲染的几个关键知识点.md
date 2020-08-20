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
   
- **mipmap和drawable职责的区分**
  
  谷歌建议将启动图标放置在mipmap下，而将其他资源图片放置在drawable对应目录下，

  

#### 2.1.3 刘海屏适配


### 三

### 参考文献
1. [Android开发高手课 20-21 ui 优化](https://time.geekbang.org/column/article/80921)
2. [Google官方-图形部分文档](https://source.android.com/devices/graphics)
3. [Google官方开发者-设备兼容性](https://developer.android.com/guide/practices/compatibility?hl=zh-cn)
4. [smallestWidth 限定符适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826381&idx=1&sn=5b71b7f1654b04a55fca25b0e90a4433&chksm=80b7b213b7c03b0598f6014bfa2f7de12e1f32ca9f7b7fc49a2cf0f96440e4a7897d45c788fb&scene=21#wechat_redirect)