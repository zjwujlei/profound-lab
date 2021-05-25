Flutter编译Engine
=================

下载编译过程网络上都有博客介绍，主要用到了gclient命令，在<a href="https://chromium.googlesource.com/chromium/tools/depot_tools.git">depot_tools</a>中。

* fork engine仓库，然后在我们本机指定的engine目录创建.gclient文件，内容为：
```
solutions = [
  {
    "managed": False,
    "name": "src/flutter",
    "url": "git@github.com:<your_name_here>/engine.git",
    "custom_deps": {},
    "deps_file": "DEPS",
    "safesync_url": "",
  },
]
```

* 执行gclient sync命令

* 进入src/flutter目录，配置upstream并拉取最新代码：
```
git remote add upstream git@github.com:flutter/engine.git
git pull upstream master
```

* 将代码切到我们使用的engine版本，可以在SDK的/flutter/bin/internal/engine.version文件下查看，这个版本就是commitId，我们回滚到这个节点。
```
git reset --hard ${engine.version} 
```

* 进行同步
```
gclient sync --with_branch_heads --with_tags
```

* 对于Android平台进行编译，这里有很多其他参数，例如指定个cpu类型，java版本等。

```
$ ./flutter/tools/gn --unoptimized
$ ./flutter/tools/gn --android --unoptimized
$ ninja -C out/android_debug_unopt && ninja -C out/host_debug_unopt
```

这里主要记录下编译过程中碰到的问题.

### 代理问题
科学上网后，浏览器可以打开https://chrome-infra-packages.appspot.com/，但gclint sync命令时一直出错：
```
Failed to connect to chrome-infra-packages.appspot.com port 443: Operation timed out
```

gclient其实可以认为是一个代码管理工具，内部都是svn，git操作。和我们平时科学上网下git clone时一样，需要设置代理，参考网络上的资料可以在环境变量文件里添加如下函数，需要是只要执行下proxy_on即可。
```
#注意，根据自己的配置设置有可能会是1080或1086
function proxy_on() {
        export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"
        export http_proxy="http://127.0.0.1:1087"
        export https_proxy=$http_proxy
        export ftp_proxy=$http_proxy
        export rsync_proxy=$http_proxy
        export HTTP_PROXY=$http_proxy
        export HTTPS_PROXY=$http_proxy
        export FTP_PROXY=$http_proxy
        export RSYNC_PROXY=$http_proxy
        echo -e "已开启代理"
}
```

###gclient sync --with_branch_heads --with_tags命令出错
```
Error: Command 'git remote update' returned non-zero exit status 1 in /Users/${xxx}/engine/src/third_party/dart/third_party/pkg/bazel_worker
Fetching origin
```


```
fatal: unable to access 'https://dart.googlesource.com/bazel_worker.git/': LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to dart.googlesource.com:443
error: Could not fetch origin
```

都是git fetch失败，这个很明显是网络问题，使用科学上网+代理。

### ./flutter/tools/gn --android --unoptimized Mac平台下xcode sdk版本问题

```
ERROR at //build/config/mac/mac_sdk.gni:27:5: Script returned non-zero exit code.
    exec_script("//build/mac/find_sdk.py", find_sdk_args, "list lines")
    ^----------
Current dir: /Users/xxx/github-space/engine/src/out/android_debug_unopt/
Command: python /Users/xxx/github-space/engine/src/build/mac/find_sdk.py --print_sdk_path 10.12
Returned 1.
stderr:

Traceback (most recent call last):
  File "/Users/xxx/github-space/engine/src/build/mac/find_sdk.py", line 90, in <module>
    print main()
  File "/Users/xxx/github-space/engine/src/build/mac/find_sdk.py", line 63, in main
    raise Exception('No %s+ SDK found' % min_sdk_version)
Exception: No 10.12+ SDK found
```
 
根据日志我们看下find_sdk.py的main函数
```
def main():
  
    //通过xcode-select命令获取Xcode.ap目录，正常是/Applications/Xcode.app/Contents/Developer
  job = subprocess.Popen(['xcode-select', '-print-path'],
                         stdout=subprocess.PIPE,
                         stderr=subprocess.STDOUT)
  out, err = job.communicate()
  if job.returncode != 0:
    print >> sys.stderr, out
    print >> sys.stderr, err
    raise Exception(('Error %d running xcode-select, you might have to run '
      '|sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer| '
      'if you are using Xcode 4.') % job.returncode)

    //SDK路径拼接，/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs
  # The Developer folder moved in Xcode 4.3.
  xcode43_sdk_path = os.path.join(
      out.rstrip(), 'Platforms/MacOSX.platform/Developer/SDKs')
  if os.path.isdir(xcode43_sdk_path):
    sdk_dir = xcode43_sdk_path
  else:
    sdk_dir = os.path.join(out.rstrip(), 'SDKs')

    //通过文件夹名称获取SDK版本
  sdks = [re.findall('^MacOSX(10\.\d+)\.sdk$', s) for s in os.listdir(sdk_dir)]
  sdks = [s[0] for s in sdks if s]  # [['10.5'], ['10.6']] => ['10.5', '10.6']
  sdks = [s for s in sdks  # ['10.5', '10.6'] => ['10.6']
          if parse_version(s) >= parse_version(min_sdk_version)]
  if not sdks:
    raise Exception('No %s+ SDK found' % min_sdk_version)
  best_sdk = sorted(sdks, key=parse_version)[0]

  return best_sdk

```

