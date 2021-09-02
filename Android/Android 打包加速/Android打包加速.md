Android 打包加速
===============



### Gradle Task 耗时分析
新版本的AGP已经提供了task耗时分析能力

./gradlew --profile --offline --rerun-tasks resguardRelease

### 使用D8 做aar优化 
打包加速的一点是业务模块aar化，正常aar对于代码只是转成java字节码，后续还是要做混淆，去糖和dx。

在dex打包工具变为D8之后，我们甚至可以将业务aar的代码直接dex化。

###### 上传时打包成dex

上传maven首先要打包aar，可以通过./gradlew :library:bundleReleaseAar -m命令快速打印整个执行过程，可以看到重点task如下。

```
:library:syncReleaseLibJars SKIPPED
:library:bundleReleaseAar SKIPPED

```

syncReleaseLibJars任务对于的是