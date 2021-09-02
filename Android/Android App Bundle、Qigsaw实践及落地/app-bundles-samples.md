## app-bundles-samples

<a href="https://github.com/android/app-bundle-samples">app-bundles-samples</a>是官方给出的Android App Bundles和Dynamic Feature Modules的samples代码库。里面给出了很多实例代码，通过阅读其实现可以很好的了解其能力。

#### instantApps

instant app 是谷歌推出的一项无须用户安装应用即可使用，类似于快应用。他本身还是原生应用，只是有拥有直接在selinux沙盒中直接运行的能力。instant app是免安装的 app bundle，属于android app bundle范畴，用户在预使用instant app可以通过其内部配置的链接下载真正的APP。

instantApps下的sample较多，我们看一下比较完整的multi-feature-module工程：

这个工程由app（application工程，作为base）、detail（dynamic-feature）和main（dynamic-feature）组成，main工程显示图片grid，detail工程显示具体某张图片的详情。和我们创建的aab工程一直。确实instant app和aab可以是完全无缝的，配置区别就是在Manifest文件中instant配置设置为true：

```xml
<dist:module dist:instant="true" dist:onDemand="false"
             dist:title="@string/title_detail_feature">
    <dist:fusing dist:include="true" />
</dist:module>
```

调试是，如果我们在AS中要以instant app的模式运行，只需要更改下Run/Debug configurations，首选要运行aab，需要在Deploy中选择APK from app bundle，这个其实就是bundletool给我们做的事情，我们也可以命令行用<a href="https://github.com/google/bundletool">bundletool</a>安装aab格式的应用。然后在勾选Deploy as instant app的单选框然后运行即可，如果要用正常安装应用模式只需要不勾选。

我用小米8手机instant app模式运行后会发现，没有启动图标。在官方rom上运行应该在运行任务中instant app会有闪电标志，在设置->应用也会有单独一栏。这时如果我们把detail模块的instant改为false发现跳转详情会奔溃，即用instant app模式是只会运行instant=true的部分。

上面说了instant app module的开发和feature module完全无缝，可以使用同一个工程，由同一个aab发布。但在页面跳转上需要使用App Links，需要对Activity设置intent-filter

```xml
<activity
          android:name="com.example.android.unsplash.DetailActivity"
          android:label="@string/app_name"
          android:parentActivityName="com.example.android.unsplash.MainActivity"
          android:theme="@style/App.Detail">
    <intent-filter
                   android:autoVerify="true"
                   android:order="2">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.BROWSABLE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:host="multi-feature.instantappsample.com" />
        <data android:pathPrefix="/detail" />
        <data android:scheme="https" />
        <data android:scheme="http" />
    </intent-filter>

</activity>
```

然后跳转是通过App Links跳转：

```java
final Intent intent = new Intent(Intent.ACTION_VIEW,
    Uri.parse("https://multi-feature.instantappsample.com/detail/" + position));
intent.setPackage(context.getPackageName());
intent.addCategory(Intent.CATEGORY_BROWSABLE);
```



#### PlayCoreKtx

<a href="https://github.com/android/app-bundle-samples/tree/main/PlayCoreKtx">PlayCoreKtx</a>是playcore的kotlin扩展库，提供了kotlin携程、扩展函数等支持方便使用。对于playcore的使用可以查看<a href="https://developer.android.google.cn/guide/playcore?hl=zh_cn">Google Play Core 库概览</a>。我们知道AAB的动态下发feature module的能力是需要google play市场进行配合使用的。我们的应用中需要动态模块时，通过google play市场进行下载然后安装，playcore就是负责做这些事情。

使用playcore库的话我们只需要在gradle中导入依赖即可：

```groovy
implementation "com.google.android.play:core:${versions.playcore}"
implementation "com.google.android.play:core-ktx:${versions.playcoreKtx}"
```

然后我们回到PlayCoreKtx这个sample中。这个示例的功能是一个屏幕手电筒的效果，可以通过相机拍照选择屏幕背景色，其中相机拍照选择这个功能是一个onDemand="true"的dynamic-feature，这里不分析dynamic-feature、跨feature使用fragment、bundletool等等，关注如何通过playcore提供的API来下载安装一个dynamic-feature的。首先看触发的函数InstallViewModel.openActivityInOnDemandModule:

```kotlin
private fun openActivityInOnDemandModule(moduleName: String, fragmentName: String) {
    if (manager.installedModules.contains(moduleName)) {
        viewModelScope.launch {
            //这里的_events就是一个channel。对外的话会转成Flow，外部通过collect函数监听事件。
            //这里已经安装的话直接发送NavigationEvent事件。
            _events.send(NavigationEvent(fragmentName))
        }
    } else {
        //pictureModuleStatus是一个LiveData。
        //通过getStatusLiveDataForModule(moduleName: String)函数获取。
        //函数中通过manager.requestProgressFlow()创建Flow，然后通过map函数过滤出moduleName对于事件。
        //对内部给出的事件再包装成ModuleStatus，封装进一些业务处理逻辑。
        //requestProgressFlow这个函数不在PlayCore中，是PlayCoreKtx提供的针对Koltin优化的函数。
        //就是将内部的安装过程事件回调转成Flow给出。这中是更符合Kotlin的一直编程思想。
        val status = when (moduleName) {
            PICTURE_MODULE -> pictureModuleStatus.value
            else -> throw IllegalArgumentException("State not implemented")
        }
        //如果是NeedsConfirmation状态，说明这个dynamic-feature库是需要用户确认安装的。
        //给出InstallConfirmationEvent事件回调，通知页面显示用户确认弹窗。
        if (status is NeedsConfirmation) {
            viewModelScope.launch {
                _events.send(InstallConfirmationEvent(status.state))
            }
        } else {
            //这里请求安装，就是调用manager.requestInstall进行安装，这也是PlayCoreKtx提供扩展函数。
            //安装成功后会更新pictureModuleStatus.value。在前面说了这个Livedate。
            requestModuleInstallation(moduleName)
        }
    }
}
```

这样就一个简单的按需动态下载dynamic-feature的。在这个demo里还有介绍使用KTX库来实现应用内更新和应用内评价。

#### DynamicFeatureNavigation

DynamicFeatureNavigation演示的是使用Jetpak中的navigation库来做DynamicFeature之间的跳转。PlayCoreKtx中其实是用反射实现加载DynamicFeature中的fragment的。其实官方提供了navigation-dynamic-features-fragment库在做这件事情，同时它内部已经集成了安装按需功能模块的能力。也可以在<a href="https://developer.android.google.cn/guide/navigation/navigation-dynamic">官方文档</a>上查看。

#### DynamicCodeLoadingKotlin

DynamicCodeLoadingKotlin演示了application模块调用feature模块代码的能力。我们知道在项目的依赖结构上feature模块依赖application模块，因此feature模块可以很方便的调用application中的代码。反过来就不是那么容易了。DynamicCodeLoadingKotlin演示了三种方式分别是：反射、依赖注入和ServiceLoader（就是java中的SPI）。

