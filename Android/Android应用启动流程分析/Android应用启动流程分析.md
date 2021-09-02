Android应用启动流程分析
=================

我们在手机的launcher上，点击一个应用图标打开一个app，这个launcher会调用startActivity(Intent)函数，Intent里就包含对应app的信息，通过binder机制会传递到ActivityManagerService中，ActivityManagerService是系统服务，一般都是在手机启动的时候就会创建。ActivityManagerService里会根据Intent中的具体信息，唤起对应app并打开对应Activity。

事实上，会引起App启动的，并不仅仅是启动Activity。Android的四大组件都会引起App的启动。我们这里以launcher上点击应用图标的流程为例完整的分析应用启动流程，其他三大组件的情形都类似。

### startActivity

我们可以通过查看google源码中/packages/apps/launcher中了解launcher的具体实现，其页面切换也是通过startActivity函数。

整个流程关键实现是在Instrumentation的execStartActivity函数

```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        ...
    }

```

ActivityTaskManager.getService()就是获取系统服务ActivityTaskManagerService实例接口对象（老版本是ActivityManagerService）。其实所谓的页面切换startActivity函数做的事情就是讲intent对象扔给ActivityTaskManagerService处理，此时是在系统进程，并不是launcher进程。

我们要跳转的页面，是在另外一个APP中的，并不是launcher进程，因此首先需要创建进程。从Zygote中fork一个进程出来，此时会调用ActivityThread的main函数：

```java
public static void main(String[] args) {
    //省略部分代码
    //这里就是Loop机制，但我们自己实现的话都是在Thread的run函数中进行的。
    //我们说ActivityThread是主线程，其实有一点不完全准确，ActivityThread并不是线程。
    //这个main函数是跑在一个线程中的，通过Looper.prepareMainLooper()函数将这个线程指定为主线程。
    Looper.prepareMainLooper();
	//省略部分代码
    //创建了ActivityThread后进行attach
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
	//创建Handler，处理发送的事件。
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
	//省略部分代码
    Looper.loop();
	//如果跳出了Looper循环，说明退出了。
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

这里的重点是thread.attach函数，对于非系统进程，其实就是调用ActivityManager.getService().attachApplication(mAppThread, startSeq);函数。通过Binder机制，回到系统进程，调用ActivityManagerService的attachApplication(mAppThread, startSeq);函数。其中核心是attachApplicationLocked函数，

```java
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
                                        int pid, int callingUid, long startSeq) {


    ProcessRecord app;
	//获取到ProcessRecord，这个是在之前在系统进程通过zygote创建app进程是进行记录的，
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
       //省略对app的校验，如果已经无效则赋值null。
    } else {
        app = null;
    }

    // It's possible that process called attachApplication before we got a chance to
    // update the internal state.
    if (app == null && startSeq > 0) {
        final ProcessRecord pending = mProcessList.mPendingStarts.get(startSeq);
        if (pending != null && pending.startUid == callingUid && pending.startSeq == startSeq
            && mProcessList.handleProcessStartedLocked(pending, pid, pending
                                                       .isUsingWrapper(),
                                                       startSeq, true)) {
            app = pending;
        }
    }

    //省略如果app == null return false的逻辑

    
	//这里注册死亡监听
    final String processName = app.processName;
    try {
        AppDeathRecipient adr = new AppDeathRecipient(
            app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        mProcessList.startProcessLocked(app,
                                        new HostingRecord("link fail", processName),
                                        ZYGOTE_POLICY_FLAG_EMPTY);
        return false;
    }

    //省略一大堆代码，看不下去了。核心是这一个，将收集好的信息通过bindApplication函数传回app进程。
    //thread是app调用是传入的IApplicationThread。
    thread.bindApplication(processName, appInfo, providerList,
                        instr2.mClass,
                        profilerInfo, instr2.mArguments,
                        instr2.mWatcher,
                        instr2.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.compat, getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.mDisabledCompatChanges);
    return true;
}
```

回到App进程里，IApplicationThread是AIDL，其实现在ApplicationThread类，对应的bindApplication函数就是将数据封装成AppBindData，然后通过hander机制发送H.BIND_APPLICATION事件，handler接受到该事件会调用handleBindApplication(data);函数，重点逻辑如下：

1. data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);创建loadedApk。
2. app = data.info.makeApplication(data.restrictedBackupMode, null);创建Application。
3. final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);创建appContext。
4. 创建Instrumentation及其ContextImpl，并进行init初始化。
5. 调用installContentProviders(app, data.providers);做provider的初始化并调用onCreate函数。
6. mInstrumentation.onCreate(data.instrumentationArgs);调用Instrumentation的onCreate函数。
7. mInstrumentation.callApplicationOnCreate(app);调用application的onCreate函数。