这里去掉了部分非重点逻辑，可以看到通过'^MacOSX(10\.\d+)\.sdk$'筛选，由于电脑系统版本升级到了11.1，在对应目录下只有MacOSX11.1.sdk目录，因此无法取得best_sdk。

这里的MacOSX版本我们没法简单的更改，可以直接修改这个函数使用11.1版本，也可以使用更新的engine版本，这里engine使用的是ee76268252c22f5c11e82a7b87423ca3982e51a7对应flutter-1.17-candidate.3。切换到更新版本(2.3.0)解决这个问题。

### ./flutter/tools/gn --android --unoptimized ninja版本问题
ninja是一款编译工具，具有快速的优点。
```
ninja: fatal: ninja version (1.5.3) incompatible with build file ninja_required_version version (1.7.2).
Failed to run ninja:  1
```

通过which ninja查看发现单前使用的是Android sdk中cmake下的nanja工具，是由于之前设置过cmake的环境变量，实际上我们安装的depot_tools中就包含ninja工具，这里使用depot_tools中ninja即可。可以通过设置临时环境变量，将depot_tools路径设置在最前面。

### 产物及使用

这里编译的是Android arm（默认就是arm，可以通过--android-cpu arm 指定）下的debug产物，包含：

* android_debug_unopt 目录
* host_debug_unopt 目录
* compile_commands.json 文件

我们使用的flutter.jar和libflutter.so就在android_debug_unopt中。

我们可以通过一下命令来直接使用
```
flutter run --local-engine-src-path=/xxx/engine/src --local-engine=/xxx/engine/src/out/android_debug_unopt
```

但如果我们想要将本地engine配合Android Studio使用，或者使用Android gradle相关命令的话，可以直接使用本地engine产物。这个在如果看过flutter sdk源码的话会有印象，我们打包Flutter APK的时候有时会碰到Flutter engine依赖下载失败，因此我们会更改MAVEN_REPO，或者使用私有maven仓库的方式解决。这部分在SDK的flutter.gradle文件中，文件路径：/flutter/packages/flutter_tools/gradle/flutter.gradle。这里有使用本地的engine的逻辑，有以下三部分：

```
#105行，local-engine-repo的配置，engine产物maven依赖目录。
String hostedRepository = System.env.FLUTTER_STORAGE_BASE_URL ?: DEFAULT_MAVEN_HOST
String repository = useLocalEngine()
    ? project.property('local-engine-repo')
    : "$hostedRepository/download.flutter.io"

project.rootProject.allprojects {
    repositories {
        maven {
            url repository
        }
    }
}
#205行，local-engine-out配置，编译的engine out目录
if (useLocalEngine()) {
    // This is required to pass the local engine to flutter build aot.
    String engineOutPath = project.property('local-engine-out')
    println("engineOutPath:"+engineOutPath)
    File engineOut = project.file(engineOutPath)
    if (!engineOut.isDirectory()) {
        throw new GradleException('local-engine-out must point to a local engine build')
    }
    localEngine = engineOut.name
    localEngineSrcPath = engineOut.parentFile.parent
}
#539行，local-engine-build-mode配置，配置debug、release、profile。
private Boolean supportsBuildMode(String flutterBuildMode) {

    if (!useLocalEngine()) {
        return true;
    }
    assert project.hasProperty('local-engine-build-mode')
    // Don't configure dependencies for a build mode that the local engine
    // doesn't support.
    return project.property('local-engine-build-mode') == flutterBuildMode
}
```

需要在gradle.properties中配置local-engine-repo、local-engine-out和local-engine-build-mode。对于配置很简单：

```
local-engine-out=/xxx/engine/src/out/android_debug_unopt
local-engine-build-mode=debug
```

local-engine-repo就不是一个engine编译结果中直接的目录了，我们可以在flutter.gradle中对local-engine-repo进行打印，在flutter run命令运行是输出的结果如下：
```
/var/folders/00/z4yrqp2150sdfhgk9h3b7l4r0000gn/T/flutter_tools.U3WOrb/flutter_tool_local_engine_repo.hYKR2x

```

这就是一个本地maven仓库，在flutter run命令时会动态生成。这里简单起见我们就将这个maven仓库直接拷贝到/xxx/engine/src/out/目录下进行使用。平台化engine编译打包的话，可以在编译生成这个engine后，使用shell或者gradle等脚本去输出maven依赖。

最后附上gradle.properties下的三项完整配置
```
local-engine-repo=/xxx/engine/src/out/android_debug_repo
local-engine-out=/xxx/engine/src/out/android_debug_unopt
local-engine-build-mode=debug
```
