Qigsaw打包流程分析及定制
=====================

###  qigsaw相关task分析

* 宿主 ComponentInfoTransform：宿主中添加的自定义Transform API，生成ComponentInfo类，内部记录了个插件的四大组件信息，通过解析插件的Manifest文件生成。
* 宿主 QigsawProcessOldApkTask：传入OldApk的时候，会将OldApk中的插件信息取出，做Tinker热修复是会用到。
* 宿主 GenerateQigsawConfig：生成QigsawConfig.java类。依赖QigsawProcessOldApkTask，如果有OldApk的使用OldApk的qigsawId，否则按照"${versionName}_${gitRev}"的规则生成。
* 宿主 QigsawAssembleTask：生成并拷贝qigsaw_${SPLIT_INFO_VERSION}.json的配置文件，如果是动态下发的插件需要上传CND，如果是buildIn插件则拷贝到assets目录。
* 宿主 QigsawInstallTask：直接安装APK，会查询所有连接的Android依次安装。
* 宿主 QigsawUploadSplitApkTask：未开启上传功能时，所有的splitApk都保存在宿主中，    通过该Task可以上传oldApk的splitApk到CDN，并更新oldApk的qigsaw配置信息。
* splitApk SplitResourcesLoaderTransform：从manifest文件中解析出四大组件，对四大组件的getResources函数做字节码注入，添加*SplitInstallHelper.loadResources(this, super.getResources());*的调用。
* splitApk CopySplitApkTask：拷贝splitApk的apk文件到宿主工程”${project.buildDir}intermediates/qigsaw/split_apk“ 目录下。
* splitApk CopySplitManifestTask：拷贝splitApk的Manifest文件到宿主工程”${project.buildDir}intermediates/qigsaw/split_manifest 目录下。


### 自定义流程

##### 打包所有插件并上传

宿主Task，基本类似原QigsawAssembleTask，生成qigsaw_${SPLIT_INFO_VERSION}.json的配置文件并拷贝到”${project.buildDir}intermediates/qigsaw/split_apk“目录下，该文件可上传git。区别在于后续宿主打包流程可不用继续。

