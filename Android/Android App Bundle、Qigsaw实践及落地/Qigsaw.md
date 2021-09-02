## Qigsaw

Qigsaw虽然说基于AAB，但个人理解上主要复用了AAB的feature（插件）定义、以此可以复用大部分打包流程，同时套用了PlayCore库的使用API，但其内部对代码、SO、资源等的加载完全是自己实现的。且实现原理和我们热修复、插件化框架基本雷同。

#### 初始化过程

初始化时调用Qigsaw.install函数，这是一个静态函数，内部创建了Qigsaw实例然后调用了onBaseContextAttached函数：

```java
private void onBaseContextAttached() {
    //省略report相关
    //init SplitLoadManager and hook PatchCLassLoader.
    SplitLoadManagerService.install(
            context,
            splitConfiguration.splitLoadMode,
            qigsawMode,
            isMainProcess,
            currentProcessName,
            splitConfiguration.workProcesses,
            splitConfiguration.forbiddenWorkProcesses);
    //data may be cached.
    SplitLoadManagerService.getInstance().clear();
    //这里会判断当前是Qigsaw模式还是AAB模式。AAB模式系统直接有处理类的查找逻辑。
    //对于类的加载，Qigsaw模式下支持独立ClassLoader模式和多ClassLoader模式。
    //injectPathClassloader中通过注入SplitDelegateClassloader来实现插件类分发的功能，
    //基于类的加载机制，用原始ClassLoader作为SplitDelegateClassloader的parent。
    //设置ClassNotFoundInterceptor，出现ClassNotFound会调用其findClass函数。
    //我们在后面具体展开。
    SplitLoadManagerService.getInstance().injectPathClassloader();
    //AABExtension，AAB的兼容，例如直接在AS中运行，我们的插件是以AAB的形式安装的。
    //qigsawMode记录了是qigsaw模式还是aab模式，是在打包过程中生成的。
    AABExtension.getInstance().clear();
    //以AAB形式安装的插件，有个问题是插件里定义的Application不会被激活生效。
    //这里通过ApplicationInfo取出所有的splitNames，然后在ComponentInfo获取到对应的Application类名。
    //ComponentInfo是在打包过程中生成的，后面打包流程中会有讲到。
    //创建插件的Application实例后调用attach函数，传入的context为宿主的ApplcaitonContext。
    AABExtension.getInstance().createAndActiveSplitApplication(context, qigsawMode);
    SplitCompat.install(context);
}
```

对于多ClassLoader模式，会调用onClassNotFound函数：

```java
private Class<?> onClassNotFound(String name) {
    //在已经load的插件中寻找。
    //由于是多ClassLoader模式，会从SplitApplicationLoaders中获取所有插件的ClassLoader遍历寻找。
    Class<?> ret = findClassInSplits(name);
    if (ret != null) {
        return ret;
    }
    //Activity、Service、Recevicer的话，由于Manifest是合并到宿主的，因此其Manifest中存在插件信息。
    //存在一种情况例如启动Service，由于Manifest中存在，因此可以接受到intent。
    //这里如果是这三种情况返回一个FakeXXX的类。
    Class<?> fakeComponent = AABExtension.getInstance().getFakeComponent(name);
    if (fakeComponent != null || isSplitEntryFragments(name)) {
        //如果是插件入口点，包括配置文件里标准的base中用到插件的Fragment，触发load所有install的插件。
        SplitLoadManagerService.getInstance().loadInstalledSplits();
        //重新调用findClassInSplits函数进行寻找。
        ret = findClassInSplits(name);
        if (ret != null) {
            SplitLog.i(TAG, "Class %s is found in Splits after loading all installed splits.", name);
            return ret;
        }
        //如果还有没找到，返回fakeComponent避免奔溃。
        if (fakeComponent != null) {
            SplitLog.w(TAG, "Split component %s is still not found after installing all installed splits, return a %s to avoid crash", name, fakeComponent.getSimpleName());
            return fakeComponent;
        }
    }
    return null;
}
```

对于单ClassLoader模式，调用onClassNotFound2：

