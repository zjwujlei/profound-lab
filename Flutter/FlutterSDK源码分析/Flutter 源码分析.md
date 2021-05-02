Flutter 源码分析
==============

### 简介
基于Flutter 1.12.13-hotfix.9的源码进行阅读，由于我们的项目里使用的是Flutter_Boost框架，因此在分析中会穿插Flutter_Boost魔改的内容，对后续重写自己的Flutter_Boost类框架做准备。


### FlutterApplication
Flutter框架提供了FlutterApplication类，里面实现相当简单，我们的Application完全可以不用继承FlutterApplication，重点就是在onCreate函数中：
```java
    FlutterMain.startInitialization(this);
```
做初始化，内部关键节点调用FlutterLoader.startInitialization函数，主要做了以下几件事情：

1.  从Manifest中读取Meta 元数据，读取AOT共享库名字（libapp.so）、Flutter Assets的目录（flutter_assets）、虚拟机快照数据（vm_snapshot_data）、隔离快照数据（isolate_snapshot_data）。这里可以看到我们如果要修改AOT共享库名字，只要在meta元数据中提供。
2. 对应用沙盒cache目录下的以“.org.chromium.Chromium.”开头的文件进行清理，原因暂未知。
3. 加载libflutter.so库。
4. 桥接原生的帧同步能力，（Choreographer相关，我们做FPS监控用到过这个）。

我们重点可以关注第一点，我们在开发的时候做些定制会用到。


对于Flutter_Boost，我



### FlutterActivity

在FlutterSDK内部存在两个FlutterActivity实现，io.flutter.embedding.android.FlutterActivity和io.flutter.app.FlutterActivity。我们创建一个Flutter项目的时候默认使用的是io.flutter.embedding.android.FlutterActivity，我们主要看这个FlutterActivity。io.flutter.embedding.android.FlutterActivity我们会在最后简单了解。

FlutterActivity的实现相当简单，基本围绕这FlutterEngine

### FlutterActivityAndFragmentDelegate


### FlutterEngine


### FlutterView



### FlutterPluginRegistry

### Flutter dart核心类实现
##### MaterialApp
MaterialApp是一个符合Material design的application，注意是application。MaterialApp也是一个StatefulWidget，但和我们写的页面不同，他是一个顶层容器，类似与Android的顶层容纳mSubDecor（本身也是个ViewGroup）。
MaterialApp的页面路由通过routes+onGenerateRoute+onUnknownRoute决定。在对应widget显示到页面上前，会经过builder(TransitionBuilder)的处理，在builder中可以做一些全局的转化配置，这里单独提这点是因为Flutter_Boost库就是通过这个builder来实现路由跳转的。

##### BuildContext

BuildContext是一个抽象类，BuildContext广泛的出现在我们的开发过程中，Widget的build函数，Navgator的of函数，Theme的of函数。那这些函数里传入的BuildContext实例到底是啥呢，其实一打开这个类，我们就可以看到如下的注释，明确告诉我们BuildContext对象其实是Element对象。
```
/// [BuildContext] objects are actually [Element] objects. The [BuildContext]
/// interface is used to discourage direct manipulation of [Element] objects.
```

开发中我们能获取到BuildContext对象的实例的都都在build函数（StatelessWidget或者State）中。从Flutter渲染过程中我们了解到build函数传入的就是该widget的element对象。


##### Navgator
首先Navgator是一个Widget，上面说道MaterialApp是顶层容器，MaterialApp的build函数中可以看到内部嵌套了WidgetsApp。我们在看WidgetsApp的build函数，返回的是一个Navgator，这个过程中会在Navgator外叠上不同的功能的Widget。

我们通过如下代码进行页面跳转：
```
Navigator.of(context).pushNamed("routeName");
```

其中Navigator.of(context)就是拿到Navigator这个widget的State。正常来说我们App里只会存在一个Navgator，这个of函数会沿着element tree的结构向上寻找出类型是NavigatorState的State（最近State还是最远State由rootNavigator决定）。

pushNamed就是调用NavigatorState的函数。通过在MaterialApp中设置的路由信息创建需要跳转的Route对象。

###### Route
我们这里主要看MaterialPageRoute的实现，我们也可以通过继承PageRoute来实现特殊要求的Route。MaterialPageRoute的继承关系如下：
> class MaterialPageRoute<T> extends PageRoute<T>{...}
> 
> abstract class PageRoute<T> extends ModalRoute<T>{...}
> 
> abstract class ModalRoute<T> extends TransitionRoute<T> with LocalHistoryRoute<T>{...}
> 
> abstract class TransitionRoute<T> extends OverlayRoute<T>{...}
> 
> abstract class OverlayRoute<T> extends Route<T>{...}

