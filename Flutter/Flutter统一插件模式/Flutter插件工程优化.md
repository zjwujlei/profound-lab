Flutter插件工程优化 - FlutterInAction分享稿
=================

一次开发、一个配置文件、一行命令，完成一个插件的完整接入。

### 插件工程拆分

对于Flutter提供的插件实现，对于每一个功能单独封装插件工程，会导致我们的工程急剧膨胀。在淘系的分享中将这种做法称为普通的实现方式。相对的就会有高级的实现，对于高级实现，我们将所有功能插件都聚合到一个插件工程中。这种做法避免了工程的膨胀，当这种聚合的插件工程失去了插件的公用性，这样的一个聚合插件工程没法在多APP之间公用，每个APP的需要的插件基本是不一致的。同时淘系的分享中也提出了他们的‘艺术程序员’的实现方式，使用FFI来提升性能，通过Code_gen来生成功能接口，对于FFI的实现和功能的具体实现再进行开发。

结合淘系的分享与我们的实际情况，我们决定采用聚合插件的方式，通是通过全面代码生成的方式来保证其公共性。在对于一个插件工程的代码我们从上到下依次划分为：1.桥接层、2.胶水层、3.原生功能层三部分。然后对官方的Pigeon方案进行扩展，我们将三部分的代码统一到一个dart文件中，然后通过一个命令来完成Channel构建、功能实现、接入的工作，大大减小了插件使用的复杂度。

###### 桥接层
Flutter和原生的通信实现，如果用官方的MethodChannel，就是Channel的原生及Flutter实现。

对于这一层，我们可以使用官方的Pigeon进行生成。详见<a href="https://mp.weixin.qq.com/s/E24bY7nt2HL0Pl-vEkECXghttps://mp.weixin.qq.com/s/E24bY7nt2HL0Pl-vEkECXg">Pigeon- Flutter多端接口一致性以及规范化管理实践</a>

通过Pigeon，我们生成了Channel的实现、能力使用方的调用对象、能力提供方的的接口定义。Pigeon主要是用来保证接口的一致性，避免的手动开发接口导致的出错等，但对于一个具体的插件，其功能还是需要进行开发的。

因此我们扩展了Pigeon的功能。将对功能接口的实现代码也在dart脚本中进行声明，并扩展出根据代码声明还原功能实现的能力。

###### 胶水层
我们的Flutter启动较晚，我们都会将一个功能的现有实现库封装成Flutter插件。当由于我们Flutter的推进进度较缓慢，我们不想因为Flutter上引用的简便性而去修改原生库。因此我们这里引入胶水层来抹平这部分差异。

我们也将所有胶水层的代码都定义在Dart脚本中，扩展Pigeon使其生成对应代码。

###### 原生功能库

我们的原生库都是以gradle/podfile的形式引入的。我们对于每个插件的配置单独作为一个gradle/podfile配置，也将这分定义放在dart文件配置文件中。

但对于库的应用有所不同，我们除了还原出gradle/podfile配置外还需要将其添加到统一插件中，对于 Android我们通过‘apply from:'./{plugin}.gradle'’的方式引入，iOS通过 @许益成 的 方式引入，我们只需要修改统一插件中的gradle/podfile文件添加这一行代码即可，这个也通过扩展Pigeon的功能。



###### 总结

![ower_plugin](./ower_plugin_2.png)

我们整一个架构入上。我们有一个plugin config repo的库。用来开发我们想要的插件，这本身时候一个聚合插件工程，我们在这个聚合插件工程中维护dart配置文件。

但我们需要在某个工程里引入一个插件功能的时候，只需要在改聚合工程下执行“flutter pub run flagon --input=./flagons/geetest.dart”命令即可完成接入。

### 附录一

极验插件的dart配置文件