```java
private Class<?> onClassNotFound2(String name) {
    //同样寻找fakeComponent
    Class<?> fakeComponent = AABExtension.getInstance().getFakeComponent(name);
    //如果是插件入口，则触发load操作
    if (fakeComponent != null || isSplitEntryFragments(name)) {
        SplitLoadManagerService.getInstance().loadInstalledSplits();
        try {
            //单ClassLoader模式，所有的类加载在originClassLoader中，直接调用loadClass。
            return originClassLoader.loadClass(name);
        } catch (ClassNotFoundException e) {
            if (fakeComponent != null) {
                SplitLog.w(TAG, "Split component %s is still not found after installing all installed splits,return a %s to avoid crash", name, fakeComponent.getSimpleName());
                return fakeComponent;
            }
        }
    }
    return null;
}
```



#### install过程

调用playcore中的SplitInstallManager类的函数进行install。API和AAB中一直都是playcore库，当执行具体功能的Service不同。AAB中是Google Play中的服务，qigsaw中则有个SplitInstaller库，里面实现了对应的Service端。

这里playcore库中的client端，与我们使用AIDL文件一步生成stub、proxy等不同，是完全自己实现的。熟悉Binder原理的应该知道。对于client端来说，通过bindService的回调拿到IBinder对象，将其转化成Proxy对象，通过调用Proxy对象来调用Service端能力。因此核心在Proxy类的实现，对于playcore中的proxy类为定义接口ISplitInstallServiceProxy，实现类为ISplitInstallServiceImpl：

```java
//IInterfaceProxy是playcore中对Proxy的封装，提供了descriptor、transact等功能。
public class ISplitInstallServiceImpl extends IInterfaceProxy implements ISplitInstallServiceProxy {
	//构造函数传入的remote即为服务链接时返回的远端IBinder对象。
    ISplitInstallServiceImpl(IBinder remote) {
        super(remote, "com.iqiyi.android.qigsaw.core.splitinstall.protocol.ISplitInstallService");
    }

    @Override
    public final void startInstall(String packageName, List<Bundle> moduleNames, Bundle versionCode, ISplitInstallServiceCallbackProxy callback) throws RemoteException {
        //参数需要通过序列化传递。
        Parcel data;
        (data = this.obtainData()).writeString(packageName);
        data.writeTypedList(moduleNames);
        ParcelHelper.writeToParcel(data, versionCode);
        ParcelHelper.writeStrongBinder(data, callback);
        //调用transact函数，就是调用remote的transact函数传递出去。
        //第一个入参1是函数code，按照功能函数定义顺序从1开始。
        //需要和service端对应，service端会通过switch-case进行分发。
        this.transact(1, data);
    }
    //省略其他函数
}
```

对于Install过程，在Demo的MainActivity中有触发，调用SplitInstallManager的startInstall函数。

```java

@Override
public Task<Integer> startInstall(SplitInstallRequest request) {
    //如果是已经安装，提交一个SplitInstall完成分发事件到handler，将事件分发出去，
    if (getInstalledModules().containsAll(request.getModuleNames())) {
        mMainHandler.post(new SplitInstalledDisposer(this, request));
        //这里返回一个result=0的成功Task出去，给调用方监听用。
        return Tasks.createTaskAndSetResult(0);
    } else {
        //调用mInstallService的函数，返回也是一个Task。
        return mInstallService.startInstall(request.getModuleNames());
    }
}
```

这里有些会搞混，有两个Task相关，一个是Task接口，一个是RemoteTask。Task接口我们可以认为其是任务+被观察者，对外提供结果观测能力。RemoteTask是一个Runnable，内部都是执行Binder相关函数。

例如对于startInstall函数，SplitInstallManager提供了一些逻辑，但如果发现确实需要Install了，会调用mInstallService（SplitInstallService）的startInstall函数。

```java
Task<Integer> startInstall(List<String> moduleNames) {
    playCore.info("startInstall(%s)", moduleNames);
    //通过一个TaskWrapper，来获取内部的Task传递给调用方进行结果观测。
    TaskWrapper<Integer> taskWrapper = new TaskWrapper<>();
    //StartInstallTask是对于client端Proxy对象ISplitInstallServiceImpl中startInstall函数封装。
    //内部就是调用其startInstall函数。
    mSplitRemoteManager.bindService(new StartInstallTask(this, taskWrapper, moduleNames, taskWrapper));
    return taskWrapper.getTask();
}
```

