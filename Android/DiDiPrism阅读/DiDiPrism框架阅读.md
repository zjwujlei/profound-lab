DiDiPrism框架阅读
================

## 介绍

#### 一、操作回放（已开源）
小桔棱镜中最具创新性的功能，也是整个棱镜平台的基础，我们基于自研的操作行为标识指令实现了在APP端的操作回放（视频回放 / 文字回放）。相比于传统的静态埋点数据它提供了动态的操作行为，可以帮助大家更好的定位问题、优化产品，为用户创造价值。

当然它还可以有很多应用场景，比如无需手写脚本的自动化测试场景，仅单纯的操作行为标识指令就可以被应用到很多我们还没有想到但已经收到诉求的场景中，因此我们选择把它开源出来造福更多人。

#### 二、操作检测（已开源）
端侧实时操作行为检测功能，同样基于自研的操作行为标识指令以及语义化的操作行为策略描述方案（DSL），支持丰富的语义和灵活的策略配置。它可以帮助我们实现端侧场景化需求，未来还希望用在客服场景中来提升用户体验，创造更多用户价值。

#### 三、数据可视化（逐步开放中..）
覆盖埋点全流程的移动端解决方案，包括埋点数据可视化范畴的多维度PV/UV、热力图、转化率漏斗、页面停留时长等功能，以及埋点辅助范畴的测试工具。它的意义在于改变了大家日常看数据的方式，让原本就擅长使用数据的同学可以更便捷的用数据，让原本不擅长使用数据的同学开始喜欢用数据。

## 操作检测 prism-monitor

对于操作检测能力，首先要进行初始化，可以放在Application的onCreate中调用：

```Java
PrismMonitor.getInstance().init(this);
```

初始化完成后在合适的时机启动并设置监听器，点击事件都会在监听器中进行回调，可以在这里进行事件上报。
```Java
PrismMonitor.OnPrismMonitorListener mOnPrismMonitorListener = new PrismMonitor.OnPrismMonitorListener() {
    @Override
    public void onEvent(EventData eventData) {
        Log.d("onEvent2", eventData.getUnionId());
        mPlaybackEvents.add(eventData);
    }
};
PrismMonitor.getInstance().start();
PrismMonitor.getInstance().addOnPrismMonitorListener(mOnPrismMonitorListener);
```
对于init函数，重点逻辑代码如下：
```java
public void init(Application application) {
    //获取TouchSlop，用来判断是点击事件还是滑动事件。
    ViewConfiguration vc = ViewConfiguration.get(context);
    sTouchSlop = vc.getScaledTouchSlop();
    //设置进程生命周期回调监听，用来发送应用前台/后台事件。
    //但看master分支代码，AppLifecycleObserver中具体逻辑都被移除了。
    mAppLifecycleObserver = new AppLifecycleObserver();
    ProcessLifecycleOwner.get().getLifecycle().addObserver(mAppLifecycleObserver);
    //做GlobalWindowManager的初始化，这里在后面事件拦截部分会详细展开。
    GlobalWindowManager.getInstance().init(context);
    //创建Activity监听，用来发送Activity生命周期函数事件
    mActivityLifecycleCallbacks = new ActivityLifecycleCallbacks();
    //创建监听，在start函数中会使用，只要是调用setWindowCallback(window);函数。
    //用来保证当前添加的Window使用了我们定制的PrismMonitorWindowCallbacks。
    mWindowObserverListener = new WindowObserver.WindowObserverListener() {
        @Override
        public void add(final Window window) {
            setWindowCallback(window);
        }

        @Override
        public void remove(Window window) {

        }
    };
}
```
然后是start函数
```java
public void start() {
    //设置在init函数中创建的ActivityLifecycle监听器
    mApplication.registerActivityLifecycleCallbacks(mActivityLifecycleCallbacks);
    //需要保证所有Window对象使用的WindowCallback都是PrismMonitorWindowCallbacks。
    //因此对于在调用start函数之前的window对象通过for循环替换，之后的通过监听器替换。
    WindowObserver windowObserver = GlobalWindowManager.getInstance().getWindowObserver();
    windowObserver.addWindowObserverListener(mWindowObserverListener);

    for (int i = 0; i < windowObserver.size(); i++) {
        View view = windowObserver.get(i);
        Window window = (Window) view.getTag(R.id.prism_window);
        if (window == null) {
            windowObserver.bindWindow(view);
            window = (Window) view.getTag(R.id.prism_window);
        }
        if (window != null && !(window.getCallback() instanceof WindowCallbacks)) {
            setWindowCallback(window);
        }
    }
}
````

#### 事件拦截

使用Window.Callback来进行全局事件拦截，包括touch和key事件。在init和start函数中保证了所有Window使用的都是PrismMonitorWindowCallbacks。
主要是GlobalWindowManager、WindowObserver和PrismMonitorWindowCallbacks。
我们在init函数中会调用GlobalWindowManager的初始化，其主要是执行reflectProxyWindowManager函数，通过反射替换mViews数组对象，通过重写其add和remove函数来达到监听Window的目的。
这里我们可以先看看 <a href="https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/WindowManagerImpl.java">WindowManagerImpl</a> 和<a href="https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/WindowManagerGlobal.java">WindowManagerGlobal</a>。然后看reflectProxyWindowManager函数做的事情就很清晰了，将windowManager中原mViews数组中的元素添加到WindowObserver中，用WindowObserver对象替换掉windowManager中原mViews。这里需要学习一些WindowManager相关的知识。

```java
private void reflectProxyWindowManager(Context context) {
    Object windowManager = context.getSystemService(Context.WINDOW_SERVICE);
    Class windowManagerImplClass = windowManager.getClass();
    Field windowManagerGlobalField = windowManagerImplClass.getDeclaredField("mGlobal");
    windowManagerGlobalField.setAccessible(true);
    Object windowManagerGlobal = windowManagerGlobalField.get(windowManager);

    Field mViewsField = windowManagerGlobal.getClass().getDeclaredField("mViews");
    mViewsField.setAccessible(true);
    Object value = mViewsField.get(windowManagerGlobal);
    if (value instanceof List) {
        List views = (List) mViewsField.get(windowManagerGlobal);
        mWindowObserver.addAll(views);
        mViewsField.set(windowManagerGlobal, mWindowObserver);
    }
}
```

然后我们看一下WindowObserver，继承的ArrayList<View>。这里主要看下add函数，remove就是移除监听的过程。
```java
public boolean add(View view) {
    //确定当前的view是不是DecorView
    if (mDecorClass == null) {
        mDecorClass = view.getRootView().getClass();
    }
    if (view.getClass() != mDecorClass) {
        return super.add(view);
    }
    //如果是DecorView，通过反射获取mWindow字段取得Window对象。
    Window window = getWindow(view);
    if (window == null) {
        return super.add(view);
    }
    //通过Tag的方式进行保存的，避免多次反射获取
    view.setTag(R.id.prism_window, window);
    //最后在所有的listener中通知出add事件。
    //这里的listener就是在PrismMonitor的init函数中创建，start函数中设置的mWindowObserverListener。
    //收到回调后会进行WindowCallback的替换。如果我们有设置其他的监听器也会收到。
    for (int i = 0; i < mListeners.size(); i++) {
        WindowObserverListener listener = mListeners.get(i);
        if (listener != null) {
            listener.add(window);
        }
    }
    return super.add(view);
}
```

最后来看一下PrismMonitorWindowCallbacks，前面两步其实就是为了保证所有Window的Window.Callback都是PrismMonitorWindowCallbacks的实例。
PrismMonitorWindowCallbacks继承自WindowCallbacks，而WindowCallbacks声明了Window.Callback。我们来看下touch事件做了何种处理，key事件其实也类似。

对于WindowCallbacks，主要是一个代理的作用，构造函数要求一个原Window.Callback对象mCallBack，所有Window.Callback接口函数实现都会代理mCallBack的实现。
对于dispatchTouchEvent则会同时调用touchEvent函数，子类可以通过重写touchEvent函数来做事件拦截。
```java
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    return mCallBack != null && (touchEvent(event) || mCallBack.dispatchTouchEvent(event));
}
```
然后是PrismMonitorWindowCallbacks子类的touchEvent函数，这里就拦截到了具体的事件，可以进行逻辑操作。
```java
public boolean touchEvent(MotionEvent event) {
    //内部有整个视图树的遍历，比较耗时，只有开启的时候才会使用。
    if (mPrismMonitor.isMonitoring()) {
        //记入当前这次event事件的信息，从down一直到up是一个TouchRecord实例，每次down事件都会重新生成一个。
        TouchRecordManager.getInstance().touchEvent(event);
        int action = event.getActionMasked();
        if (action == MotionEvent.ACTION_UP) {
        //对于up事件，TouchRecord内部会通过PrismMonitor.sTouchSlop判断是否是点击事件。
        //如果是点击事件，通过TouchTracker.findTargetView()函数获取点击的targetView，这里需要遍历整个视图树。
        //然后调用TouchEventHelper.createEventData()函数获取这次点击事件的EventData，里面会获取xpath，需要对targetView溯源也是一个循环。
        //构造好EventData后调用mPrismMonitor.post()进行上报。
            TouchRecord touchRecord = TouchRecordManager.getInstance().getTouchRecord();
            if (touchRecord != null && touchRecord.isClick) {
                int[] location = new int[]{(int) touchRecord.mDownX, (int) touchRecord.mDownY};
                View targetView = TouchTracker.findTargetView((ViewGroup) window.getDecorView(), touchRecord.isClick ? location : null);
                if (targetView != null) {
                    EventData eventData = TouchEventHelper.createEventData(window, targetView, touchRecord);
                    if (eventData != null) {
                        mPrismMonitor.post(eventData);
                    }
                }
            }
        }
    }
    return false;
}
```

#### xPath
xPath是一个概念，在XML里面运用的定义是：使用路径表达式来选取 XML 文档中的节点或者节点集。用在这里也一样，通过路径表达式来表述节点在Android视图树中的位置。

但在Android视图树的场景中，视图树可能是动态构建的，因此对路径的定义就会十分关键。这里通过ViewPath来记录一个唯一控件的全部信息。

```java
public class ViewPath {
    //xpath 字符串，例如demo中在ViewPager第一页中的一个按钮其path：btn/*/viewpager/content[01]/decor_content_parent/
    public String path;
    //在源码中没有体现，猜测是一种特殊脱离原生控件的视图容器的逻辑处理。例如Flutter、完全canvas自绘控件。
    //如果是这种容器就记录symbol（类型？）和url（内容标记？）
    public ViewContainer viewContainer;
    //如果是WebView，会记录其对应URL。
    public String webUrl;
    //列表控件信息记录，在路径上有多个列表控件会通过“,”分割,
    //例如demo中在ViewPager第一页中的一个按钮其listInfo:"v:0,0"，v代表viewpage，第一个0代表第一页，第二个0代表index。
    public String listInfo;
    //当前十分在可滚动控件中
    public boolean inScrollableContainer;

}
```
其获取在getViewPathInfo函数中：
```java
private static ViewPath getViewPathInfo(View touchView, TouchRecord touchRecord, StringBuilder eventId, EventData eventData) {
    ViewPath viewPath = new ViewPath();
    //如果当前点击的是WebView容器，通过viewPath.webUrl记录URL并标记inScrollableContainer为true。
    if (touchView instanceof WebView) {
        WebView webView = (WebView) touchView;
        String webUrl = webView.getUrl();
        if (!TextUtils.isEmpty(webUrl)) {
            Uri uri = Uri.parse(webUrl);
            viewPath.webUrl = uri.getScheme() + "://" + uri.getHost() + uri.getPath();
        }
        viewPath.inScrollableContainer = true;
    }

    boolean hasLastViewId = false;
    String listInfo = null;
    StringBuilder viewPathBuilder = new StringBuilder();
    //通过do-while循环进行溯源
    do {
        ViewParent viewParent = touchView.getParent();

        if (viewParent instanceof ViewGroup) {
            //取得在父容器中的index
            ViewGroup viewGroup = (ViewGroup) viewParent;
            int index = viewGroup.indexOfChild(touchView);
            boolean isList = false;
            String positionInfo = null;
            if (touchRecord.isClick && viewParent instanceof AbsListView) {
                //如果是在AbsListView中的点击事件，标记isList和viewPath.inScrollableContainer为true。
                //对于AbsListView设计到控件复用，需要使用position来标记在父容器中的位置（positionInfo）
                //拼接到listInfo中，后面设置到ViewPath中。
                viewPath.inScrollableContainer = true;
                AbsListView listView = (AbsListView) viewParent;
                int[] location = new int[2];
                listView.getLocationOnScreen(location);
                //格式：“l:${position},${index}”
                positionInfo = "l:" + listView.pointToPosition((int) touchRecord.mDownX - location[0], (int) touchRecord.mDownY - location[1]) + "," + index;
                isList = true;
                if (listInfo == null) {
                    listInfo = positionInfo;
                } else {
                    listInfo += "," + positionInfo;
                }
            } else if (touchRecord.isClick && viewParent instanceof RecyclerView) {
                //对于recycle，同样的逻辑，只是可以通过getChildAdapterPosition直接拿到Position。
                viewPath.inScrollableContainer = true;
                RecyclerView recyclerView = (RecyclerView) viewParent;
                //格式：“r:${position},${index}”
                positionInfo = "r:" + recyclerView.getChildAdapterPosition(touchView) + "," + index;
                isList = true;
                if (listInfo == null) {
                    listInfo = positionInfo;
                } else {
                    listInfo += "," + positionInfo;
                }
            } else if(touchRecord.isClick && viewParent instanceof ViewPager) {
                //对于ViewPager，则要标记它的index。
                viewPath.inScrollableContainer = true;
                ViewPager viewPager = (ViewPager) viewParent;
                //格式：“r:${pageIndex},${index}”
                positionInfo = "v:" + viewPager.getCurrentItem() + "," + index;
                isList = true;
                if (listInfo == null) {
                    listInfo = positionInfo;
                } else {
                    listInfo += "," + positionInfo;
                }
            } else if (viewParent instanceof ScrollView || viewParent instanceof HorizontalScrollView) {
                //对于其他滚动事件进行标记。
                viewPath.inScrollableContainer = true;
            }
            //这里才是真正生成xPath的逻辑。
            //如果有resourceName，使用resourceName标记控件。
            //如果是列表控件，通过*标记控件，会和listInfo配合使用，后面会讲到。
            //
            String resourceName = getResourceName(touchView.getContext(), touchView.getId());
            if (resourceName != null) {
                viewPathBuilder.append(resourceName).append("/");
                hasLastViewId = true;
            } else if (isList) {
                viewPathBuilder.append("*").append("/"); // 替换符，保持vp一致，去vl取
            } else {
                //这里看到，但有过resourceName的pach后，就不拼接index的path了。
                if (!hasLastViewId) {
                    viewPathBuilder.append(index).append("/");
                }
            }
            touchView = viewGroup;
        } else {
            //如果父容器是不是ViewGroup直接跳出。
            //我们可能觉得奇怪，只有ViewGroup内部才可以添加子控件。
            //这是因为DecorView（com.android.internal.policy.DecorView）的父容器是ViewRootImpl。
            //而ViewRootImpl并没有继承ViewGroup或者View，声明了ViewParent接口。
            break;
        }
    } while (true);
    if (listInfo != null) {
        viewPath.listInfo = listInfo;
    }
    viewPath.path = viewPathBuilder.toString();
    return viewPath;
}
```

这里有一些记录的数据，如listInfo，inScrollableContainer等，在后面用到的时候具体展开。

#### 埋点信息EventData及其收集
我们来在看下一个点击事件完整收集的数据EventData：

```java
public class EventData {
    //事件发生的事件
    public long eventTime;
    //事件类型，8种包括点击、生命周期函数等，在PrismConstants.Event中定义
    public int eventType;
    //ID，对于非点击事件，就是Activty信息（PrismConstants.Symbol.ACTIVITY_NAME + PrismConstants.Symbol.DIVIDER_INNER + activity.getClass().getName();）
    //对于点击事件会有复杂的生成逻辑。
    public String eventId;
    //存储额外的埋点数据
    public HashMap<String, Object> data;

