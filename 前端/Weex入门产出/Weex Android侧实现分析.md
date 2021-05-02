Weex Android侧实现分析
=====================

### Weex显示页面主要类

###### WXEnvironment

Weex环境配置类，标记Weex的debug开关，日志开关，平台信息等配置。在初始化时常会如下设置：
```Java
    WXEnvironment.sDebugServerConnectable = connectable;
    WXEnvironment.sRemoteDebugMode = debug;
    WXEnvironment.sRemoteDebugProxyUrl = "ws://" + host + ":8088/debugProxy/native";
```

###### WXSDKEngine
Weex执行引擎，整个app中只需要有一个WXSDKEngine实例，为JS层和native层提供通道。我们为Weex提供的能力都在WXSDKEngine类中注册。

weex项目启动的时候，需要对WXSDKEngine做初始化，完成后就可以使用weex了。
```Java
WXSDKEngine.addCustomOptions("appName", "WXSample");
WXSDKEngine.initialize(this,
       new InitConfig.Builder()
           .setImgAdapter(new ImageAdapter())
           .setWebSocketAdapterFactory(new DefaultWebSocketAdapterFactory())
           .setJSExceptionAdapter(new JSExceptionAdapter())
           .setHttpAdapter(new InterceptWXHttpAdapter())
           .setApmGenerater(new ApmGenerator())
           .build()
      );
```

我们可以通过WXSDKEngine.addCustomOptions来给Weex设置全局参数，这个就想是WebView的UserAgent，在WXBridge初始化的时候设置，所有weex页面都能获取。

###### WXSDKInstance
Weex运行实例，WXSDKInstance会和Weex容器（Activity）绑定，因此会存在多个实例。

WXSDKInstance赋予了Activity操作Weex页面的能力。例如渲染，刷新，渲染进度监听，设置滚动视图及监听等。

同时也需要Activity同步生命周期给WXSDKInstance，用来维护Weex的生命周期。

想要Weex容器显示一个页面，简单的只需要如下实现：
```Java
@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    mWXSDKInstance.registerRenderListener(this);
    Map<String, Object> options = new HashMap<>();
    options.put(WXSDKInstance.BUNDLE_URL, mUrl);
    mWXSDKInstance.renderByUrl(PAGE_NAME, mUrl, options, null, WXRenderStrategy.APPEND_ASYNC);
    ...
}

@Override
public void onViewCreated(WXSDKInstance instance, View view) {
    if (view.getParent() != null) {
        ((ViewGroup) view.getParent()).removeView(view);
    }
    mProgress.setVisibility(View.INVISIBLE);
    mContainer.removeAllViews();
    mContainer.addView(view);
}
```

其中options是我们给Weex页面传递的参数，相当于H5页面我们是通过URL里凭借参数。但在H5上我们可以在Cookie里设置的方式提供参数给前段，针对Weex只能只能通过WXModule等方式获取。


###### WXSDKManager

WXSDKEngine（WXBridge）是全局唯一的，因此我们给Weex提供的能力（WXModule等）是单例的。但WXSDKInstance是跟Weex容器（Weex页面）走的，因此多个WXSDKInstance（Weex页面）使用了原生能力时，如何正确的回调就成为了问题。WXSDKManager就是用来管理多个WXSDKInstance实例，保证WXBridge能够正确回调。

```Java
public WXSDKInstance(Context context) {
    mInstanceId = WXSDKManager.getInstance().generateInstanceId();
    init(context);
}
```

WXSDKInstance在创建实例的时候，会从WXSDKManager中回去一个InstanceId作为WXSDKInstance实例的标记，之后的JS异常通知，回调都这个ID来确定是哪个WXSDKInstance。

WXSDKManager提供了对WXSDKInstance实例的监听，可以看到也是通过instanceId实现。
```Java
public void registerInstanceLifeCycleCallbacks(InstanceLifeCycleCallbacks callbacks) {
    if (mLifeCycleCallbacks == null) {
      mLifeCycleCallbacks = new ArrayList<>();
    }
    mLifeCycleCallbacks.add(callbacks);
}

public interface InstanceLifeCycleCallbacks {
    void onInstanceDestroyed(String instanceId);
    void onInstanceCreated(String instanceId);
}

```

WXBridgeManager中，JS调用WXModule时也是通过InstanceId来确定WXSDKInstance，继而确定是在哪个页面中调用。

```Java
static Object callModuleMethod(final String instanceId, String moduleStr, String methodStr, JSONArray args) {
    ...
    WXSDKInstance instance = WXSDKManager.getInstance().getSDKInstance(instanceId);
    wxModule.mWXSDKInstance = instance;
    ...
}
```

###原生控件及功能的提供

######Adapter
Adapter是Weex框架自身提供的基础组件，功能需要用到的扩展，部分扩展提供了默认实现。如果不需要使用其相应功能，可以不做定义。例如IWXHttpAdapter:







