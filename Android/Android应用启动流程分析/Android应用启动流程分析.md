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
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
        ...
    }

```

ActivityManager.getService()就是获取系统服务ActivityManagerService实例接口对象。其实所谓的页面切换startActivity函数做的事情就是讲intent对象扔给ActivityManagerService处理。