    //Window信息，在TouchEventHelper.getWindowInfo函数确定
    //格式：w_&_${titleInfo}_&_${type}
    public String w;
    //ViewId，在TouchEventHelper.getViewId函数确定，如果有name会设置，保存的就是name字段（xml里设置的@+id/xxx）。
    public String vi;
    //VIEW_REFERENCE，在TouchEventHelper.getViewContent函数确定，记录显示的相关信息来在页面上直观的确定控件。
    //这里使用文本信息（代码里有图片控件使用图片资源进行标识的片段，但看逻辑没有完整的）。
    //对于ViewGroup会最多寻找10个子控件，对于View当前只支持TextView。
    //对于ViewGroup，获取取字体最大的文本控件和最左上角的文本控件。
    //会将文本内容记入到vr中。
    public String vr;
    //VIEW_QUADRANT，在TouchEventHelper.getQuadrant函数中确定。
    //对于在滚动容器内控件（inScrollableContainer）不会确定其象限。
    //对于不在滚动容器内控件，会根据其中心坐标计算其在页面属于上、下、左、右、中5个区域中的哪个。
    public String vq;
    //VIEW_LIST，在TouchEventHelper.createEventData函数中确定。
    //其实就是前面xPath部分中ViewPath的listInfo字段。
    public String vl;
    //VIEW_PATH，在TouchEventHelper.createEventData函数中确定。
    //其实就是前面xPath部分中ViewPath的path字段。
    public String vp;
    //WEB_URL，在TouchEventHelper.createEventData函数中确定。
    //其实就是前面xPath部分中ViewPath的webUrl字段。
    public String wu;
    //VIEW_TAG，没有赋值
    public String vf;
    //ACTIVITY_NAME，没有赋值。EventData其实是关联Window的。如果是Activity的Window，其Activity明在w字段中有体现。
    public String an;

