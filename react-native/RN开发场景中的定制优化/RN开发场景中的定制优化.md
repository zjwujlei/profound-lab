RN实际开发场景中的定制优化
=====================

### 模块化开发场景下ReactApplication的优化

ReactNativeHost类可以认为是React Native的Application，全局单例、持有ReactInstanceManager实例，同时也是ReactNative的配置类，JSBundle文件、开发者模式等。

ReactNativeHost类默认通过ReactApplication接口绑定到了Application中，获取是通过将Application实例强转为ReactApplication后调用getReactNativeHost获取。例如ReactActivityDelegate中实现：

```Java
    protected ReactNativeHost getReactNativeHost() {
        return ((ReactApplication) getPlainActivity().getApplication()).getReactNativeHost();
    }
```

这种方式在模块化开发场景下就不合理了。RN模块的具体实现需要侵入并依赖宿主模块,且多模块使用了RN技术时整个APP就会存在多个JSBundle文件。

查看ReactNativeHost源码，其类注释如下：
```Java
/**
 * Simple class that holds an instance of {@link ReactInstanceManager}. This can be used in your
 * {@link Application class} (see {@link ReactApplication}), or as a static field.
 */
```

这里也说到可以作为一个静态属性。因此我们只需要保证ReactNativeHost类在当前模块下全局单例，可以将ReactNativeHost实例绑定到每个模块自己的生命周期上。

查看源码发现ReactApplication.getReactNativeHost()函数只在HeadlessJsTaskService，ReactActivityDelegate，BlobProvider三处进行使用，对其分析发现都是可以进行重写来实现ReactNativeHost实例获取方式替换，这里以ReactActivityDelegate为例。

ReactActivityDelegate作为RN页面的代理，只在ReactActivity基类中使用。我们只需要提供新的Delegate类即可实现:

新的Delegate类：RNModuleActivityDelegate主要实现。
```Java
public class RNModuleActivityDelegate extends ReactActivityDelegate{

    protected PracticalReactNativeHost getReactNativeHost() {
        ReactNativeModule module = ((IModuleApplication)getPlainActivity().getApplication()).getModule(ReactNativeModule.class);
        return module.getReactNativeHost();
    }
}
```

其中ReactNativeModule类为模块生命周期类。也可以通过其他方式提供，只需要保证模块内全局单例。

*onCreate等部分函数存在调用权限问题，需要做修改*

*如果ReactNativeModule类没有保证单例，会出现页面渲染显示且可滑动，但点击无响应等奇怪现象*

然后实现Activity基类，这里可以照抄ReactActivity,只需要替换RNModuleActivityDelegate。

HeadlessJsTaskService，BlobProvider的实现也是类似，替换ReactNativeHost的获取方式即可。

###JSBundle预加载
React Native页面在首次启动过程中，会进行初始化ReactCoutext,这个过程中会加载JSBundle文件。这是一个较耗时的IO操作，会出现白屏现象，为了解决这个问题，可以需要对ReactCoutext进行预创建,我们只需要在合适的时机调用如下代码即可实现预创建。

```Java
    ReactNativeModule module = ((IModuleApplication)getApplication()).getModule(ReactNativeModule.class);
    ReactNativeHost host = module.getReactNativeHost();
    ReactInstanceManager mManager = host.getReactInstanceManager();
    mManager.createReactContextInBackground();
```


*特别注意的，预加载需要在UI线程里进行调用。*

###多MainComponent支持

React Native和原生混合开发时，RN实现页面很难做到完全的闭包，这样就存在需要有多个MainComponent暴露为Activity的情况。

在JS代码中，提供了AppRegistry来注册Component，这是一个JavaScriptModule，原生可以直接调用，具体可以看react component显示过程源码。因此我们在JS层需要调用AppRegistry将所有要暴露的MainComponent都进行注册。

###### 原生测定制
在原生侧，可以为每个MainComponent都定义一个Activity，但这样如果需要暴露的MainComponent过多时就不适合了，是否可以通过入参的方式确定当前要现实哪个MainComponent?

我们看一下ReactActiviy中getMainComponentName()方法设置的mainComponentName是怎么使用的。

```Java
public abstract class ReactActivity extends Activity implements DefaultHardwareBackBtnHandler, PermissionAwareActivity {
    private final ReactActivityDelegate mDelegate = this.createReactActivityDelegate();

    protected ReactActivity() {
    }

    @Nullable
    protected String getMainComponentName() {
        return null;
    }

    protected ReactActivityDelegate createReactActivityDelegate() {
        return new ReactActivityDelegate(this, this.getMainComponentName());
    }
}
```

可以看到getMainComponentName()是在ReactActivity对象创建的时候就被使用，如果我们直接在重写的getMainComponentName()中通过Intent获取入参，此时getIntent()返回必定是Null，会有空指针异常。

在查看ReactActivityDelegate，使用到mMainComponentName的有两处:
```Java
protected void onCreate(Bundle savedInstanceState) {
    ...
    if (mMainComponentName != null && !needsOverlayPermission) {
      loadApp(mMainComponentName);
    }
    ...
}
public void onActivityResult(int requestCode, int resultCode, Intent data) {
    ...
          if (mMainComponentName != null) {
            loadApp(mMainComponentName);
          }
    ...
}
```

mMainComponentName真正使用必定在onCreate之后，我们对ReactActivityDelegate进行改造，在真正用到之前通过Intent获取MainComponentName。

新的Delegate类：PracticalReactActivityDelegate部分代码
```Java
public class PracticalReactActivityDelegate {
    private @Nullable String mMainComponentName;
    public PracticalReactActivityDelegate(Activity activity) {
        mActivity = activity;
        mFragmentActivity = null;
    }

    public PracticalReactActivityDelegate(
            FragmentActivity fragmentActivity) {
        mFragmentActivity = fragmentActivity;
        mActivity = null;
    }
    protected void onCreate(Bundle savedInstanceState,String mainComponentName) {
        this.mMainComponentName = mainComponentName;
        ...
    }
}

```

新的Activity基类：PracticalReactActivity部分代码

```Java
public abstract class PracticalReactActivity extends Activity
        implements DefaultHardwareBackBtnHandler, PermissionAwareActivity {

    protected PracticalReactActivityDelegate createReactActivityDelegate() {
        return new PracticalReactActivityDelegate(this);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mDelegate.onCreate(savedInstanceState, getMainComponentName());
    }
}
```

这样我们在重写getMainComponentName()时就可以通过Intent获取入参来获取需要显示的MainComponent了。

###### JS测多入口情定制
如果我们采用的是原生+RN混合开发的形式，

###多JSBundle文件情况

默认情况下ReactNativeHost，JSBundle文件是意义对应的。在多模块开发情况下，ReactNativeHost绑定在模块的生命周期函数内，因此只要保证每个模块只有一个JSBundle文件上是不存在问题的。

但如果一个模块存在多个JSBundle文件文件，或者我们的模块并没有生命周期的概念，就会存在多JSBundle文件的情况。

