```dart
import 'package:flagon/ast.dart';
import 'package:flagon/flagon_lib.dart';


@ApiGlue(platform: 'Android',fileName: 'WrapperToolUtil.java')
const WRAPPER_TOOL_UTIL = '''package com.wedoctor.guahao.wrapper_geetest;

import android.content.Context;
import android.content.SharedPreferences;

import com.greenline.wyapptrackmodule.EventManager;
import com.guahao.android.utils.WYAppConfigUtils;
import com.guahao.android.utils.WYStringUtils;

import java.util.regex.Pattern;

public class WrapperToolUtil {
    private static final String TAG = "ToolUtil";

    public static boolean isAllZero(String str){
        if (WYStringUtils.isNotNull(str)) {
            Pattern pattern = Pattern.compile("[0]*");
            return pattern.matcher(str).matches();
        } else {
            return false;
        }
    }

    public static String getDeviceId(Context context) {
        if (context == null){
            return "";
        }
        SharedPreferences phoneInfo = context.getSharedPreferences("phoneInfo", 0);
        String deviceId = phoneInfo.getString("deviceId", null);
        if (deviceId == null || "".equals(deviceId) || isAllZero(deviceId)) {
            deviceId = WYAppConfigUtils.getDeviceId(context);

            if (deviceId == null || "".equals(deviceId) || isAllZero(deviceId)) {
                //uuid替代deviceid
                deviceId =  EventManager.getUUID(context);
            }
            phoneInfo.edit().putString("deviceId", deviceId).commit();
        }
        return deviceId;
    }
}

''';

@ApiGlue(platform: 'Android',fileName: 'WrapperProcessResponse.java')
const WRAPPER_PROCESS_RESPONSE = '''package com.wedoctor.guahao.wrapper_geetest;

import com.greenline.guahao.common.server.okhttp.JSONResponse;

import org.json.JSONObject;

public class WrapperProcessResponse extends JSONResponse {
    public String serialNumber;
    //是否使用极验证 0：否 1：是
    public boolean isPolarVerification;


    public WrapperProcessResponse(JSONObject object) throws Exception {
        super(object);
        parse(object);
    }

    private void parse(JSONObject obj) throws Exception {
        JSONObject jsonObject = obj.optJSONObject("item");
        serialNumber = jsonObject.optString("serialNumber");
        isPolarVerification = jsonObject.optInt("polarVerification") == 1 ? true : false;
    }
}
''';

@ApiGlue(platform: 'Android',fileName: 'WrapperProcessRequest.java')
const WRAPPER_PROCESS_REQUEST = '''package com.wedoctor.guahao.wrapper_geetest;

import com.greenline.guahao.common.server.okhttp.SimplifyJSONRequest;

import org.json.JSONException;
import org.json.JSONObject;

public class WrapperProcessRequest extends SimplifyJSONRequest<WrapperProcessResponse> {
    //客户端sessionId
    private String mSessionId;
    //业务类型
    private int mBusinessType;
    //账户
    private String mAccountName;

    private int mUserType;

    public WrapperProcessRequest(String sessionId, int businessType, String accountName, int userType) {
        this.mSessionId = sessionId;
        this.mBusinessType = businessType;
        this.mAccountName = accountName;
        this.mUserType = userType;
    }

    @Override
    protected String url() {
        return WrapperGeetestPlugin.PRE_PROCESS_CHECK;
    }

    @Override
    protected String body() throws JSONException {
        JSONObject obj = new JSONObject();
        obj.put("sessionId", mSessionId);
        obj.put("businessType", mBusinessType);
        obj.put("accountName", mAccountName);
        obj.put("userType", mUserType);
        return obj.toString();
    }

    @Override
    protected WrapperProcessResponse result(JSONObject json) throws Exception {
        return new WrapperProcessResponse(json);
    }
}
''';

@ApiGlue(platform: 'Android',fileName: 'WrapperGeeTestHelper.java')
const WRAPPER_GEETEST_HELPER = '''package com.wedoctor.guahao.wrapper_geetest;

import android.content.Context;
import android.util.Log;

import com.geetest.sdk.GT3ConfigBean;
import com.geetest.sdk.GT3ErrorBean;
import com.geetest.sdk.GT3GeetestUtils;
import com.geetest.sdk.GT3Listener;
import com.guahao.android.utils.WYLogUtils;

import org.json.JSONException;
import org.json.JSONObject;

public class WrapperGeeTestHelper {
    private static final String TAG = "WrapperGeeTestHelper";
    private GT3GeetestUtils mGT3GeetestUtils;
    private OnFinishedSDKCallback mCallback;
    private String mSid;
    private GT3ConfigBean mGT3ConfigBean;
    private Context mContext;

    private WrapperGeeTestHelper(Context context, String sid, String challenge, OnFinishedSDKCallback callback) {
        this.mSid = sid;
        this.mCallback = callback;
        this.mContext = context;
        mGT3GeetestUtils = new GT3GeetestUtils(context);
        mGT3ConfigBean = new GT3ConfigBean();
        initGT3ConfigBean();
        mGT3GeetestUtils.init(mGT3ConfigBean);
        // 开启验证
        mGT3GeetestUtils.startCustomFlow();
        try {
            JSONObject jsonObject = new JSONObject(challenge);
            mGT3ConfigBean.setApi1Json(jsonObject);
        } catch (JSONException e) {
            WYLogUtils.e(TAG, "WrapperGeeTestHelper----------: " + e.getMessage(), e);
        }
    }

    private void initGT3ConfigBean() {
        // 设置验证模式，1：bind；2：unbind
        mGT3ConfigBean.setPattern(1);
        // 设置点击灰色区域是否消失，默认不消失
        mGT3ConfigBean.setCanceledOnTouchOutside(false);
        // 设置语言，如果为null则使用系统默认语言
        mGT3ConfigBean.setLang(null);
        // 设置加载webview超时时间，单位毫秒，默认10000。仅且webview加载静态文件超时，不包括之前的http请求
        mGT3ConfigBean.setTimeout(10000);
        // 设置webview请求超时（，前端请求后端用户点选或滑动完成接口），单位毫秒，默认10000
        mGT3ConfigBean.setWebviewTimeout(10000);
        mGT3ConfigBean.setListener(mGT3Listener);
    }

    private GT3Listener mGT3Listener = new GT3Listener() {
        /**
         * 验证加载完成
         * @param duration 加载时间和版本等信息，为json格式
         */
        @Override
        public void onDialogReady(String duration) {
            super.onDialogReady(duration);
            WYLogUtils.d(TAG, "onDialogReady: -------:" + duration);
        }

        /**
         * 验证结果
         * @param result
         */
        @Override
        public void onDialogResult(String result) {
            WYLogUtils.d(TAG, "onDialogResult: -------:" + result);
            try {
                JSONObject res_json = new JSONObject(result);
                String challenge = res_json.getString("geetest_challenge");
                String validate = res_json.getString("geetest_validate");
                String seccode = res_json.getString("geetest_seccode");
                res_json.put("sessionId", mSid)
                        .put("challenge", challenge)
                        .put("validate", validate)
                        .put("seccode", seccode);
                mGT3GeetestUtils.showSuccessDialog();
                if (mCallback != null) {
                    mCallback.onSuccess(res_json);
                }
            } catch (JSONException e) {
                mGT3GeetestUtils.showFailedDialog();
                mCallback.onFailure();
            }
        }

        /**
         * 统计信息，参考接入文档
         * @param result
         */
        @Override
        public void onStatistics(String result) {
            WYLogUtils.d(TAG, "onStatistics: -------:" + result);
        }

        /**
         * 验证被关闭
         * @param num
         */
        @Override
        public void onClosed(int num) {
            WYLogUtils.d(TAG, "onClosed: ------:" + num);
        }

        /**
         * 验证成功回调
         * @param s
         */
        @Override
        public void onSuccess(String s) {
            WYLogUtils.d(TAG, "onSuccess: -------:" + s);
            mGT3GeetestUtils.showSuccessDialog();
        }

        /**
         * 验证失败回调
         * @param gt3ErrorBean 版本号、错误码、错误描述等信息
         */
        @Override
        public void onFailed(GT3ErrorBean gt3ErrorBean) {
            WYLogUtils.d(TAG, "onFailed: --------:" + gt3ErrorBean.toString());
            mGT3GeetestUtils.showFailedDialog();
        }

        /**
         * api1回调
         */
        @Override
        public void onButtonClick() {
            WYLogUtils.d(TAG, "onButtonClick: -------");
        }
    };

    /**
     * 执行极验
     */
    public void launchGeetest() {
        mGT3GeetestUtils.getGeetest();
    }

    /**
     * 极验SDK内部完成验证后，将相关数据传出。
     */
    public interface OnFinishedSDKCallback {
        void onSuccess(JSONObject jsonObject);

        void onFailure();
    }

    public static class Builder {
        private Context mContext;
        private String mSid;
        private String mChallenge;
        private OnFinishedSDKCallback mCallback;

        public Builder setContext(Context context) {
            this.mContext = context;
            return this;
        }

        public Builder setSid(String sid) {
            this.mSid = sid;
            return this;
        }

        public Builder setChallenge(String Challenge) {
            this.mChallenge = Challenge;
            return this;
        }

        public Builder setOnFinishedSDKCallback(OnFinishedSDKCallback callback) {
            this.mCallback = callback;
            return this;
        }

        public WrapperGeeTestHelper build() {
            if (mContext == null) {
                throw new IllegalArgumentException("context can not be null");
            }
            return new WrapperGeeTestHelper(mContext, mSid, mChallenge, mCallback);
        }
    }
}
''';

const LAUNCH_GEETEST_CODE = '''final Activity activity;
            if (mActivity == null || (activity = mActivity.get()) == null) {
                WYLogUtils.w(TAG, "onMethodCall: launchGeetest[activity已销毁]");
                return;
            }

            final String sessionId = WrapperToolUtil.getDeviceId(activity) + SystemClock.currentThreadTimeMillis();
            int businessType = call.argument("businessType");
            String accountName = call.argument("accountName");
            int userType = call.argument("userType");

            new WrapperProcessRequest(sessionId, businessType, accountName, userType).schedule(new SimplifyJSONListener<WrapperProcessResponse>() {
                @Override
                public void onSuccess(WrapperProcessResponse processResponse) {
                    if (processResponse.isPolarVerification) {
                        launchGeetest(activity, sessionId, processResponse.serialNumber, result);
                    } else {
                        Map<String, Object> data = new HashMap<>();
                        data.put("polarVerification", 0);
                        result.success(data);
                    }
                }

                @Override
                public void onFailed(Throwable e) {
                    result.error("-1", "网络异常，请求失败", "网络异常，请求失败");
                }
            });
''';

class GeetestRequest{
  int businessType;
  String accountName;
  int userType = 0;
}

class GeetestResponse{
  int result;
}

@DependenceLibrary()
class Dependencies{
  static String androidDependencies = '''android {

    dependencies {
        //极验证
        compileOnly 'com.geetest.sensebot:sensebot:4.2.1'

        implementation "com.guahao.android:net-client:1.6.4.9-SNAPSHOT"
        implementation 'com.squareup.okhttp3:okhttp:3.11.0'
        implementation 'io.reactivex.rxjava2:rxandroid:2.0.1'
        implementation 'io.reactivex.rxjava2:rxjava:2.0.1'
        implementation 'com.guahao.android:wyapptrack:0.2.0-SNAPSHOT'
    }
} 
''';
  static String iOSDependencies = '''
''';
}

@ImplImport(platform: 'Android',statement: 'import io.flutter.embedding.engine.FlutterEngine;')
@HostApi()
abstract class GeeTest{
  @ApiImpl(code: LAUNCH_GEETEST_CODE,platform: 'Android')
  GeetestResponse launchGeetest(GeetestRequest request);

}

void configureFlagon(FlagonOptions opts) {
  opts.dartOut = './lib/geetest.dart';
  opts.objcHeaderOut = 'ios/Classes/GeeTest.h';
  opts.objcSourceOut = 'ios/Classes/Geetest.m';
  opts.objcOptions.prefix = 'FLT';
  opts.javaOut =
  'android/src/main/java/com/profound/geetest/Geetest.java';
  opts.javaOptions.package = 'com.profound.geetest';
  opts.gradleOut='android/geetest.gradle';
}

```