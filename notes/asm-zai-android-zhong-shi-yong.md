# Android 中ASM怎么使用

## 前情提要

最近准备做一个APM，调阅资料发现居然使用的是插桩的骚操作。一直想玩但是没有玩过，记录下操作的过程和坑点。 本文可能涉及的点，Android中的自定义gradle，ASM的使用等。

## 一、知识点预热

### 1.1 什么是字节码插桩

[Android官网编译的过程](https://developer.android.com/studio/build/index.html?hl=zh-cn#build-process)中如图1-1，没有老版本的清晰如图1-2所示，先将xxx.java变成.class,然后再变成.dex文件，插桩就是对class文件的进行编辑。字节码插桩一般常用的有AspectJ，ASM，Redex（[Redex方案](https://github.com/facebook/redex)是Facebook的优化方案直接对dex进行操作，所见即所得）

 ![](https://developer.android.com/images/tools/studio/build-process_2x.png?hl=zh-cn)  
1-1 新版打包流程

 ![](https://user-gold-cdn.xitu.io/2017/3/2/35a4d886bc51ec6be29456eadd4b1fd2.png)  
1-2 老版打包流程

### 1.2 什么是Transform

Transform是Android gradle plugin 1.5开始引入的概念。 我们先从如何引入Transform依赖说起，首先我们需要编写一个自定义插件，然后在插件中注册一个自定义Transform。这其中我们需要先通过gradle引入Transform的依赖，这里有一个坑，Transform的库最开始是独立的，后来从2.0.0版本开始，被归入了Android编译系统依赖的gradle-api中。 它的原理如下图1-3所示。

 ![](https://upload-images.jianshu.io/upload_images/13278007-e3181a3e9742296f?imageMogr2/auto-orient/strip|imageView2/2/w/900/format/webp)  
1-3 Transform的原理

每个Transform其实都是一个gradle task，Android编译器中的TaskManager将每个Transform串连起来，第一个Transform接收来自javac编译的结果，以及已经拉取到在本地的第三方依赖（jar. aar），还有resource资源，注意，这里的resource并非android项目中的res资源，而是asset目录下的资源。这些编译的中间产物，在Transform组成的链条上流动，每个Transform节点可以对class进行处理再传递给下一个Transform。我们常见的混淆，Desugar等逻辑，它们的实现如今都是封装在一个个Transform中，而我们自定义的Transform，会插入到这个Transform链条的最前面。

但其实，上面这幅图，只是展示Transform的其中一种情况。而Transform其实可以有两种输入，一种是消费型的，当前Transform需要将消费型型输出给下一个Transform，另一种是引用型的，当前Transform可以读取这些输入，而不需要输出给下一个Transform，比如Instant Run就是通过这种方式，检查两次编译之间的diff的。

最终，我们定义的Transform会被转化成一个个TransformTask，在Gradle编译时调用。

## 二、思路整理

在动手之前先整理一下思路,已知我们需要在编译期 修改目标class文件，可以借助字节码插桩的技术，这里选用ASM进行字节码插桩。他的逆向顺序应该是 1. 自定义Gradle插件 2. 写一个Transform脚本，将其放入自定义的Gradle 3. 在Transform中完成自己的插桩代码 4. 进行编译

## 三、自定义Gradle

具体过程可以参考文章[在AndroidStudio中自定义Gradle插件](https://blog.csdn.net/huachao1001/article/details/51810328)，嫌麻烦的，如果只是在当前项目中使用**BuildSrc**的目录。 另外文章有几个没有提到的点，在AS的编译器里，在groovy目录下创建的是xxx.groovy的文件，由于groovy的文件标志和class文件的标志几乎差不多，语法上也可以使用java，所以参考demo写的时候会出现某些语法不能用的状况，两个标志一个是方的，一个是圆角。

第二个，编译项目的时候要先clean一下，如果报错在脚本module中无需在意。之后构建成功。

关于混淆的 如果引用关系一直显示找不到或者报错的话，看下pack是否成功引入，没有的话手动添加一下 更新完代码记得build 在gradle 指令里面

## 四、自定义Transform

### 4.1 自定义Transform

基本概念如上面的内容所述，具体实现如下：

```text
public class LifeCyclerTransform extends Transform {

    @Override
    String getName() {
        //返回当前的名字即可
        return "LifeCyclerTransform"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        //输入类型 一种是Classes，另一种是Resources，
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        //Transform表明作用域，无特殊需求一般选择全部项目
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        //是否开启增量编译
        return false
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)
        System.out.println("======LifeCyclerTransform init======")
        long startTime = System.currentTimeMillis()

        //当前是否是增量编译，
        boolean isIncremental = transformInvocation.isIncremental()
        //OutputProvider管理输出路径
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider()
        if (outputProvider != null) {//删除之前的输出
            outputProvider.deleteAll()
        }
        //引用型输入，无需输出。
        Collection<TransformInput> referencedInputs = transformInvocation.getReferencedInputs();

        transformInvocation.inputs.each { TransformInput transformInput ->

            //遍历文件夹的文件
            transformInput.directoryInputs.each { DirectoryInput directoryInput ->
                //处理directoryInputs
                handleDirectoryInput(directoryInput, outputProvider)
            }
            //遍历jar包文件
            transformInput.jarInputs.each { JarInput jarInput ->
                //处理jarInputs
                handleJarInputs(jarInput, outputProvider)
            }

        }


        long endTime = System.currentTimeMillis()
        System.out.println(String.format("======LifeCyclerTransform end cost %s======", endTime - startTime))
    }

    void handleDirectoryInput(DirectoryInput directoryInput, TransformOutputProvider outputProvider) {
        System.out.println("handleDirectoryInput method init")

        //是否是目录
        if (directoryInput.file.isDirectory()) {
            //列出目录所有文件（包含子文件夹，子文件夹内文件）
            directoryInput.file.eachFileRecurse { File file ->
                def name = file.name

                if (checkClassFile(name)) {
                    println '----------- deal with "class" file <' + name + '> -----------'
                    ClassReader classReader = new ClassReader(file.bytes)
                    ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
                    ClassVisitor cv = new LifecycleClassVisitor(classWriter)
                    classReader.accept(cv, EXPAND_FRAMES)
                    byte[] code = classWriter.toByteArray()
                    FileOutputStream fos = new FileOutputStream(
                            file.parentFile.absolutePath + File.separator + name)
                    fos.write(code)
                    fos.close()
                }
            }
        }
        //处理完输入文件之后，要把输出给下一个任务
        def dest = outputProvider.getContentLocation(directoryInput.name,
                directoryInput.contentTypes, directoryInput.scopes,
                Format.DIRECTORY)
        FileUtils.copyDirectory(directoryInput.file, dest)
    }

    void handleJarInputs(JarInput jarInput, TransformOutputProvider outputProvider) {
        if (jarInput.file.getAbsolutePath().endsWith(".jar")) {
            //重名名输出文件,因为可能同名,会覆盖
            def jarName = jarInput.name
            def md5Name = DigestUtils.md5Hex(jarInput.file.getAbsolutePath())
            if (jarName.endsWith(".jar")) {
                jarName = jarName.substring(0, jarName.length() - 4)
            }
            JarFile jarFile = new JarFile(jarInput.file)
            Enumeration enumeration = jarFile.entries()
            File tmpFile = new File(jarInput.file.getParent() + File.separator + "classes_temp.jar")
            //避免上次的缓存被重复插入
            if (tmpFile.exists()) {
                tmpFile.delete()
            }
            JarOutputStream jarOutputStream = new JarOutputStream(new FileOutputStream(tmpFile))
            //用于保存
            while (enumeration.hasMoreElements()) {
                JarEntry jarEntry = (JarEntry) enumeration.nextElement()
                String entryName = jarEntry.getName()
                ZipEntry zipEntry = new ZipEntry(entryName)
                InputStream inputStream = jarFile.getInputStream(jarEntry)
                //插桩class
                if (checkClassFile(entryName)) {
                    //class文件处理
                    println '----------- deal with "jar" class file <' + entryName + '> -----------'
                    jarOutputStream.putNextEntry(zipEntry)
                    ClassReader classReader = new ClassReader(IOUtils.toByteArray(inputStream))
                    ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
                    ClassVisitor cv = new LifecycleClassVisitor(classWriter)
                    classReader.accept(cv, EXPAND_FRAMES)
                    byte[] code = classWriter.toByteArray()
                    jarOutputStream.write(code)
                } else {
                    jarOutputStream.putNextEntry(zipEntry)
                    jarOutputStream.write(IOUtils.toByteArray(inputStream))
                }
                jarOutputStream.closeEntry()
            }
            //结束
            jarOutputStream.close()
            jarFile.close()
            def dest = outputProvider.getContentLocation(jarName + md5Name,
                    jarInput.contentTypes, jarInput.scopes, Format.JAR)
            FileUtils.copyFile(tmpFile, dest)
            tmpFile.delete()
        }
    }

    /**
     * 检查class文件是否需要处理
     * @param fileName
     * @return
     */
    static boolean checkClassFile(String name) {
        //只处理需要的class文件
        return (name.endsWith(".class") && !name.startsWith("R\$")
                && !"R.class".equals(name) && !"BuildConfig.class".equals(name)
                && "androidx/fragment/app/FragmentActivity.class".equals(name))
    }
}
```

在定义玩Transform之后需要将它放入编译目录中，此时需要定义一个Plugin，完成后编译内容将自动完成， 你可以在build时候看到相关日志。 上文中有关于ASM的应用下面第五节会解释。

```text
public class PerfPlugin implements Plugin<Project> {


    @Override
    void apply(Project project) {
        System.out.println("========================")
        System.out.println("mother fuck apm!")
        def android = project.extensions.getByType(AppExtension)

        android.registerTransform(new LifeCyclerTransform())

        System.out.println("========================")
    }
}
```

### 4.2 Transform的优化：增量与并发

如上文所示，可以编译之后，你再添加几个Transform进去，可以明显的感受到编译速度变慢了，接下来我们就可以更近一步的使用Transform的高级用法，首先要设置增量编译为True。

```text
@Override 
public boolean isIncremental() {
    return true;
}
```

虽然开启了增量编译，但也并非每次编译过程都是支持增量的，毕竟一次clean build完全没有增量的基础，所以，我们需要检查当前编译是否是增量编译。

如果不是增量编译，则清空output目录，然后按照前面的方式，逐个class/jar处理 如果是增量编译，则要检查每个文件的Status，Status分四种，并且对这四种文件的操作也不尽相同.

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7MfKcpYNP8NWww77qGsiaVyibgxNvdAdQBv9JINmXuyee9ud7u9d03sudLiabkpE4KrjsAm14nQtxNw/640?wx_fmt=png)

* NOTCHANGED: 当前文件不需处理，甚至复制操作都不用；
* ADDED、CHANGED: 正常处理，输出给下一个任务；
* REMOVED: 移除outputProvider获取路径对应的文件。

代码大致如下：

```text
                switch(status) {
                    case NOTCHANGED:
                        break;
                    case ADDED:
                    case CHANGED:
                        transformJar(jarInput.getFile(), dest, status);
                        break;
                    case REMOVED:
                        if (dest.exists()) {
                            FileUtils.forceDelete(dest);
                        }
                        break;
                }
```

实现了增量编译后，我们最好也支持并发编译，并发编译的实现并不复杂，只需要将上面处理单个jar/class的逻辑，并发处理，最后阻塞等待所有任务结束即可。

```text
private WaitableExecutor waitableExecutor = WaitableExecutor.useGlobalSharedThreadPool();


//异步并发处理jar/class
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveJar(srcJar, destJar);
    return null;
});
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveSingleClassToFile(file, outputFile, inputDirPath);
    return null;
});  


//等待所有任务结束
waitableExecutor.waitForTasksWithQuickFail(true);
```

## 五、ASM的使用

简单来说就是利用ClassVisitor找到对应的class文件，然后利用MethodVisitor对插入对应的字节码，通过ASM Bytecode Outline插件生成代码。

```text
 @Override
    public void visitCode() {
        super.visitCode();
        //方法执行前插入
        mv.visitLdcInsn("Lifecycler");
        mv.visitLdcInsn("---->%s():%s");
        mv.visitInsn(ICONST_2);
        mv.visitTypeInsn(ANEWARRAY, "java/lang/Object");
        mv.visitInsn(DUP);
        mv.visitInsn(ICONST_0);
        mv.visitVarInsn(ALOAD, 0);
        mv.visitFieldInsn(GETFIELD, "com/cn/performance_gradle_plugin/LifecyclerMethodVisitor", "methodName", "Ljava/lang/String;");
        mv.visitInsn(AASTORE);
        mv.visitInsn(DUP);
        mv.visitInsn(ICONST_1);
        mv.visitVarInsn(ALOAD, 0);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Object", "getClass", "()Ljava/lang/Class;", false);
        mv.visitInsn(AASTORE);
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/String", "format", "(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;", false);
        mv.visitMethodInsn(INVOKESTATIC, "android/util/Log", "e", "(Ljava/lang/String;Ljava/lang/String;)I", false);
        mv.visitInsn(POP);
    }
```

## 参考文献

1. [在AndroidStudio中自定义Gradle插件](https://blog.csdn.net/huachao1001/article/details/51810328)
2. [一起玩转Android项目中的字节码](http://quinnchen.cn/2018/09/13/2018-09-13-asm-transform/)
3. [【Android】函数插桩（Gradle + ASM）](https://www.jianshu.com/p/16ed4d233fd1)