    //拼接生成唯一ID。
    public String getUnionId() {
        return "e" + PrismConstants.Symbol.DIVIDER_INNER + eventType + (eventId != null ? (PrismConstants.Symbol.DIVIDER + eventId) : "");
    }

}
```

最后来说一下eventId字段。对于点击事件，这是一个拼接的字段。上面各个参数生成的函数中都会将信息拼接到eventId中，我们看下createEventData函数：

```java
public static EventData createEventData(Window window, View touchView, TouchRecord touchRecord) {
    if (touchView == null) {
        return null;
    }
    //创建StringBuilder
    StringBuilder eventId = new StringBuilder();
    //创建EventData，指定eventType为PrismConstants.Event.TOUCH。
    //EventData的构造函数也会指定eventTime。
    EventData eventData = new EventData(PrismConstants.Event.TOUCH);
    eventData.view = touchView;
    //获取window信息，赋值w字段，同时会将相关信息拼接到eventId中。
    getWindowInfo(window, eventId, eventData);
    //生成ViewPath，在前面部分有讲到。
    ViewPath viewPath = getViewPathInfo(touchView, touchRecord, eventId, eventData);
    if (viewPath.viewContainer != null) { // containerView
        //特定容器的话会拼接symbol和url。
        ViewContainer viewContainer = viewPath.viewContainer;
        eventId.append(PrismConstants.Symbol.DIVIDER).append(viewContainer.symbol).append(PrismConstants.Symbol.DIVIDER_INNER).append(viewContainer.url);
    } else if (!TextUtils.isEmpty(viewPath.webUrl)) { // webview
        //WebView的话会拼接其URL。
        eventId.append(PrismConstants.Symbol.DIVIDER).append(PrismConstants.Symbol.WEB_URL).append(PrismConstants.Symbol.DIVIDER_INNER).append(viewPath.webUrl);
        eventData.wu = viewPath.webUrl;
    } else { // native
        //原生控件就拼接name
        getViewId(touchView, eventId, eventData);
    }
    //vp字段，拼接path（xPath）
    eventId.append(PrismConstants.Symbol.DIVIDER).append(PrismConstants.Symbol.VIEW_PATH).append(PrismConstants.Symbol.DIVIDER_INNER).append(viewPath.path);
    eventData.vp = viewPath.path;
    //vl字段，拼接listInfo
    if (!TextUtils.isEmpty(viewPath.listInfo)) { // in list container view
        eventId.append(PrismConstants.Symbol.DIVIDER).append(PrismConstants.Symbol.VIEW_LIST).append(PrismConstants.Symbol.DIVIDER_INNER).append(viewPath.listInfo);
        eventData.vl = viewPath.listInfo;
    }
    //wr字段，拼接控件相关内容引用。
    // view content
    if (touchRecord.isClick && TextUtils.isEmpty(viewPath.webUrl)) {
        getViewContent(touchView, eventId, eventData);
    }
    //vq字段，拼接所在位置象限。
    // quadrant
    getQuadrant(touchView.getContext(), touchView, viewPath.inScrollableContainer,touchRecord, eventId, eventData);
    //最终生成了eventId。
    eventData.eventId = eventId.toString();
    return eventData;
}
```

#### 总结
对于操作检测部分，对于点击事件：
* 通过反射替换mViews数组对象全局实现了window的监听。
* 对Window添加WindowCallback实现全局事件监听。
* 通过遍历视图树按照一定逻辑生成ViewPath标记点击控件信息。
* 收集其他辅助信息生成EventData进行上报。

对于KeyEvent实现类似，在Window.Callback的dispatchKeyEvent函数中拦截实现，当前只实现了Back事件。
对于生命周期事件则通过ActivityLifecycleCallbacks实现。

## 操作回放 prism-playback

对于playback功能的使用也十分简单，在Appliaction中初始化，然后通过playback函数喂入数据（EventData数组）即可。

```java
PrismPlayback.getInstance().init(this);
PrismPlayback.getInstance().playback(mPlaybackEvents);
```

初始化init函数中，和prism-monitor一样，也设置了一个WindowObserverListener。在接受到add事件是，创建PrismWindow并通过PrismWindowManager来管理。主要是提供获取顶层Window对象的能力，顶层Window就是当前event所在的视图。

然后是playback函数
```java
public void playback(List<EventData> eventDataList) {
    //将记入转化为EventInfo。
    List<EventInfo> eventInfoList = new ArrayList<>();
    for (int i = 0; i < eventDataList.size(); i++) {
        EventData eventData = eventDataList.get(i);
        //调用PlaybackHelper.convertEventInfo函数进行转化。
        EventInfo eventInfo = PlaybackHelper.convertEventInfo(eventData);
        if (eventInfo != null) {
            eventInfoList.add(eventInfo);
        }
    }
    mCurrentIndex = 0;
    mEventInfoList = eventInfoList;
    //开始回放
    startPlayback();
}
```
playback函数很简单，是因为这里使用了handler机制，只要调用startPlayback函数开启，就会根据mEventInfoList进行一个个回放，真正调用的是next函数。

```java
private void next(final int retryTimes) {
    //播放完成。
    if (mCurrentIndex + 1 > mEventInfoList.size()) {
        Toast.makeText(mContext, "回放结束", Toast.LENGTH_LONG).show();
        mHandler.sendEmptyMessageDelayed(3, 1000);
        return;
    }

    EventInfo eventInfo = mEventInfoList.get(mCurrentIndex);
    if (eventInfo.eventType == PrismConstants.Event.TOUCH) {
        //对于点击事件，如果是webview则只做toast。
        if (eventInfo.eventData.containsKey(PrismConstants.Symbol.WEB_URL)) {
            Toast.makeText(mContext, "暂不支持web点击", Toast.LENGTH_LONG).show();
            mHandler.sendEmptyMessage(1);
        } else {
            //取得最顶层的Window，每次都获取，弹窗等Window并不是Activity的Window。
            PrismWindow prismWindow = PrismWindowManager.getInstance().getTopWindow();
            //取得这次事件被点击的控件
            View targetView = PlaybackHelper.findTargetView(prismWindow, eventInfo);
            if (targetView == null) {
                //进行重试，因为这边都是通过延时进行模拟的，例如弹出等如果1秒内还没弹出，会导致TopWindow不对而无法获取对应的targetView。
                if (retryTimes == 3) {
                    Toast.makeText(mContext, "回放失败", Toast.LENGTH_SHORT).show();
                } else {
                    Toast.makeText(mContext, "正在重试(" + (retryTimes + 1) + ")", Toast.LENGTH_SHORT).show();
                    Message message = mHandler.obtainMessage(2);
                    message.arg1 = retryTimes + 1;
                    mHandler.sendMessageDelayed(message, 1500);
                }
                return;
            }
            //如果targetView不可点击，这去寻找可点击控件，赋值到targetView。
            //这里是因为我们在记录的时候是根据Touch事件的event坐标去视图树里遍历取得点击的控件，这个控件不一定是响应点击事件的控件。
            //例如如果我们是对父容器添加了点击事件，touch事件刚好在其子控件是，通过我们的记录方案取得的是子控件，真正响应点击的是父容器。
            //这样看来，其实这一步也是可以在PrismMonitor中做，来确定真正可以响应点击的控件。
            //但这样做在RN上会有问题，RN上的点击响应事件流是在RN内部确定处理的，原生测只要上报点击MotionEvent信息，因此RN会将所有的原生控件设置为不可点击。
            //同理这样看来，回放功能在RN上应该不生效（？）
            if (!targetView.isClickable()) {
                targetView = PlaybackHelper.getClickableView(targetView);
            }
            final View tempView = targetView;
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    highLightTriggerView(tempView, new OnAnimatorListener() {
                        @Override
                        public void onAnimationEnd() {
                            //确定最终控件后触发Click事件。
                            MotionHelper.simulateClick(tempView);
                            mHandler.sendEmptyMessage(1);
                        }
                    });
                }
            }, 100);
        }
    } else if (eventInfo.eventType == PrismConstants.Event.ACTIVITY_START) {
        Toast.makeText(mContext, "页面跳转", Toast.LENGTH_SHORT).show();
        mHandler.sendEmptyMessage(1);
    } else if (eventInfo.eventType == PrismConstants.Event.BACK) {
        Toast.makeText(mContext, "返回", Toast.LENGTH_SHORT).show();
        //触发Back事件。
        MotionHelper.simulateBack();
        mHandler.sendEmptyMessage(1);
    } else if (eventInfo.eventType == PrismConstants.Event.DIALOG_SHOW) {
        mHandler.sendEmptyMessage(1);
    } else if (eventInfo.eventType == PrismConstants.Event.DIALOG_CLOSE) {
        mHandler.sendEmptyMessage(1);
    } else if (eventInfo.eventType == PrismConstants.Event.BACKGROUND) {
        Toast.makeText(mContext, "App退至后台", Toast.LENGTH_SHORT).show();
        mHandler.sendEmptyMessage(1);
    } else if (eventInfo.eventType == PrismConstants.Event.FOREGROUND) {
        Toast.makeText(mContext, "App进入前台", Toast.LENGTH_SHORT).show();
        mHandler.sendEmptyMessage(1);
    }
}
```

这里事件playback的代码，似乎并不是完整的，或者说应该有一个内部完整版本，这里会有一些问题：

我们在事件记录的时候，事件类型还有ACTIVITY_RESUME = 7;和ACTIVITY_PAUSE = 8;。当前的逻辑没有处理导致无法进行回放，可以做默认处理直接调用mHandler.sendEmptyMessage(1);。

另外对ACTIVITY_START事件，当前只是一个toast提示，因此无法进行页面跳转。demo里面是手动先跳转到对应页面然后开启回放，如果中间有做页面跳转也会没法实现。其实我们这完全可以通过mContext进行页面跳转，同时对于页面间的传参，也是可以在事件记录的时候（onStart生命周期函数）从intent中遍历取得。

包括其他一些事件的处理现在都只是Toast，这些我们都可以根据实际情况进行自己的实现。

#### 数据转化
我们记录的时候是EventData对象，回放的时候使用的是EventInfo对象，这里需要转化。

这里本身没有复杂逻辑，这里拿出来讲一下主要是因为在实际使用场景下，录制的数据更可能是以文件的形式保存的记录而不是EventData数组。

```java
public static EventInfo convertEventInfo(EventData eventData) {
    EventInfo eventInfo = new EventInfo();
    //取得UnionId，前面知道unionId是eventType+eventId。
    String unionId = eventData.getUnionId();
    eventInfo.originData = unionId;
    //根据"_\\^_"进行分割，是DIVIDER，是各个信息之间的风格符。
    //然后转化成Map结构。
    String[] result = unionId.split("_\\^_");
    HashMap<String, String> keysMap = new HashMap<>(result.length);
    for (int i = 0; i < result.length; i++) {
        if (TextUtils.isEmpty(result[i])) {
            continue;
        }
        int index = result[i].indexOf(PrismConstants.Symbol.DIVIDER_INNER);
        if (index == -1) {
            continue;
        }
        String key = result[i].substring(0, index);
        String value = result[i].substring(index + 3);
        keysMap.put(key, value);
    }
    //如果有“w”，说明有window信息，做解析确定windowType。
    if (keysMap.containsKey(PrismConstants.Symbol.WINDOW)) {
        String windowData = keysMap.get(PrismConstants.Symbol.WINDOW);
        String[] windowInfo = windowData.split(PrismConstants.Symbol.DIVIDER_INNER);
        if (windowInfo.length > 1) {
            eventInfo.windowType = Integer.parseInt(windowInfo[1]);
        } else {
            eventInfo.windowType = 1;
        }
    } else {
        eventInfo.windowType = 1;
    }

    if (!keysMap.containsKey("e")) {
        return null;
    }

    eventInfo.eventData = keysMap;
    //根据eventType设置不同的eventDesc。
    eventInfo.eventType = Integer.parseInt(keysMap.get("e"));
    switch (eventInfo.eventType) {
        case 0:
            eventInfo.eventDesc = "点击";
            break;
        case 1:
            eventInfo.eventDesc = "返回";
            break;
        case 2:
            eventInfo.eventDesc = "退至后台";
            break;
        case 3:
            eventInfo.eventDesc = "进入前台";
            break;
        case 4:
            eventInfo.eventDesc = "弹出弹窗";
            break;
        case 5:
            eventInfo.eventDesc = "弹窗关闭";
            break;
        case 6:
            eventInfo.eventDesc = "页面跳转";
            break;
    }
    return eventInfo;
}
```
本身没有复杂逻辑，但我们看到整个解析过程只用到了eventData.getUnionId()。因此如果我们要持久化记录埋点事件信息，其实只要记入这个unionId即可。

#### 点击回放
在前面我们了解到，对于点击回放主要就是确定targetView然后触发click。

对于确定targetView，首先是调用PlaybackHelper.findTargetView(prismWindow, eventInfo);获得，这是一个通过xpath寻找控件的过程。
```java
public static View findTargetView(PrismWindow prismWindow, EventInfo eventInfo) {
    //获取控件相关信息
    HashMap<String, String> eventData = eventInfo.eventData;
    View targetView = null;
    String windowData = eventData.get(PrismConstants.Symbol.WINDOW);
    String viewId = eventData.get(PrismConstants.Symbol.VIEW_ID);
    String viewPath = eventData.get(PrismConstants.Symbol.VIEW_PATH);
    String viewList = eventData.get(PrismConstants.Symbol.VIEW_LIST);
    String viewReference = eventData.get(PrismConstants.Symbol.VIEW_REFERENCE);
    //通过比对Window的信息（windowTitle和windowType）来确定当前视图是否正确。
    if (validateWindow(prismWindow, windowData)) {
        if (viewPath != null) {
            if (viewList != null) {
                //如果存在viewList（listInfo），说明是在列表容器（ViewPage、RecyclerView、AbsListView）中，取得其直属第一个列表容器。
                //findTargetViewContainer函数我们后面具体讲一下。
                ViewGroup container = findTargetViewContainer(prismWindow.getDecorView(), viewPath, viewList);
                if (container != null) {
                    //获取在该列表容器中的信息。（r:position,index）
                    String listData = viewList.split(",")[0];
                    //获取相对于container的Path。
                    String relativePath = viewPath.substring(0, viewPath.indexOf("*"));
                    //取得position
                    int realItemPosition = Integer.parseInt(listData.split(":")[1]);
                    // 根据position确定是否需要滚动
                    if (needScroll(container, realItemPosition)) {
                        //进行滚动然后返回null。
                        //这里单纯从逻辑上看会有点诧异，但结合实际来看，触发滚动之后需要等到它滚动到目标位置后才能进行后续操作。
                        //因此这里返回null其实是等待它滚动完成的操作，基于重试机制下次进入的时候可能已经滚动完成了。
                        smoothScrollToPosition(container, realItemPosition);
                        return null;
                    } else {
                        // 是否存在点击的控件是存在viewId（xml中的@+id/xxx，我们也会叫它控件名），
                        if (viewId != null) {
                            //首先获取ItemView
                            View itemView = findItemViewByPosition(container, realItemPosition);
                            if (itemView != null) {
                                //根据ID获取对于的targetView，当内部无法根据viewId精确匹配到控件时，会根据viewReference去寻找最有可能的控件进行返回。
                                targetView = findTargetViewById(itemView, viewId, relativePath, viewReference);
                                if (targetView != null) {
                                    //如果targetView不在可视区域内，滚动到可视区域。这里因为targetView已经确定了，就不需要返回null了。
                                    smoothScrollToVisible(container, targetView);
                                    return targetView;
                                }
                            }

                            //例如RecyclerView，其显示是有数据撑起来的，当数据变化时itemView会变化，如果列表里面有不同的itemType，在原来的ItemView中可能无法找到对于的控件。
                            //其实这里就算找到了，也可能是其他数据的ItemView中的对应控件了。
                            //因此这里会尝试在container内所有的child中去获取一遍。
                            for (int i = 0; i < container.getChildCount(); i++) {
                                itemView = container.getChildAt(i);
                                targetView = findTargetViewById(itemView, viewId, relativePath, viewReference);
                                if (targetView != null) {
                                    smoothScrollToVisible(container, targetView);
                                    return targetView;
                                }
                            }
                            // 如果无法通过viewId查找到view，退出。
                            // 这里可以思考一下，其实这种方式也是无法完全精确的找到这个控件的。
                            // 刚说的，数据变化是，尝试在整个列表中去，但这是只会遍历所有当前的child，但由于存在复用机制，其实是没办法遍历到所有的item的。
                            // 再次会通过viewReference去判断，前面我们知道viewReference其实是左上角TetView控件的文案。在列表情况下还是很有可能命中的。
                            return null;
                        }

                        // 如果不存在ViewId，则尝试通过relativePath查找，这里就是通过记录的index或者viewId一级一级的获取child。
                        targetView = findTargetViewByPath(container, relativePath);
                        if (targetView != null) {
                            smoothScrollToVisible(container, targetView);
                            return targetView;
                        } else {
                            return null; // 无法通过viewPath查找到view，退出
                        }
                    }
                } else {
                    return null; // 无法查找到container，退出
                }
            }
            //如果不在列表容器中，存在viewId进行获取。
            if (viewId != null) { // 存在viewId
                targetView = findTargetViewById(prismWindow.getDecorView(), viewId, viewPath, viewReference);
                if (targetView != null) {
                    return needScrollOrNot(targetView);
                }
            }
            //否则通过viewPath获取。
            targetView = findTargetViewByPath(prismWindow.getDecorView(), viewPath);
            if (targetView != null) {
                return needScrollOrNot(targetView);
            }
            //都没获取到尝试通过viewReference进行模糊获取。
            if (!TextUtils.isEmpty(viewReference)) {
                targetView = findTargetViewByReference(prismWindow.getDecorView(), viewReference);
                if (targetView != null) {
                    return needScrollOrNot(targetView);
                }
            }

        }

    }
    return targetView;
}
//对于列表控件，获取最近的列表容器，可能会找不到。
private static ViewGroup findTargetViewContainer(ViewGroup viewGroup, String viewPath, String viewList) {
    String[] viewPaths = viewPath.split("/");
    String[] viewLists = viewList.split(",");
    //这里是应为其格式是“r:position,index,r:position,index”，通过“,”分割后其个数要/2。
    int listIndex = viewLists.length / 2;
    int pathIndex = viewPaths.length - 1;
    ViewGroup container = viewGroup;
    //遍历
    while (pathIndex >= 0) {
        //碰到node是*，说明是在列表容器，在viewList中取出node信息。
        String node = viewPaths[pathIndex];
        if (node.equals("*")) {
            listIndex--;
            //listIndex == 0代表寻找已经是和目标控件最近的位置了。
            if (listIndex == 0) {
                break;
            } else {
                node = viewLists[listIndex * 2 + 1];
            }
        }
        try {
            //如果node是int类型，说明是index，通过getChildAt取。
            int position = Integer.parseInt(node);
            if (position < container.getChildCount()) {
                View childView = container.getChildAt(position);
                if (childView instanceof ViewGroup) {
                    container = (ViewGroup) childView;
                } else {
                    //findPossibleContainerView其实就是返回viewGroup子控件中的列表控件，这里在同一父容器下寻找可能的容器。
                    //如果viewGroup本身是列表控件就直接返回。
                    return findPossibleContainerView(viewGroup);
                }
            } else {
                return findPossibleContainerView(viewGroup);
            }
        } catch (Exception e) {
            //如果不是int类型，可能是ViewId，尝试获取。
            View view = findTargetViewById(container, node, null, null);
            if (view == null) {
                return findPossibleContainerView(viewGroup);
            } else {
                if (view instanceof ViewGroup) {
                    container = (ViewGroup) view;
                } else {
                    return findPossibleContainerView(viewGroup);
                }
            }
        }
        pathIndex--;
    }

    if (container instanceof RecyclerView || container instanceof AbsListView || container instanceof ViewPager) {
        return container;
    } else {
        return findPossibleContainerView(viewGroup);
    }
}
```

然后要确定真正响应点击事件的控件，PlaybackHelper.getClickableView(targetView);，这里就是向上（父容器）寻找的过程，十分简单，这里就不展开讲了。

最后是触发点击，这里通过模拟用户点击实现。
```java
public static void simulateClick(View view) {
    //获取控件中心点座庙
    int[] outLocation = new int[2];
    view.getLocationOnScreen(outLocation);
    float x = outLocation[0] + view.getWidth() / 2;
    float y = outLocation[1] + view.getHeight() / 2;
    long downTime = SystemClock.uptimeMillis();
    //创建down事件和up事件，控件中心点+100毫秒间隔，转入onTouchEvent函数，模拟用户点击操作。
    final MotionEvent downEvent = MotionEvent.obtain(downTime, downTime, MotionEvent.ACTION_DOWN, x, y, 0);
    final MotionEvent upEvent = MotionEvent.obtain(downTime + 100, downTime + 100, MotionEvent.ACTION_UP, x, y, 0);
    view.onTouchEvent(downEvent);
    view.onTouchEvent(upEvent);
    downEvent.recycle();
    upEvent.recycle();
}
```

#### back事件触发

最后提一下back事件触发。
```java
public static void simulateBack() {
    execute(new Runnable() {
        @Override
        public void run() {
            try {
                Instrumentation inst = new Instrumentation();
                inst.sendKeyDownUpSync(KeyEvent.KEYCODE_BACK);
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
    });
}
```
代码十分简单，但这里用到了一个功能十分强大的类Instrumentation。
说它强大是因为可以用来模拟用户操作，像微信抢红包助手，自动化框架Appium，辅助功能无障碍模式等都会有用到它。有兴趣的可以去详细了解。