我们看下Route类的注释：
>/// An abstraction for an entry managed by a [Navigator].
>///
>/// This class defines an abstract interface between the navigator and the
>/// "routes" that are pushed on and popped off the navigator. Most routes have
>/// visual affordances, which they place in the navigators [Overlay] using one
>/// or more [OverlayEntry] objects.

可以看到Route是给Navigator用来管理entry的。这个entry是OverlayEntry可以指一个画面。

对于Route类来说最重要的是install函数。

```dart
  /// Called when the route is inserted into the navigator.
  ///
  /// Use this to populate [overlayEntries] and add them to the overlay
  /// (accessible as [Navigator.overlay]). (The reason the [Route] is
  /// responsible for doing this, rather than the [Navigator], is that the
  /// [Route] will be responsible for _removing_ the entries and this way it's
  /// symmetric.)
  ///
  /// The `insertionPoint` argument will be null if this is the first route
  /// inserted. Otherwise, it indicates the overlay entry to place immediately
  /// below the first overlay for this route.
  @protected
  @mustCallSuper
  void install(OverlayEntry insertionPoint) { }
```

install函数主要用来在navigator中插入对于的route（insertionPoint）。真正的插入实现在OverlayRoute中

```dart
  @override
  void install(OverlayEntry insertionPoint) {
    assert(_overlayEntries.isEmpty);
    _overlayEntries.addAll(createOverlayEntries());
    navigator.overlay?.insertAll(_overlayEntries, above: insertionPoint);
    super.install(insertionPoint);
  }
```

主要就两步，1.根据这个Route的信息创建OverlayEntries。2.在对应的Overlay中插入这些OverlayEntries，插入到insertionPoint之上。这时该Route对应的OverlayEntries就被插入在最顶部进行显示。

createOverlayEntries()是个空函数，具体实现在子类ModalRoute中：
```dart
  @override
  Iterable<OverlayEntry> createOverlayEntries() sync* {
    yield _modalBarrier = OverlayEntry(builder: _buildModalBarrier);
    yield OverlayEntry(builder: _buildModalScope, maintainState: maintainState);
  }
```

这里用到了同步生成器函数，函数返回了包含_modalBarrier和OverlayEntry(builder: _buildModalScope, maintainState: maintainState)的两个OverlayEntry元素的迭代器。

##### Overlay


### Flutter渲染






### io.flutter.app.FlutterActivity

FlutterActivity的实现相当简单，但是可以作为我们查阅源码的入口。
```java
    private final FlutterActivityDelegate delegate = new FlutterActivityDelegate(this, this);
    private final FlutterActivityEvents eventDelegate; //Android Activity事件代理、包括生命周期函数
    private final Provider viewProvider;//提供FlutterView
    private final PluginRegistry pluginRegistry;//插件相关
```

关注着几个全局变量，创建一个FlutterActivityDelegate，其他三个变量其实都是这个delegate，只是通过单一职责的设计模式通过不同的接口定义拆分功能。

### io.flutter.app.FlutterActivityDelegate

Activity的onCreate通过eventDelegate在FlutterActivityDelegate中实现，做的也是创建视图的工作。


### io.flutter.view.FlutterNativeView和io.flutter.view.FlutterView

FlutterNativeView实现了BinaryMessenger接口，作为Android和Flutter的的信道，提供send函数来发送消息。FlutterNativeView的全局变量：
```java
    private final FlutterPluginRegistry mPluginRegistry;
    private final DartExecutor dartExecutor;
    private FlutterView mFlutterView;
    private final FlutterJNI mFlutterJNI;
```
FlutterJNI是执行libflutter.so相关函数的类，例如初始化的时候，AOT模式下拼接shellArgs的方式调用FlutterJNI.nativeInit（静态函数）进行使用，在C++层进行加载libapp.so。例如通信的时候调用nativeDispatchPlatformMessage（）函数来发送消息。FlutterJNI对象和FlutterNativeView绑定。不同的FlutterNativeView存在不同的FlutterJNI对象。

DartExecutor本身也实现了BinaryMessenger接口，FlutterNativeView对BinaryMessenger接口函数的实现都是调用了DartExecutor进行。而DartExecutor内对原生和Flutter的通信就是调用FlutterJNI对象实现的。同时DartExecutor也是Dart代码的执行器，在FlutterActivityDelegate初始时可以看到调用executeDartEntrypoint()来执行dart代码，当然内部也是调用的FlutterJNI对象实现。

FlutterPluginRegistry是做插件注册的，我们后面单独阅读下相关实现。这里主要负责创建该FlutterView对应的FlutterPluginRegistry提供使用。

