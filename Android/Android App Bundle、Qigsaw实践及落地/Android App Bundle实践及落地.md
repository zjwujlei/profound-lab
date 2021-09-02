Android App Bundle实践及落地
==========================

Android App Bundle是Google推出的动态安装apk的方案。在SplitApk的基础上，提供了按需加载，动态下发的实现。

### SplitApk

了解splitapk的话我们从LoadedApk入手，LoadedApk有如下参数

```java
//split模块名称
private String[] mSplitNames;
//split模块apk文件路径
private String[] mSplitAppDirs;
//split模块中资源路径
private String[] mSplitResDirs;
//对应的classloader
private String[] mSplitClassLoaderNames;
```

我们从这些值是如何产生的和如何使用两方面来了解。

###### 如何产生

这里如果了解应用启动流程的话会有看到一些。我们以倒推的形式看一下：

```java
public LoadedApk(ActivityThread activityThread, ApplicationInfo aInfo,
                 CompatibilityInfo compatInfo, ClassLoader baseLoader,
                 boolean securityViolation, boolean includeCode, boolean registerPackage) {

    mActivityThread = activityThread;
    //LoadedApk构造函数里在setApplicationInfo函数中通过ApplicationInfo传递进行赋值。
    setApplicationInfo(aInfo);
    //省略
}
```

在启动流程中，LoadedApk是在ActivityThread.handleBindApplication流程中调用getPackageInfoNoCheck(data.appInfo, data.compatInfo)创建，data就是AppBindData，data.appInfo就是ApplicationInfo。是ActivityManagerService通过Binder机制调用ApplicationThread.bindApplication传入。

启动流程中当app进程创建并执行ActivityThread的main函数时，创建了ActivityThread并调用其attach函数时，是通过Binder机制调用ActivityManagerService的attachApplication，在其对应的ProcessRecord仍旧有效是，会调用ApplicationThread.bindApplication并传入ProcessRecord中持有的ApplicationInfo。

至此我们追溯到这些SplitApk相关信息是系统在做APP进程初始化的时候收集的。

###### 如何使用

我们回到LoadedApk的创建的setApplicationInfo函数中：

```java
 private void setApplicationInfo(ApplicationInfo aInfo) {
        //省略
     	//这里留一个adjustNativeLibraryPaths函数，这个函数与Split无关但可以了解下,是用来决定abi的。
        aInfo = adjustNativeLibraryPaths函数(aInfo);
        //设置Split相关信息
        mSplitNames = aInfo.splitNames;
        mSplitAppDirs = aInfo.splitSourceDirs;
        mSplitResDirs = aInfo.uid == myUid ? aInfo.splitSourceDirs : aInfo.splitPublicSourceDirs;
        mSplitClassLoaderNames = aInfo.splitClassLoaderNames;
		//如果使用隔离模式，及每个split独立的ClassLoader
        if (aInfo.requestsIsolatedSplitLoading() && !ArrayUtils.isEmpty(mSplitNames)) {
            mSplitLoader = new SplitDependencyLoaderImpl(aInfo.splitDependencies);
        }
    }
```




### Android App Bundle实现

Android App Bundle的feature apk加载实际上就是使用的splites apk的机制实现。splites apk是google在Android 5.0之后开始提供的构建多维度apk的方案，本身不支持动态下发加载feature apk，提供的是插件化类似的加载实现。

那类比插件化，我们会关注其dex、resources、so的加载，以及添加四大组件的支持。我们通过查看8.0的源码来了解其相关实现。

##### dex的加载

当前主流的插件化对dex的加载通过ClassLoader来完成，实现方式略有不同。例如滴滴的VirtualAPK，是直接将插件的dex插入到宿主的ClassLoader中（MultiDex实现加载多dex的方案）。又例如360的Replugin则替换了宿主的PathClassLoader为RePluginClassLoader类，来做加载器的代理路由，RePluginClassLoader中的loadClass函数路由到具体插件的DexClassLoader，这也是Replugin宣称的对宿主的唯一Hook点。其他的一些框架还有一些不同实现。

而splites apk对dex的加载针对不同的系统版本也有所不同。早期5.0刚出的时候，采用的是splites apk和宿主是你用同一个ClassLoader。LoadedApk内保存了宿主和splites apk的appDir，创建ClassLoader的时候会将其都作为代码路径进行创建，且没有提供update操作。因此限制了splites apk刚出时是无法做到动态加载feature apk的。

我们这里重点来看一下8.0上的实现。




### apk devily

### bundletool

bundletool是google提供了的用来生成Android App Bundle和生成特定设备APK的工具。<a href="https://github.com/google/bundletool">bundletool</a>已经开源，google官方对bundletool的介绍可以查看<a href="https://developer.android.google.cn/studio/command-line/bundletool">这里</a>。

### 

