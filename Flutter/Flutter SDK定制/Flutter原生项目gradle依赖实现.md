Flutter原生项目gradle依赖实现
==========================

### pubspec.yaml

pubspec.yaml文件定义了项目类型，Flutter、Dart等版本，依赖库和资源，这里需要关注的是项目类型：

对于Flutter plugin
```yaml
flutter:
  plugin:
    androidPackage: ${packageName}
    pluginClass: WrapperMapPlugin
```
对于plugin项目，会通过plugin指明，并提供androidPackage和pluginClass。

对于Flutter module

```yaml
flutter:
  module:
    androidX: true
    androidPackage: ${packageName}
    iosBundleIdentifier: ${bundleId}
```
对于modulemodule项目，会通过modulemodule指明，并提供androidPackage等信息。

Flutter app和Flutter package并没有相关指定。

### .flutter-plugins 和.flutter-plugins-dependencies

每个flutter在rootDir下都后生成这两个文件，查看flutter.gradle中对configurePlugins函数的注释可以简单了解这两个文件的生成和作用。

```groovy
/**
 * Configures the Flutter plugin dependencies.
 *
 * The plugins are added to pubspec.yaml. Then, upon running `flutter pub get`,
 * the tool generates a `.flutter-plugins` file, which contains a 1:1 map to each plugin location.
 * Finally, the project's `settings.gradle` loads each plugin's android directory as a subproject.
 */
```

### Flutter app下的settings.gradle

Android项目的settings.gradle注册了所有以源工程的形式集成的project。通过include ":${name}:，通过project(":${name}").dir = ${file}来指定工程的路径。

对于Flutter项目就是读取.flutter-plugins文件，将文件中指定依赖的Flutter plugin项目的android工程进行添加。

```groovy
    def pluginDirectory = flutterProjectRoot.resolve(path).resolve('android').toFile()
    include ":$name"
    project(":$name").projectDir = pluginDirectory
```

这样的涉及导致了一个问题，

### flutter plugin的Androd工程

和我们正常创建的工程相比，是rootProject即为自身的module。settings.gradle中没有额外include其他module，rootProject的build.gradle和module的build.gradle做了合并。


