预装APP SO问题分析
==================

### 问题说明
App作为预装应用时，启动会由于找不到SO库导致奔溃。


### 问题分析
预装应用加载SO的路径如下：nativeLibraryDirectories=[/system/app/${app}/lib/arm, /system/app/${app}/${app}.apk!/lib/armeabi-v7a, /system/lib, /system/vendor/lib, /system/lib, /system/vendor/lib]。

系统应用在安装时，PackageManaer并不会自动抽离释放APK内部的so文件。系统应用的SO是在ROM包打包的时候进行释放到同级目录下的lib目录下（对应so加载路径”/system/app/${app}/lib/arm“）。这步释放需要在ROM包的packages/apps/${app}下创建lib目录放置so文件并在Android.mk文件中进行配置。


### 解决方案



方案二：ROM包制作过程中入手，根据ROM包的buid规范实现SO自动释放，但不确定能否解决。

方案三：直接将我们的SO拷贝到/system/lib下，该方案可以解决，当需要和厂商进行协调，且会被作为公共SO。