FlutterView是真正显示Flutter页面的地方，本身继承于SurfaceView，使用双缓冲机制来优化渲染。同时也是
我们首先来看下FlutterView的初始化工作：
```java
  public FlutterView(Context context, AttributeSet attrs, FlutterNativeView nativeView) {
        super(context, attrs);

        Activity activity = getActivity(getContext());
        if (activity == null) {
            throw new IllegalArgumentException("Bad context");
        }
        //step 1:创建FlutterNativeView对象，从上文可知和Flutter层的交互实际上都是通过mNativeView实现。
        if (nativeView == null) {
            mNativeView = new FlutterNativeView(activity.getApplicationContext());
        } else {
            mNativeView = nativeView;
        }
        //step 2:获取dartExecutor，上文可知dartExecutor即是二进制信道，也是执行Dart代码的地方。
        dartExecutor = mNativeView.getDartExecutor();
        //step 3:获取flutterRenderer，并不是说原生层通过flutterRenderer来渲染Flutter页面，在后面可以看到主要来做手势数据的传递。
        flutterRenderer = new FlutterRenderer(mNativeView.getFlutterJNI());
        mIsSoftwareRenderingEnabled = mNativeView.getFlutterJNI().nativeGetIsSoftwareRenderingEnabled();
        //step 4:视图尺寸。
        mMetrics = new ViewportMetrics();
        mMetrics.devicePixelRatio = context.getResources().getDisplayMetrics().density;
        setFocusable(true);
        setFocusableInTouchMode(true);
        //step 5:attach，让mNativeView持有FlutterView
        mNativeView.attachViewAndActivity(this, activity);
        //step 6:SurfaceView的Callback，相关事件传递给Flutter。
        mSurfaceCallback = new SurfaceHolder.Callback() {
            @Override
            public void surfaceCreated(SurfaceHolder holder) {
                assertAttached();
                mNativeView.getFlutterJNI().onSurfaceCreated(holder.getSurface());
            }

            @Override
            public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
                assertAttached();
                mNativeView.getFlutterJNI().onSurfaceChanged(width, height);
            }

            @Override
            public void surfaceDestroyed(SurfaceHolder holder) {
                assertAttached();
                mNativeView.getFlutterJNI().onSurfaceDestroyed();
            }
        };
        getHolder().addCallback(mSurfaceCallback);
        //step 7:设置相关监听器，通过mFirstFrameListeners可以做Flutter首帧的监听。
        mActivityLifecycleListeners = new ArrayList<>();
        mFirstFrameListeners = new ArrayList<>();
        //step 8:平台相关Channel，这些都是和一个FlutterView对应。
        // Create all platform channels
        navigationChannel = new NavigationChannel(dartExecutor);
        keyEventChannel = new KeyEventChannel(dartExecutor);
        lifecycleChannel = new LifecycleChannel(dartExecutor);
        localizationChannel = new LocalizationChannel(dartExecutor);
        platformChannel = new PlatformChannel(dartExecutor);
        systemChannel = new SystemChannel(dartExecutor);
        settingsChannel = new SettingsChannel(dartExecutor);
        //step 9 :通过platformChannel来通知Activity的onPostResume事件
        // Create and setup plugins
        PlatformPlugin platformPlugin = new PlatformPlugin(activity, platformChannel);
        addActivityLifecycleListener(new ActivityLifecycleListener() {
            @Override
            public void onPostResume() {
                platformPlugin.updateSystemUiOverlays();
            }
        });
        //step 10 :文本输入TextInputPlugin,这里的InputMethodManager在TextInputPlugin有创建。
        mImm = (InputMethodManager) getContext().getSystemService(Context.INPUT_METHOD_SERVICE);
        PlatformViewsController platformViewsController = mNativeView.getPluginRegistry().getPlatformViewsController();
        mTextInputPlugin = new TextInputPlugin(this, dartExecutor, platformViewsController);
        //step 11 :Android 点击事件相关处理。
        androidKeyProcessor = new AndroidKeyProcessor(keyEventChannel, mTextInputPlugin);
        //step 12 :Android 手势事件相关处理，和上面flutterRenderer创建对应
        androidTouchProcessor = new AndroidTouchProcessor(flutterRenderer);
        mNativeView.getPluginRegistry().getPlatformViewsController().attachTextInputPlugin(mTextInputPlugin);
        //step 11: 通过localizationChannel传递语言信息和通过systemChannel传递用户设置信息
        // Send initial platform information to Dart
        sendLocalesToDart(getResources().getConfiguration());
        sendUserPlatformSettingsToDart();
    }
```
通过看FlutterView的初始化函数，我们了解到除了做页面显示工作外，FlutterView还负责Flutter相关类的创建及Android系统相关事件的通信。

### 两个FlutterActivity实现的比较 

我们将






