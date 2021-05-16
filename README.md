

Profound Android开发学习笔记
===============

<div style="align: center">
<img src="./images/study.png"/>
</div>

### 介绍 - 纯手打，勿喷。

11年底开始做Android，掐指一算也十年了。期间技术一直迭代，从热修复、热更、插件化，从H5 hybrid、phoneGap到weex、RN和Flutter，从dex动态加载、不落地加载到指令抽离、VPM加固方案。甚至开发工具从eclipse到AS，开发语言从Java到Kotlin，编译工具从Ant到Gradle。还有每年雷打不动的Android新版本发布的适配，一年比一年严酷的行业竞争的倒逼，学习之心一刻不敢停。

从16年开始，养成了使用markdown进行记录的习惯，有学习笔记、有技术方案选型，也有公司项目方案推进。近期参与周边朋友的组团学习计划，将自己的学习内容进行整理回顾，挑选合适部分在github上进行公开记录，部分笔记有对应的代码库会同步上传。万一哪天去开包子铺了呢。

部分笔记刚起了个头，后续看看有时间可以可以回顾补全。


### Android-JNI-Linux
- [Android JNI使用三方库入门](./Android-JNI-Linux/Android JNI使用三方库入门/Android JNI使用三方库入门.md)
- [Android高效数据存储-mmap](./Android-JNI-Linux/Android高效文件存储实现-mmap/Android高效文件存储实现-mmap.md)
- [共享内存Ashmen](./Android-JNI-Linux/共享内存Ashmem/Android共享内存实践.md)
- [JNI开发内存释放学习笔记](./Android-JNI-Linux/共享内存Ashmem/JNI开发内存释放学习记录 .md)

### Android 相关学习笔记
- [主流插件化框架横向对比](./Android/Android App Bundle、Qigsaw实践及落地/主流插件化框架横向对比.md)
- [Qigsaw打包流程分析及定制](./Android/Android App Bundle、Qigsaw实践及落地/Qigsaw打包流程分析及落地.md)
- [Android App Bundle](./Android/Android App Bundle、Qigsaw实践及落地/Android App Bundle实践及落地.md)
- [Android Native Hook技术研究](./Android/Android Native Hook技术研究/Native Hook技术.md)
- [Android Pipline环境搭建](./Android/Android Pipline环境搭建/Android Pipline代码扫描搭建.md)
- [AndroidAPK分析及逆向加固 - 资源篇](./Android/AndroidAPK分析及逆向、加固/arsc文件分析及逆向、加固.md)
- [AndroidAPK分析及逆向加固 - so篇](./Android/AndroidAPK分析及逆向、加固/SO文件分析及逆向、加固.md)
- [Android代码编译过程理解](./Android/Android代码编译过程理解/Android代码编译过程理解.md)
- [Android应用启动流程分析](./Android/Android应用启动流程分析/Android应用启动流程分析.md)
- [ScriptEngine-Android动态逻辑JS下发实现](./Android/JSEngine/ScriptEngine-Android动态逻辑JS下发实现.md)
- [RXAndroid学习思考](./Android/RXAndroid/RXAndroid学习思考.md)
- [SO加载路径分析](./Android/SO加载路径分析/so库加载路径分析.md)
- [Tinker源码研究分析](./Android/Tinker源码研究分析/Tinker源码研究分析.md)
- [响应式编程框架AndX](./Android/响应式框架AndX/响应式编程方案AndX.md)
- [基于V2签名的渠道包生成方案](./Android/基于V2签名方案的渠道包生成方案/基于V2签名方案的渠道包生成方案.md)
- [权限申请分析](./Android/权限申请分析/权限适配说明.md)
- [webview资源缓存方案分析](./Android/opt-webview/optwebview.md)
- [Android架构组件Jetpack](./Android/Android架构组件Jetpack使用记录/Android架构组件Jetpack.md)

### Flutter 相关学习笔记
- [dart:mirrors库及注解Metadata](./Flutter/Dart进阶之路/dart反射mirrors库及注解Metadata.md)
- [Flutter Android热更新方案设计](./Flutter/Flutter Android热更实现/Flutter Android热更新方案设计.md)
- [Flutter Pigeon实现](./Flutter/Flutter Pigeon及Dart注解实现/FlutterPigeon及Dart注解实现.md)
- [Flutter SDK定制](./Flutter/Flutter SDK定制/Flutter SDK定制.md)
- [Flutter原生项目gradle依赖实现](./Flutter/Flutter SDK定制/Flutter原生项目gradle依赖实现.md)
- [Flutter_Boost源码解析及定制](./Flutter/Flutter_Boost源码解析及定制/Flutter_Boost源码解析及定制.md)
- [FlutterSDK源码分析](./Flutter/FlutterSDK源码分析/Flutter 源码分析.md)
- [Flutter Command-line分析](./Flutter/FlutterSDK源码分析/Flutter Command-line分析.md)
- [Flutter下Tangram实现动态化](./Flutter/Tangram-Flutter/Tangram-Flutter.md)
- [Flutter统一插件模式](./Flutter/Flutter统一插件模式/Flutter插件工程优化.md)

### RN 相关学习笔记
- [RN 安卓侧源码阅读记要](./react-native/RN Android侧源码阅读/RN 安卓侧源码阅读记要.md)
- [RN开发场景中的定制优化](./react-native/RN开发场景中的定制优化/RN开发场景中的定制优化.md)
- [React Native的运行、打包、热更新分析](./react-native/RN运行、打包、热更新分析/RN运行、打包、热更新分析.md)
- [RN源码编译及修改](./react-native/RN源码编译及修改/RN源码编译及修改.md)

### Kotlin、Java相关
- [Java线程池使用介绍](./kotlin、Java/Java线程池介绍/Java线程池使用介绍.md)
- [类加载机制](./kotlin、Java/JVM虚拟机学习笔记/类加载机制.md)
- [使用KTNative开发核心逻辑可行性](./kotlin、Java/kotlinNative/KTNative落地分析.md)
- [kotlin、协程语法及实现](./kotlin、Java/kotlin、协程语法及实现/kotlin、协程语法及实现.md)

### 前端相关
- [vue入门学习笔记](./前端/vue学习笔记/vue入门学习笔记.md)
- [weex学习入门](./前端/Weex入门产出/Weex学习入门.md)
- [weex Android侧实现分析](./前端/Weex入门产出/Weex Android侧实现分析.md)




