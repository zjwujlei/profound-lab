Flutter_Boost源码解析及定制
=========================

### 引擎复用能力实现

我们先看下Flutter框架本身对engine的使用逻辑。启动一个FlutterActivity的时候，可以用NewEngineIntentBuilder或者CachedEngineIntentBuilder创建Intent进行打开，在其onCreate方法中会做FlutterActivityAndFragmentDelegate的onAttach工作，其中通过setupFlutterEngine函数做engine相关创建工作。

```java
@VisibleForTesting
void setupFlutterEngine() {
    Log.d("FlutterActivityAndFragmentDelegate", "Setting up FlutterEngine.");
    String cachedEngineId = this.host.getCachedEngineId();

    //使用CachedEngineIntentBuilder创建Intent跳转时，cachedEngineId存在，尝试拿对应ID的engine
    if (cachedEngineId != null) {
        this.flutterEngine = FlutterEngineCache.getInstance().get(cachedEngineId);
        this.isFlutterEngineFromHost = true;
        if (this.flutterEngine == null) {
            throw new IllegalStateException("The requested cached FlutterEngine did not exist in the FlutterEngineCache: '" + cachedEngineId + "'");
        }
    } else {
        //不用缓存情况下，如果Activity提供了Engine直接使用
        this.flutterEngine = this.host.provideFlutterEngine(this.host.getContext());
        if (this.flutterEngine != null) {
            this.isFlutterEngineFromHost = true;
        } else {
            //否则新创建一个
            Log.d("FlutterActivityAndFragmentDelegate", "No preferred FlutterEngine was provided. Creating a new FlutterEngine for this FlutterFragment.");
            this.flutterEngine = new FlutterEngine(this.host.getContext(), this.host.getFlutterShellArgs().toArray());
            this.isFlutterEngineFromHost = false;
        }
    }
}
```
我们在比较下Flutter_Boost里对应函数的实现

```java
private void setupFlutterEngine() {
    Log.d("FlutterActivityAndFragmentDelegate", "Setting up FlutterEngine.");
    this.flutterEngine = this.host.provideFlutterEngine(this.host.getContext());
    if (this.flutterEngine != null) {
        this.isFlutterEngineFromHost = true;
    } else {
        Log.d("FlutterActivityAndFragmentDelegate", "No preferred FlutterEngine was provided. Creating a new FlutterEngine for this NewFlutterFragment.");
        this.isFlutterEngineFromHost = false;
    }
}
```

只会使用Activity提供了Engine直接使用，否则的话flutterEngine会为null。*必须在Host中提供Engine*

看Host这种是实现，通过FlutterBoost的单例类获取Engine，再跟踪下去会发现是在FlutterBoost中doInitialFlutter函数创建，该函数又根据配置不同有不同的执行时机，详见FlutterBoost.init（）函数。
```java
@Nullable
public FlutterEngine provideFlutterEngine(@NonNull Context context) {
    return FlutterBoost.instance().engineProvider();
}
```

### Android侧定制


### 路由桥接

##### Flutter的路由注册
```dart
  /// Register a map builders
  void registerPageBuilders(Map<String, PageBuilder> builders) {
    ContainerCoordinator.singleton.registerPageBuilders(builders);
  }
```

调用registerPageBuilders注入所有的页面，内部保存在一个Map里面。但是想要注入给Flutter用还需要将其设置到MaterialApp中。我们知道Flutter的路由页面由routes+onGenerateRoute+onUnknownRoute决定，然后页面的Widget都会经过builder(TransitionBuilder)做包裹。Flutter_Boost都是在这一次包裹操作中替换成真正需要显示的页面。

##### Flutter open页面
Flutter_Boost的路由桥接，解决的是混合开发模式下Flutter和原生页面的交叉跳转。通过调用FlutterBoost.singleton.open(）函数进行跳转。

```dart
  Future<Map<dynamic, dynamic>> open(
    String url, {
    Map<String, dynamic> urlParams,
    Map<String, dynamic> exts,
  }) {
    final Map<String, dynamic> properties = <String, dynamic>{};
    properties['url'] = url;
    properties['urlParams'] = urlParams;
    properties['exts'] = exts;
    return channel.invokeMethod<Map<dynamic, dynamic>>('openPage', properties);
  }
```

这里可以明确，通过MethodChannel调用原生的openPage函数。我们在Android原生侧接入Flutter_Boost初始化时会实现INativeRouter，在这个实现类里面做跳转到BoostFlutterActivity的逻辑，这里说下参数，重点是EXTRA_URL和EXTRA_PARAMS，通过设置不同的ContainerUrl跳转不同页面。