了解Binder机制的同学应该清楚，Binder机制Server端处理Client端请求的能力是有限了，默认最大支持16个线程。PlayCore是复用Google Play市场+AAB场景。想象一下一个手机中可能安装几十个APP，但提供提供服务的Google Play市场APP只有一个。如果每个Client端的链接是长存的，16个线程数肯定会爆。因此这里使用了一个RemoteManager来按需管理链接。

SplitInstallService中的mSplitRemoteManager（RemoteManager）是在其初始化的时候创建的，我们来看一下RemoteManager的构造函数：

```java
public RemoteManager(Context context, PlayCore playCore, String key, Intent intent, IRemote<T> remote, OnBinderDiedListener onBinderDiedListener) {
    //context，没什么好讲的。
    this.mContext = context;
    //PlayCore是一个日志打印类。
    this.mPlayCore = playCore;
    //Key，内部有个静态Map保存了每个key对应的Handler。当前只有"SplitInstallService"
    this.mKey = key;
    //启动SplitInstallService的Intent
    this.mSplitInstallServiceIntent = intent;
    //SplitRemoteImpl类，提供了将远端的IBinder对象转成Proxy（ISplitInstallServiceProxy）的能力。
    this.mRemote = remote;
    //死亡通知回调。
    this.mOnBinderDiedListenerWkRef = new WeakReference<>(onBinderDiedListener);
}
```

这里有几个点：

1. 入参mKey中说道了Handler，这是进一步减少Google Play 中Service压力的设计。这里通过Handler机制，调用的RemoteManager中都通过Handler机制转为Handler对应线程中执行，由于调用Binder函数是阻塞的，因此保证了同一时间只有一个进行中的调用。
2. mSplitInstallServiceIntent，这个Intent的Action为"com.iqiyi.android.play.core.splitinstall.BIND_SPLIT_INSTALL_SERVICE"，实际配合Google Play使用是肯定不是这个。
3. mOnBinderDiedListenerWkRef，我们知道Binder机制中是有死亡通知的，可以通过IBinder的linkToDeath函数注册。

对RemoteManager有所了解之后我们来看一下之前调用的bindService函数：

```java
public void bindService(RemoteTask task) {
    //这里里很简单，BindService操作也是Binder通信，因此也需要进行管控。
    //通过封装成BindServiceTask（也是RemoteTask），扔给Hander调度执行。
    post(new BindServiceTask(this, task));
}
```

BindServiceTask这个RemoteTask的execute函数就是调用bindServiceInternal：

```java
void bindServiceInternal(RemoteTask remoteTask) {
    //如果还没有绑定过且不是真在绑定Service，
    if (this.mIInterface == null && !this.mBindingService) {
        mPlayCore.info("Initiate binding to the service.");
        //将remoteTask保存到mPendingTasks中后续会用到。
        this.mPendingTasks.add(remoteTask);
        //创建ServiceConnectionImpl，就是一个ServiceConnection。
        //在其onServiceConnected和onServiceDisconnected中分别在RemoteManager中post
        //ServiceConnectedTask和ServiceDisconnectedTask，这两个都是RemoteTask。
        //ServiceConnectedTask中的execute函数主要就是通过mRemote（构造函数传入）获取proxy对象，
        //然后取出mPendingTasks中的Task执行。同时会调用RemoteManager的linkToDeath添加死亡通知。
        //ServiceDisconnectedTask做的则是清理重置。
        this.mServiceConnection = new ServiceConnectionImpl(this);
        this.mBindingService = true;
        //调用bindService绑定Service
        if (!this.mContext.bindService(this.mSplitInstallServiceIntent, this.mServiceConnection, Context.BIND_AUTO_CREATE)) {
            this.mPlayCore.info("Failed to bind to the service.");
            this.mBindingService = false;
            //如果没有成功，通过TaskWrapper将错误异常回调给调用方（前面说的Task）。
            for (RemoteTask splitRemoteTask : mPendingTasks) {
                TaskWrapper taskWrapper = splitRemoteTask.getTask();
                if (taskWrapper != null) {
                    taskWrapper.setException(new RemoteServiceException());
                }
            }
            this.mPendingTasks.clear();
        }
    } else {
        //如果正在绑定，添加进mPendingTasks，绑定成功会进行调用。
        if (this.mBindingService) {
            this.mPlayCore.info("Waiting to bind to the service.");
            this.mPendingTasks.add(remoteTask);
            return;
        }
        //如果已经绑定的，直接调用其run函数即可。
        remoteTask.run();
    }
}
```

