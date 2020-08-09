# Android 中ASM怎么使用
### 前情提要
最近准备做一个APM，调阅资料发现居然使用的是插桩的骚操作。一直想玩但是没有玩过，记录下操作的过程和坑点。
本文可能涉及的点，Android中的自定义gradle，ASM的使用等。

### 一、思路整理
在动手之前先整理一下思路，

关于混淆的
如果引用关系一直显示找不到或者报错的话，看下pack是否成功引入，没有的话手动添加一下
更新完代码记得build 在gradle 指令里面

### 参考文献
1. [在AndroidStudio中自定义Gradle插件](https://blog.csdn.net/huachao1001/article/details/51810328)
2. [一起玩转Android项目中的字节码](http://quinnchen.cn/2018/09/13/2018-09-13-asm-transform/)
