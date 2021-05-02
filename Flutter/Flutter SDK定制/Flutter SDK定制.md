Flutter SDK定制
================

## 代码模板定制

//TODO

## Android默认armeabi-v7a支持

Flutter开发项目是，运行debug包默认是适配armeabi-v8a的，且无法通过build.gradle中NDK.abiFilters进行更改。而我们在开发项目往往只支持armeabi-v7a或者armeabi，对于很多三方库可能并为提供armeabi-v8a的SO库。

对于Flutter中的Android项目，在build.gradle中会进行如下配置

```groovy
    apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"
```

对于Flutter项目的打包过程就在这个flutter.gradle中。我们主要看其中对SO的处理。

```groovy
    /**
     * Adds the dependencies required by the Flutter project.
     * This includes:
     *    1. The embedding
     *    2. libflutter.so
     */
    void addFlutterDependencies(buildType) {
        String flutterBuildMode = buildModeFor(buildType)
        if (!supportsBuildMode(flutterBuildMode)) {
            return
        }
        String repository = useLocalEngine()
            ? project.property('local-engine-repo')
            : MAVEN_REPO

        project.rootProject.allprojects {
            repositories {
                maven {
                    url repository
                }
            }
        }
        // Add the embedding dependency.
        addApiDependencies(project, buildType.name,
                "io.flutter:flutter_embedding_$flutterBuildMode:$engineVersion")

        List<String> platforms = getTargetPlatforms().collect()
        // Debug mode includes x86 and x64, which are commonly used in emulators.
        if (flutterBuildMode == "debug" && !useLocalEngine()) {
            platforms.add("android-x86")
            platforms.add("android-x64")
        }
        platforms.each { platform ->
            String arch = PLATFORM_ARCH_MAP[platform].replace("-", "_")
            // Add the `libflutter.so` dependency.
            addApiDependencies(project, buildType.name,
                    "io.flutter:${arch}_$flutterBuildMode:$engineVersion")
        }
    }
```

addFlutterDependencies函数会根据platform不同，依赖不同的maven库，根据注释可以看到这个库里主要包括embedding和libflutter.so。而我们的问题就是这里添加依赖的是armeabi-v8a的库。

```groovy
    private List<String> getTargetPlatforms() {
        if (!project.hasProperty('target-platform')) {
            return DEFAULT_PLATFORMS
        }
        return project.property('target-platform').split(',').collect {
            if (!PLATFORM_ARCH_MAP[it]) {
                throw new GradleException("Invalid platform: $it.")
            }
            return it
        }
    }
```

我们再看getTargetPlatforms()函数，我们可以试着输出*project.property('target-platform')*，在AS中运行是android-arm64。但我们项目里并没有设置target-platform，那这个是从哪来的呢？
那我们得从Flutter打包入手，Flutter所有的命令都在"$flutterRoot/packages/flutter_tools/lib/“下，我们从executabe.dart开始寻找，executabe.dart->src/commands/build.dart->src/commands/build_apk.dart->src/android/android_builder.dart->src/android/gradle.dart。可以看到是在AS 执行Run时确定的，这下就涉及到AS插件就不好处理了。我们可以换一个思路直接从getTargetPlatforms()的结果入手，在这个值中读取build.gradle中NDK.abiFilters，切理论上这两个也应该保持一致。


```groovy
    private List<String> getTargetPlatforms() {
        //新增ndk.abiFilters判断
        if(project.android.defaultConfig.ndk.abiFilters != null){
            return project.android.defaultConfig.ndk.abiFilters.collect {
                if(it == ARCH_ARM32){
                    return PLATFORM_ARM32
                }else if(it == ARCH_ARM64){
                    return PLATFORM_ARM64
                }else if(it == ARCH_X86){
                    return PLATFORM_X86
                }else if(it == ARCH_X86_64){
                    return PLATFORM_X86_64
                }else{
                    throw new GradleException("Invalid platform: $it.")
                }
            }
        }

        ...
    }
```

