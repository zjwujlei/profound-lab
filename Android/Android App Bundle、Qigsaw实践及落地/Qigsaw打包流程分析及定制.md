Qigsaw打包流程分析及定制
=====================

###  qigsaw宿主相关task
宿主相关task都在QigsawAppBasePlugin中

###### 宿主 qigsawProcessReleaseManifest

对应类：QigsawProcessManifestTask
Task依赖关系：mustRunAfter processManifestTask

做所有splitApk的Manifest文件的合并。task要求的Input包括splitManifestOutputDir目录和mergedManifestFile文件。

splitManifestOutputDir目录是所有的splitApk打包时Manifest文件输出目录。mergedManifestFile文件是android原processManifestTask任务的输出目录，processManifestTask任务做了所有宿主依赖模块的Manifest文件合并。

这个Task做的事情就是插件和宿主的Manifest文件的合并。从splitManifestOutputDir目录中读取插件的Manifest文件合并到mergedManifestFile文件中。

###### 宿主 qigsawProcessReleaseOldApk

对应类：QigsawProcessOldApkTask

在配置了OldApk时有效，这个Task做的事情其实就是解压OldApk，将插件相关信息释放到oldApkOutputDir中。

这些信息会在GenerateQigsawConfig和QigsawAssembleTask两个Task中用到，分别是配置qigsawId和比对使用OldApk的插件，会在对应插件中具体讲到。

###### 宿主 generateReleaseQigsawConfig

对应类：GenerateQigsawConfig

收集信息生成QigsawConfig.java类，记录了插件信息

```
public final class QigsawConfig {
  public static final boolean QIGSAW_MODE = false;
  public static final String QIGSAW_ID = "4.3.2_c3f922e";
  public static final String VERSION_NAME = "4.3.2";
  public static final String DEFAULT_SPLIT_INFO_VERSION = "4.3.2_1.0.0";
  public static final String[] DYNAMIC_FEATURES = {"moduleMiPush","moduleVivoPush"};
}

```

当没在OldApk时，会在assets下的json配置文件：qigsaw_${appVersion}_${splitInfoVersion}.json中读取QIGSAW_ID，否则会根据版本+git信息："${versionName}_${gitRev}"的方式生成。

###### 宿主 qigsawProcessReleaseProguard
对应类：QigsawProguardConfigTask

Task依赖相关：

```
//混淆task需要依赖generateReleaseQigsawConfig
if (proguardTask != null) {
    proguardTask.dependsOn proguardConfigTask
} else {
    if (r8Task != null) {
        r8Task.dependsOn proguardConfigTask
    }
}
//generateReleaseQigsawConfig需要在qigsawProcessManifestTask之后
proguardConfigTask.mustRunAfter qigsawProcessManifestTask

//将QigsawProguardConfigTask生成的混淆文件qigsaw_proguard.pro设置到该构建变体中。
variant.getBuildType().buildType.proguardFiles(proguardConfigTask.getOutputProguardFile())
```

处理混淆相关，在开启混淆时有效。将Qigsaw框架相关的混淆规则（QigsawProguardConfigTask.PROGUARD_CONFIG_SETTINGS常量记录的值）写入qigsaw_proguard.pro中，并作为混淆规则文件之一。

######宿主 qigsawAssembleRelease
对应类：QigsawAssembleTask

主要做的是生成JSON配置文件和拷贝插件文件到指定目录。

首先针对每一个插件收集配置信息，这里处理版本等常规信息外，还有插件间的依赖，这些依赖信息由插件工程打包的时候生成并保存在splitDependenciesOutputDir目录下，具体生成任务会在插件Task中讲到。主要用到SplitInfoProcessor类，生成所有插件的信息SplitInfo。这类需要插件工程参与，会通过遍历dfProjects（所有dynamic feature工程），搜集其打包产物（manifest文件、splitApk、json配置文件）中的信息生成。

对于本地插件，可能拷贝到assets目录下，也可能会释放到libs目录下。首先确定mergeJniLibInternalDir目录，这里根据AGP版本不同有所适配
```
if (versionAGP >= VersionNumber.parse("3.5.0")) {
    mergeJniLibInternalDir = new File(mergeJniLibDir, "lib")
} else {
    File contentJsonFile = new File(mergeJniLibDir, CONTENT_JSON_FILE_NAME)
    if (contentJsonFile.exists()) {
        List contents = TypeClassFileParser.parseFile(contentJsonFile, List.class)
        mergeJniLibInternalDir = new File(mergeJniLibDir, "${(int) (contents.get(0).index)}/lib")
        QigsawLogger.e("> Task :${name} mergeJniLibs content_json :" + contents.toString())
    }
}
```

