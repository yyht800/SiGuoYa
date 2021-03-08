## Android进阶解密第六章——AMS（基于Android10）

### 1.AMS的几个版本差异

以下版本差异都是以Activity的启动处，Instrumentation的execStartActivity方法内部。

#### 1.1 Android 7.0版本

通过ActivityManagerNative.getDefault()获取一个ActivityManagerProxy（简称AMP），AMP是AMS的代理类。AMP是AMN的内部类，且AMP和AMN都实现了IActivityManager的接口，用于实现代理机制和Binder通信。

#### 1.2 Android 8.0版本

在8.0版本去除了AMP，ActivityManager的getService()方法，获取IBinder类型的AMS引用。

#### 1.3 Android 10版本

在10版本中，新增了ATMS（ActivityTaskManagerService）将AMS中的部分功能转移到ATMS了，通过ActivityTaskManager.getService()获取ATMS的对象。



### 2. AMS的启动过程

起点在SystemServer的main方法，里面只有一行代码继续调用run方法，这里的代码逻辑可以看系统启动的章节，这里直接看核心代码

```java
//SystemServer.java

private void run(){
 		...
    try{
      //1. 启动引导服务
      startBootstrapServices();
      //2. 启动核心服务
      startCoreServices();
      //3. 启动其他服务
      startOtherServices();
    }
  ....
}
```

看引导服务startBootstrapServices

```java
private void startBootstrapServices(){
  		...
    		ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
  			mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
  
 ...	
}
```

这里也印证了，AMS将部分功能转移到ATMS中，ATMS传入的对象类型也变成了ActivityTaskManagerService.Lifecycle.class,而在AMS中也有一个ActivityManagerService.Lifecycle的对象，都是集成自SystemService。

当调用 SystemService类型的 service的 onStart方法时，实际上是调用了 AMS 的 start 方法 。 注释 3 处的 Lifecycle 的 getService 方住 返回 AMS 实例，这样我们就知道 SystemServer 的 startBootstrapServices 方法的注释 l 处 mSystemServiceManager.startService(..).getService()实际 得到的就是 ATMS 实例，而AMS的启动需要一个ATMS的实例对象。到这里AMS的创建就完成了。

### 4. AMS和应用程序的关系

总结：AMS会判断进程是否存在，不存在则让Zygote创建一个。

### 5. AMS中一些重要的类

由于Activity相关的迁移到了ATMS中，这个部分还是看Activity的启动那个章节吧。





PS：随着对版本的差异性，比较我甚至怀疑后续可能AMS可能会换个名字，逐步将四大组件的服务特性拆出去，最后剩个壳子了。