至此，PlayCore侧的调用就完全理清了。其实在Qigsaw场景，可以不用那么复杂，Binder的Service端也是当前APP自用的，很少会冲突。

然后我们看一下在Qigsaw的Service端startInstall函数是如何实现的，是在com.iqiyi.android.qigsaw.core.splitinstall.remote.SplitInstallService中，跟踪调用流程，最终调用了SplitInstallSupervisorImpl的startInstall函数：

```java
public void startInstall(List<Bundle> moduleNames, Callback callback) {
    //这个就是从Bundle中取出Stirng的splitName
    List<String> moduleNameList = unBundleModuleNames(moduleNames);
    //对内部状态的异常检测。
    //包括如果moduleNames对应的插件是否都已经install，对于已经install的直接load即可。
    int errorCode = onPreInstallSplits(moduleNameList);
    if (errorCode != SplitInstallInternalErrorCode.NO_ERROR) {
        callback.onError(bundleErrorCode(errorCode));
    } else {
        //通过getNeed2BeInstalledSplits函数筛选出没有install的插件
        List<SplitInfo> needInstallSplits = getNeed2BeInstalledSplits(moduleNameList);
        //check network status
        //如果需要install的插件有非builtIn的，需要网络进行下载。
        if (!isAllSplitsBuiltIn(needInstallSplits) && !isNetworkAvailable(appContext)) {
            callback.onError(bundleErrorCode(SplitInstallInternalErrorCode.NETWORK_ERROR));
            return;
        }
        //进行下载并安装，包括BuiltIn的插件从APK文件中释放出来。
        //对于Install过程其实是签名校验+插件文件释放到指定目录，这里不再展开。
        startDownloadSplits(moduleNameList, needInstallSplits, callback);
    }
}
```



#### 代码load过程

loader过程就是将之前下载释放准备好的feature.apk中代码资源等加载进来。其相关API在SplitLoadManager中定义，实现类看SplitLoadManagerImpl，这个在前面初始化过程中有提到。我们可以从启动加载已安装插件的函数preloadInstalledSplits为入口看看其load过程。

```java
public void preloadInstalledSplits(Collection<String> splitNames) {
    //对于非qigsaw模式（前面说道的直接AS中run-as），系统会自动加载内置的splitApk。
    if (!qigsawMode) {
        return;
    }
    if (isProcessAllowedToWork()) {
       	//如果当前进程允许，则进行load
        loadInstalledSplitsInternal(splitNames);
    }
}
```

```java
private void loadInstalledSplitsInternal(Collection<String> splitNames) {
    //省略
    //获取需要load的splits的SplitInfo，是我们在assets目录下qigsaw_${version}.json中记录的信息。
    Collection<SplitInfo> splitInfoList;
    if (splitNames == null) {
        splitInfoList = manager.getAllSplitInfo(getContext());
    } else {
        splitInfoList = manager.getSplitInfos(getContext(), splitNames);
    }
    //省略
    //创建split load的Intent，通过Intent保存splitName、apk、added-dex、native-lib-dir
    //和dex-opt-dir（oat文件目录）。
    List<Intent> splitFileIntents = createInstalledSplitFileIntents(splitInfoList);
    if (splitFileIntents.isEmpty()) {
        SplitLog.w(TAG, "There are no installed splits!");
        return;
    }
    //创建loadTask，并执行run函数
    createSplitLoadTask(splitFileIntents, null).run();
}
```

我们前面说道，qigsaw存在单classloader模式和多classloader模式，对与这两种其loadCode的过程不一样，createSplitLoadTask中分别使用SplitLoadTaskImpl和SplitLoadTaskImpl2进行load。其核心是调用SplitLoadHandler的loadSplits函数：