对于3.5.0以下，这里附上CONTENT_JSON_FILE_NAME文件内容：
```
[{"name":"resources","index":0,"scopes":["PROJECT","SUB_PROJECTS","EXTERNAL_LIBRARIES"],"types":["NATIVE_LIBS"],"format":"DIRECTORY","present":true}]
```
针对上面的配置取到的是目录“0”。

然后会根据宿主的ndk.abiFilters和mergeJniLibInternalDir目录存在的abiDirNames确定是保存目录。
```
if (abiFilters.empty) {
    if (abiDirNames.size() == 1) {
        copyToAssets = false
    }
} else {
    if (!abiDirNames.empty) {
        if (abiFilters.size() == 1 && abiDirNames.containsAll(abiFilters)) {
            copyToAssets = false
        }
    }
}
```

然后调用SplitDetailsProcessor类，做整体信息收集以及SplitApk预处理。处理QigsawId等整体信息外，会确定SplitApk文件的url，如果是远程插件会上传到CDN。如果配置了OldApk，对于没有版本变更的插件会直接使用OldApk内原插件，包括QigsawId。

最会，根据收集好的SplitDetails生成"qigsaw_${splitInfoVersion}${SdkConstants.DOT_JSON}"的配置文件，以及拷贝本地插件文件。

###### 宿主 transformClassesWithCreateComponentInfoForRelease

看Task名称我们就知道是Transform API，对应类：ComponentInfoTransform。

主要是用来做Android四大组件和Application的代理。我们知道AndroidManifest文件是在安装的时候注册的，我们无法更新，因此必须要讲插件的AndroidManifest文件合并到宿主中。这里我们想一下这类场景，如果我们插件还没安装，此时收到一个插件里的广播会发生什么情况，是不是会奔溃？

是的，ComponentInfoTransform就是来解决这个问题，我们对四大组件和Application生成一个空的类保存到宿主中。这样宿主在未安装插件的时候就不会奔溃。特别是Provider，对于FileProvider，在启动的时候就会进行使用过，有FileProvider时不这样做就会启动奔溃。

这里又有一个问题，根据类加载机制，如果已经加载过了同名空的类，安装插件后就无法加载插件中真正的类了。这个问题会在框架部分介绍。

###### 宿主 qigsawInstallRelease

对应类：QigsawInstallTask

通过ADB命令对打包完成的宿主APK进行安装。

###### 宿主 qigsawUploadSplit

对应类：QigsawUploadSplitApkTask

未开启上传功能时，所有的splitApk都保存在宿主中，对于远端插件会调用该task上传CDN，这里上传的实现需要我们自己实现并设置到SplitApkUploaderInstance中。

###  qigsaw插件相关task

插件相关task逻辑，一部分在QigsawAppBasePlugin中，一部分在QigsawDynamicFeaturePlugin中。

在QigsawAppBasePlugin中部分，并不是说将Task定义在宿主的project中，只是因为这部分Task主要做的是插件打包产物向宿主输出，既宿主的打包流程依赖这部分Task，本身还是添加在插件project下的。
```
dfProjects.each { Project dfProject ->
    try {
        configQigsawAssembleTaskDependencies(dfProject, variantName, mergeJniLibsTask, dfClassPaths,
                splitApkOutputDir, splitManifestOutputDir, splitDependenciesOutputDir)
        QigsawLogger.w("dynamic feature project ${dfProject.name} has been evaluated!")
    } catch (Throwable ignored) {
        dfProject.afterEvaluate {
            configQigsawAssembleTaskDependencies(dfProject, variantName, mergeJniLibsTask, dfClassPaths,
                    splitApkOutputDir, splitManifestOutputDir, splitDependenciesOutputDir)
            QigsawLogger.w("dynamic feature project ${dfProject.name} has not been evaluated!")
        }
    }
}
```

