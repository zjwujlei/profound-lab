Android App Bundle实践及落地
==========================

Android App Bundle是Google推出的动态安装apk的方案。在splites apk的基础上，提供了按需加载，动态下发的实现。
### multiple APKs和splites apk


### Android App Bundle实现

Android App Bundle的feature apk加载实际上就是使用的splites apk的机制实现。splites apk是google在Android 5.0之后开始提供的构建多维度apk的方案，本身不支持动态下发加载feature apk，提供的是插件化类似的加载实现。

那类比插件化，我们会关注其dex、resources、so的加载，以及添加四大组件的支持。我们通过查看8.0的源码来了解其相关实现。

##### dex的加载

当前主流的插件化对dex的加载通过ClassLoader来完成，实现方式略有不同。例如滴滴的VirtualAPK，是直接将插件的dex插入到宿主的ClassLoader中（MultiDex实现加载多dex的方案）。又例如360的Replugin则替换了宿主的PathClassLoader为RePluginClassLoader类，来做加载器的代理路由，RePluginClassLoader中的loadClass函数路由到具体插件的DexClassLoader，这也是Replugin宣称的对宿主的唯一Hook点。其他的一些框架还有一些不同实现。

而splites apk对dex的加载针对不同的系统版本也有所不同。早期5.0刚出的时候，采用的是splites apk和宿主是你用同一个ClassLoader。LoadedApk内保存了宿主和splites apk的appDir，创建ClassLoader的时候会将其都作为代码路径进行创建，且没有提供update操作。因此限制了splites apk刚出时是无法做到动态加载feature apk的。

我们这里重点来看一下8.0上的实现。

##### 安装过程
Android App Bundle的安装过程在playcore库中但是并没有开源，相关代码都经过混淆。也可以查看爱奇艺重写的Qigsaw，Qigsaw的playcorelibrary是对google的playcore的爱奇艺版实现，去掉了语言信息等逻辑。我们这边先直接看playcore库：

安装入口函数：SplitInstallManager.startInstall，具体的实现在M.startInstall，返回值是一个Task<Integer>对象。我们可以在Task里添加成功、失败、完成的的监听器，而在添加监听器的时候会判断这个Task是否是complete状态，如果是会根据直接回调，特别是在已经加载过的情况下就会直接回调。

```java
public final Task<Integer> startInstall(SplitInstallRequest var1) {
        //5.0前的手机，如果有语言信息，知己诶抛出异常
        if (!var1.getLanguages().isEmpty() && VERSION.SDK_INT < 21) {
            return Tasks.a(new SplitInstallException(-5));
        } else {
            List var3 = var1.getModuleNames();
            //已经安装的处理，Qigsaw中直接不做支持
            if (this.getInstalledModules().containsAll(var3)) {
                //支持语言是否有新增
                var3 = var1.getLanguages();
                boolean var10000;
                Set var4;
                if ((var4 = this.c.b()) == null) {
                    var10000 = true;
                } else {
                    HashSet var5 = new HashSet();
                    Iterator var6 = var3.iterator();

                    while(var6.hasNext()) {
                        Locale var7 = (Locale)var6.next();
                        var5.add(var7.getLanguage());
                    }

                    var10000 = var4.containsAll(var5);
                }

                if (var10000) {
                    this.d.post(new n(this, var1));
                    //已经feature apk已经加载过，直接放回一个成功的完成的Task
                    return Tasks.a(0);
                }
            }
            //未安装调用a.a函数，传入需要安装的模块数组和对应的语言信息数组
            return this.a.a(var1.getModuleNames(), b(var1.getLanguages()));
        }
    }
```
未安装的情况下，调用a.a（）函数进行安装。
```java
    public final Task<Integer> a(Collection<String> var1, Collection<String> var2) {
        b.a("startInstall(%s,%s)", new Object[]{var1, var2});
        //i 是一个Task的包装类。
        i var3 = new i();
        //重点在这里，com.google.android.play.core.splitinstall.q是一个Runable，这里就是执行其run函数，调用Binder机制在Google Play提供的安装服务内安装成功，然后回调执行var3这个Task
        this.a.a(new com.google.android.play.core.splitinstall.q(this, var3, var1, var2, var3));
        return var3.a();
    }
```

这里安装流程需要使用到Google Play提供的服务，Android App Bundle对google的依赖也就在这一点。Qigsaw提供了splitinstaller库来完成Google Play需要做的事，将整个安装过程形成闭环。安装成功后即可正常使用feature apk中功能。


### apk devily

### bundletool

bundletool是google提供了的用来生成Android App Bundle和生成特定设备APK的工具。<a href="https://github.com/google/bundletool">bundletool</a>已经开源，google官方对bundletool的介绍可以查看<a href="https://developer.android.google.cn/studio/command-line/bundletool">这里</a>。

### 自研实现ExclusiveApp