```java
private void loadSplits(final OnSplitLoadFinishListener loadFinishListener) {
    //省略
    for (Intent splitFileIntent : splitFileIntents) {
        long loadStart = System.currentTimeMillis();
        //获取SplitInfo对象。
        final String splitName = splitFileIntent.getStringExtra(SplitConstants.KET_NAME);
        SplitInfo info = infoManager.getSplitInfo(getContext(), splitName);
        //省略
        SplitBriefInfo splitBriefInfo = new SplitBriefInfo(info.getSplitName(), info.getSplitVersion(), info.isBuiltIn());
        //省略
        String splitApkPath = splitFileIntent.getStringExtra(SplitConstants.KEY_APK);
       //省略
        String dexOptPath = splitFileIntent.getStringExtra(SplitConstants.KEY_DEX_OPT_DIR);
        //省略
        String nativeLibPath = splitFileIntent.getStringExtra(SplitConstants.KEY_NATIVE_LIB_DIR);
        //省略
        List<String> addedDexPaths = splitFileIntent.getStringArrayListExtra(SplitConstants.KEY_ADDED_DEX);
        ClassLoader classLoader;

        // check if need compat native lib path on android 5.x
        SplitLog.d(TAG, "split name: %s, origin native path: %s", splitName, nativeLibPath);
        nativeLibPath = mapper.map(splitName, nativeLibPath);
        SplitLog.d(TAG, "split name: %s, mapped native path: %s", splitName, nativeLibPath);

        try {
            //这里调用不同SplitLoadTaskImpl或者SplitLoadTaskImpl2进行代码（包括SO）加载。
            //这里其实就是创建一个Split的classloader或者将额外的代码路径添加到主classloader中。
            //这里面的逻辑就和其他插件化框架一致了。
            classLoader = splitLoader.loadCode(splitName,
                                               addedDexPaths, dexOptPath == null ? null : new File(dexOptPath),
                                               nativeLibPath == null ? null : new File(nativeLibPath),
                                               info.getDependencies()
                                              );
        } catch (SplitLoadException e) {
            SplitLog.printErrStackTrace(TAG, e, "Failed to load split %s code!", splitName);
            loadErrorInfos.add(new SplitLoadError(splitBriefInfo, e.getErrorCode(), e.getCause()));
            continue;
        }
        //开始对插件Application进行激活。
        //前面初始化过程中，对于非Qigsaw模式下，也会对spilt内的Application进行激活。
        //是通过在ComponentInfo中找到对于的类进行激活的。
        //这里其实也是通过调用AABExtension中的逻辑来获取Split的Application实例。
        //这里省略异常处理
        final Application application = activator.createSplitApplication(classLoader, splitName);
        //activateSplit函数主要做三步：
        //1.调用application的attach函数。
        //2.创建和激活真正的Provider，这里我们等阅读完代码详细讲解。
        //3.调用application的onCreate函数。
       	//这里省略异常处理
        activateSplit(splitName, splitApkPath, application, classLoader);
        
       //省略
    }
    //省略
}
```

这里我们先说一下activateSplit中的Provider相关。对启动流程有所了解的话应该知道，对于Provider其启动是在application之前的。启动过程中系统进程调用app进程的bindApplication会传递ApplicationInfo信息，这里面包括了在manifest里面注册的Provider，然后Provider的onCreate等函数是在Application之前调用的，也因此我们有时SDK的初始化会用Provider实现。正因为如此，在Qigsaw中对于Provider都会生成一个名为providerName + "_Decorated_" + splitName并继承SplitContentProvider的类，这是一个代理类，并注册到宿主的Manifest中。这其实就和一些插件化框架通过提前占坑来做到能新增四大组件实现类似。在未加载split之前，通过这个代理类保证应用启动不崩并进行占坑。在加载split是，取出真正的provider对象进行实例化并通过attachInfo函数替换代理Provider。

#### 资源的load

资源的load是在Qigsaw的getResources函数中触发Qigsaw.onApplicationGetResources(super.getResources());。其核心代码逻辑就是调用原始resource对象的addAssets函数添加额外的资源目录。这个比较简单，就不展开了。



#### 