特别的，对这一部分有如上逻辑。我们知道gradle存在初始化、配置、执行三个阶段，afterEvaluate就是配置阶段完成的回调。这里存在一种情况，宿主project已经完成了配置，但插件还没，此时直接去获取插件的task做逻辑就会出异常，因此这里的处理是先直接调用configQigsawAssembleTaskDependencies（）函数，如果发生异常说明插件还没完成配置，则通过dfProject.afterEvaluate {}在插件完成配置后在调用configQigsawAssembleTaskDependencies（）函数。

###### 插件 copySplitApkRelease
对应类：CopySplitApkTask

将插件工程的输出文件splitApk拷贝到宿主工作目录（build/ntermediates/qigsaw/split_apk/）下，宿主的qigsawAssembleRelease任务会用到这些文件。

这个任务单纯做文件拷贝并没有其他逻辑。确实在Qigsaw新版本的打包流程中已经去除了这个Task，不做拷贝，使用时直接用插件工程下的文件。

###### 插件 copySplitManifestRelease
对应类：CopySplitManifestTask

将插件工程的输出文件Manifest拷贝到宿主工作目录（build/ntermediates/qigsaw/split_manifest/）下，宿主的qigsawAssembleRelease和qigsawProcessReleaseManifest任务会用到这些文件。

这个任务单纯做文件拷贝并没有其他逻辑。确实在Qigsaw新版本的打包流程中已经去除了这个Task，不做拷贝，使用时直接用插件工程下的文件。

###### 插件 analyzeSplitDependenciesRelease
对应类：AnalyzeSplitDependenciesTask

分析插件的dependencies依赖信息，找出插件间是否有依赖。首先对所有的插件工程，通过"${dfProject.group}:${dfProject.name}:${dfProject.version}"获取其GAV坐标，然后针对当前插件通过：
```
Configuration configuration = dfProject.configurations."${variantName.uncapitalize()}CompileClasspath"
configuration.incoming.dependencies.each {
    splitDependencies.add("${it.group}:${it.name}:${it.version}")
}
```
获取所有依赖进行对比，如果存在，保存依赖的artifactId到json文件中，并输出到宿主工作目录（build/ntermediates/qigsaw/split_dependencies/）下。

对应的是在宿主的qigsawAssembleRelease中使用，会将依赖信息输出到SplitInfo中，插件加载的时候会校验使用，这里在插件运行框架里会有用到。同样的在新版本Qigsaw中也去掉了这个task。

###### 插件 transformClassesWithSplitResourcesLoaderForRelease
在QigsawDynamicFeaturePlugin中定义，使用的Transform API，对应的类：SplitResourcesLoaderTransform

主要做的事情是在activity、service和receiver中注入代码，保证插件的资源被加载。对于Activity，在getResources()函数中调用SplitInstallHelper的loadResources(Activity activity, Resources resources)函数，来触发加载所有插件的资源。如果本身没有重写getResources()也还会进行添加。

对于service是在onCreate()函数中添加调用 SplitInstallHelper的loadResources(Service service）函数。对于receiver是在onReceive()函数中添加调用 SplitInstallHelper的loadResources(BroadcastReceiver receiver, Context context）函数。

###### 插件 transformClassesWithSplitLibraryLoaderForRelease
在QigsawDynamicFeaturePlugin中定义，使用的Transform API，对应的类：SplitLibraryLoaderTransform

主要做的事情是定义一个"com.iqiyi.android.qigsaw.core.splitlib." + project.name + "SplitLibraryLoader"类提供一个loadSplitLibrary（String）函数，内部调用System.loadLibrary函数来加载插件的SO。这个函数在框架的SplitLibraryLoaderHelper类中使用，用来加载插件的so。

这里是应为我们可能采用MULTIPLE_CLASSLOADER模式，每个插件一个ClassLoader，此时对于SO的加载路径也是隔离的，在宿主活其他插件的ClassLoader下无法获取该插件的SO，必须通过该插件内的类在该插件的ClassLoader中进行调用才能获取。
### 自定义流程

##### 打包所有插件并上传

宿主Task，基本类似原QigsawAssembleTask，生成qigsaw_${SPLIT_INFO_VERSION}.json的配置文件并拷贝到”${project.buildDir}intermediates/qigsaw/split_apk“目录下，该文件可上传git。区别在于后续宿主打包流程可不用继续。

