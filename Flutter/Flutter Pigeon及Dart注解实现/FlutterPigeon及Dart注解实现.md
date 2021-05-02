Flutter Pigeon实现
==========================

### 命令行

```
flutter pub run pigeon --input pigeons/pigeonDemoMessage.dart
```

正常dart编写命令行app入口都是main函数，当我们在pigeon类内部并没有找到main函数。这里我们就需要先了解下flutter命令运行后的流程了，入口函数是run函数。

```

```


### ast.dart

AST(abstract syntax trees),抽象预发树。在前端上运用的很多，包括Webpack都是基于AST实现（使用recast库），前端可以通过recast来创建、修改JS代码，类似于javapoet等。


这里定义的ast.dart，是定义我们用来生成代码的接口。可以通过这里的定义来对Pigeon有一个整体的了解。

Pigeon是用来生成Flutter和原生交互channel实现的，因此主要是需要提供方法调用和数据传递。

```
/// Represents a collection of [Method]s that are hosted ona given [location].
class Api extends Node {
  /// Parametric constructor for [Api].
  Api({this.name, this.location, this.methods, this.dartHostTestHandler});

  /// The name of the API.
  String name;

  /// Where the API's implementation is located, host or Flutter.
  ApiLocation location;

  /// List of methods inside the API.
  List<Method> methods;

  /// The name of the Dart test interface to generate to help with testing.
  String dartHostTestHandler;
}
```

Api类定义的channel提供的功能列表，里面主要包含了Method。

```
/// Represents a class with [Field]s.
class Class extends Node {
  /// Parametric constructor for [Class].
  Class({this.name, this.fields});

  /// The name of the class.
  String name;

  /// All the fields contained in the class.
  List<Field> fields;

  @override
  String toString() {
    return '(Class name:$name fields:$fields)';
  }
}
```

Class类定义的channel传递的数据结构。只有属性没有函数。

```
enum ApiLocation{
  host,
  flutter,
}

```

特别的对于ApiLocation枚举，其中host是指原生提供给Flutter的能力，而flutter是指flutter提供给原生的能力。

### generator_tools.dart

用来输出代码。定义了Indent类，做文本输出。Indent类并不针对Android、iOS或者Dart，更多的是文本输出。不像javapoet等工具是针对java预发封装的，因此在使用的时候需要注意对应的语法。

generator_tools.dart的实现与pigeon无关，甚至也和Flutter无关，任何代码生成都可使用。

### dart_generator.dart
s
用来生成dart侧代码。主要是数据结构定义类（Class）和功能（API），其中功能又分为host功能和Flutter功能。所以dart_generator.dart中：

* 定义了_writeHostApi函数来生成原生功能实现（ApiLocation.host）。
* 定义了_writeFlutterApi来生成Flutter功能实现（ApiLocation.flutter）。
* 定义了generateDart函数来生成代码。
    - 1.生成类注释。
    - 2.生成导包。
    - 3.生成数据结构定义类（Class）
    - 4.调用_writeHostApi和_writeFlutterApi生成对应功能。

特别的对于原生提供的功能，提供了mock实现，可以生成Flutter对应函数来做mock。

### java_generator.dart
用来生成java侧代码。整体功能和dart_generator.dart一直。特别的是类型转化：_javaTypeForDartTypeMap。

### objc_generator.dart

同java_generator.dart+dart_generator.dart

### pigeon_lib.dart

###### dart:mirrors 库
这个是dart的反射库，和java的反射类似。同样的dart:mirrors已支持获取注解，在Dart里面叫做Metadata。但和Java不通，Metadata没有明确的运行期、编译期、源码期的区分，但我们可以实现在打包期间进行注解处理。对于dart:mirrors库我们另行讨论。


###### pigeon类
pigeon类的run函数是整个功能的入口。调用dart:mirrors相关知识进行解析出所有进行了HostApi、FlutterApi、async的注解，收集信息，调用generator生成代码。

对于HostApi、FlutterApi注解的定义，直接进行类定义即可

```
class HostApi {

  const HostApi({this.dartHostTestHandler});

  final String dartHostTestHandler;
}


class FlutterApi {

  const FlutterApi();
}

```

run函数的流程如下：

* 解析命令行转入参数，使用'package:args/args.dart'库。
* 如果配置文件（input指定的dart文件)如果定义了configurePigeon函数，调用设置options。
* 如果配置有误输出usage
* 运营mirrors的知识查询currentMirrorSystem下的所有Mirror找到进行了HostApi、FlutterApi注解的Mirror。
* 调用parse函数进行解析所有的API，创建出Root节点。
* 调用generator相关实现输出目标文件。

###### bin/pigeon.dart

如果我们需要使用"flutter pub run pigeon xxx"的命令来执行相关功能，需要在项目的bin目录下创建一个pigeon.dart文件，在其中提供入口main函数。

这里用到了dart的一些异步知识。我们可以查看'dart:isolate'库的相关资料。

在bin/pigeon.dart中，我们要做的在temp目录下动态创建了一个'_pigeon_temp_.dart'。然后使用Isolate.spawnUri函数来创建一个isolate来异步执行，然后我们在本isolate中通过ReceivePort来监听执行情况。

### 后记
pigeon的实现是一个完整的通过注解来动态生成代码实现。在Android开发中我们有很多功能都是通过注解+自定义注解处理器+javapoet等动态代码生成来进行的。我们使用pigeon的这一套实现完全可以进行Flutter版本的提效库实现。


### 资料

<a href="https://github.com/flutter/packages/tree/master/packages/pigeon">pigeon源码</a>