当特别要注意的是build.gradle中的配置，build.gradle中对flutter.gradle的引入必须在android.defaultConfig.ndk.abiFilters声明之后，否则会拿不到abiFilters的值。

### 产物自动上传maven配置

我们通过”flutter build aar target-platform=armeabi“命令打包完flutter产物后，后保存到Flutter工程的”/build/host/outputs“目录下,我们查看产物可以发现其完全符合一个maven依赖库的规范，有groupId、有version、archives，也有maven的pom.xml。基于此我们判断打包完flutter产物是有maven发布的过程，但是发布在了本地仓库中你那个。

查询Flutter源码中build aar的实现过程，这里先介绍下源码查看的过程，先看flutter命令工具，用vim或者其他编辑器打开。可以看到执行的入口是 SNAPSHOT_PATH="$FLUTTER_TOOLS_DIR/bin/flutter_tools.snapshot"，这个是在Flutter安装第一次使用、升级等情况下在Building flutter tool过程中通过 SCRIPT_PATH="$FLUTTER_TOOLS_DIR/bin/flutter_tools.dart" 生成的，所以源码里我们从该文件入手查看。

查看flutter_tools.dart就只封装了executable.dart的调用，我们是build，所以看BuildCommand->BuildAarCommand->androidBuilder.buildAar->_AndroidBuilderImpl.buildAar->buildGradleAar(gradle.dart)->aar_init_script.gradle

我们要定制的就在aar_init_script.gradle的configureProject中，我们改成根据配置发布不同仓库：
```groovy
void configureProject(Project project, String outputDir) {
    ...
    //新增上传maven逻辑，根据rootProject的配置确定上传maven还是local。
    // project.version = project.version.replace("-SNAPSHOT", "")
    project.version = (project.rootProject.hasProperty("pushMaven")&&project.rootProject.pushMaven)
        ?project.rootProject.versionId:project.version.replace("-SNAPSHOT", "")
    
    project.uploadArchives {
        repositories {
        
            mavenDeployer {
                if(project.rootProject.hasProperty("pushMaven")&&project.rootProject.pushMaven){
                    repository(url: project.rootProject.mavenUrl){
                        authentication(userName: project.rootProject.userName, password: project.rootProject.password)
                    }
                }else{
                    repository(url: "file://${outputDir}/outputs/repo") 
                }
            } 
        }
    }
    ...
}
```

这时我们打包，会发现很诡异的事情，输出打包AAR成功并且在maven上有正确的产物，当时命令执行失败了，报错如下：
```shell
Gradle task assembleAarRelease failed to produce LocalDirectory: '/Users/xxx/build/host/outputs/repo'.
```
我们将产物上传到了maven，但这里还会去校验本地产物是否存在来判断是否执行成功，我们回到gradle.dart的buildGradleAar()函数中发现有如下校验，我们将其注释掉：
```dart
Future<void> buildGradleAar({
  @required FlutterProject project,
  @required AndroidBuildInfo androidBuildInfo,
  @required String target,
  @required Directory outputDirectory,
}) async {
  
...

// 有推送maven的情况，这里不做本地repo产物校验  
//  final Directory repoDirectory = getRepoDirectory(outputDirectory);
//  if (!repoDirectory.existsSync()) {
//    printStatus(result.stdout, wrap: false);
//    printError(result.stderr, wrap: false);
//    throwToolExit(
//      'Gradle task $aarTask failed to produce $repoDirectory.',
//      exitCode: exitCode,
//    );
//  }
//  printStatus(
//    '$successMark Built ${fs.path.relative(repoDirectory.path)}.',
//    color: TerminalColor.green,
//  );
}
```

这个时候我们重新执行，当发现并未生效。回到开头的flutter命令工具源码中，我们了解到执行的其实是打包好的flutter_tools.snapshot文件，我们将其删除flutter_tools.snapshot和flutter_tools.stamp删除重新执行，整个过程就完成了。
