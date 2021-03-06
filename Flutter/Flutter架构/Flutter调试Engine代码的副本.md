Flutter调试Engine代码
====================

Flutter Engine部分代码较多，切网上资料较少，学习切入点不好寻找。这里通过调试作为切入点进行学习。

### lldb

<a href="https://lldb.llvm.org/use/remote.html">lldb</a>是一个代码调试工具，是一个底层调试器(low level debugger)Android中NDK下开发调试就是基于lldb。lldb是基于Client-Server架构，Client和Server通过gdb-remote协议进行通信，基于TCP/IP进行传输。

#### lldb-server

对于server，在Android平台下，手机端并未自带lldb-server，我们需要push server端到手机中。lldb-server是一个二进制可执行文件，既然Android中NDK下开发调试使用lldb，且Android平台本身不带，那大概率是在Android开发环境下的，在我们进行Native调试的时候push到手机中的。确实，lldb-server在SDK下（/sdk/ndk/21.2.6472646/toolchains/llvm/prebuilt/darwin-x86_64/lib64/clang/9.0.8/lib/linux/aarch64），push到手机目录/data/local/tmp下，如果不想手动寻找push，其实可以创建一个NDK项目对Native代码调试下就有了。

在/data/local/tmp，发现了lldb-server和start_lldb_server.sh，可以直接使用start_lldb_server.sh，这个sh文件就是对lldb-server的一个调用封装。

```
#进入设备 shell
adb shell 
#进入对应APP空间，必须是debug
run-as com.profound.flutter_app
#拷贝lldb-server和start_lldb_server.sh
cp /data/local/tmp/lldb-server ./lldb/bin
cp /data/local/tmp/start_lldb_server.sh ./lldb/bin
#启动服务
sh ./lldb/bin/start_lldb_server.sh /data/data/com.profound.flutter_app/lldb unix-abstract /com.profound.flutter_app debug.sock "lldb process:gdb-remote packets"
```

这样服务就启动了。对于start_lldb_server.sh的参数含义如下:
```
LLDB_DIR=$1
LISTENER_SCHEME=$2
DOMAINSOCKET_DIR=$3
PLATFORM_SOCKET=$4
LOG_CHANNELS=$5

BIN_DIR=$LLDB_DIR/bin
LOG_DIR=$LLDB_DIR/log
TMP_DIR=$LLDB_DIR/tmp
PLATFORM_LOG_FILE=$LOG_DIR/platform.log
```

#### lldb

对于Client端，使用lldb命令。基本机器上就有lldb命令，在/usr/bin下。我们直接使用lldb命令开启lldb client进程。
```
~ lldb
(lldb) 
```

就和python等一样会进入lldb进程，我们在这里连接lldb-server服务器。
```
#选择remote-android，可以通过platform list列出所有的platform，如果不选择默认是host。执行成功会输出Platform和Connected状态（NO）
platform select remote-android
#连接服务器，和server创建是的配置对应。执行成功会输出配置信息，包括Connected: yes
platform connect unix-abstract-connect:///com.profound.flutter_app/debug.sock
#绑定进程，执行成功会输出进程信息。
process attach -p 24836
#绑定动态库符号，编译engine时使用的是debug，这里直接关联打包产物下的libflutter.so。执行成功会输出将本机和Android设备上libflutter.so的绑定关系。
add-dsym /xxx/engine/src/out/android_debug_unopt/libflutter.so
#关联源码库和符号目录。
settings set target.source-map /xxx/engine/src/out/android_debug_unopt /Users/wujinglei/github-space/engine/src/
```

至此，lldb client端完成，可以通过命令行进行调试。lldb的调试语法可以参见<a href="https://lldb.llvm.org/use/map.html#id2">Breakpoint Commands</a>

对于这一部分，其实在engine中已经提供了sh脚本来简便操作，

### 使用clion进行调试 
使用命令行调试毕竟不符合我们的开发习惯。对于cpp，习惯使用clion进行开发。clion（Clion 2020.1.1）本身并没有直接提供lldb remote debug的配置，倒是提供了gdb的配置。在Clion的plugin market上找到了一款lldb的插件实测也无法使用，这下就尴尬了。

于是乎如果要使用clion进行调试，似乎只剩下了使用gdb一条路了。庆幸的是，对于gdb，flutter官方已经封装了一python脚本提供我们使用，在如下目录：/engine/src/flutter/sky/tools/flutter_gdb。（特别的需要使用src目录下的flutter_gdb，engine下我们clone了整个engine代码，也有sky/tools/flutter_gdb，不能使用这个，原因可以看看flutter_gdb这个py脚本的实现。）

对flutter_gdb工具的注释如下：
```
"""Tool for starting a GDB client and server to debug a Flutter engine process on an Android device.
Usage:
  flutter_gdb server com.example.package_name
  flutter_gdb client com.example.package_name
The Android package must be marked as debuggable in its manifest.
The "client" command will copy system libraries from the device to the host
in order to provide debug symbols.  If this has already been done on a
previous run for a given device, then you can skip this step by passing
--no-pull-libs.
"""
```

看到这里，似乎gdb才是Flutter engine调试的正解，虽然gdb正在被LLDB所替代。

简单的介绍一下flutter_gdb server命令做的事情：

* 获取adb命令，通过adb命令获取包名对于的pid。
* 推送/engine/src/third_party/android_tools/ndk/prebuilt/android-arm64/gdbserver到手机/data/local/tmp目录下。
* 推送/data/local/tmp/gdbserver到/data/data/${package_name}/目录下。
* kill掉正在运行的gdbserver进程。我们在做反调试相关的时候知道，一个进程同时只能被一个进程ptrace。
* 执行gdbserver --attach:端口 pid 命令起一个gdbserver服务。端口不设置的话默认是8888

其中后三步都需要在package_name对应空间下执行，既需要执行 run-as ${package_name}命令。其实我们也完全可以手动执行相关命令。执行完成的话终端会有如下输出：

>Attached; pid = 10548
>
>Listening on port 1234

这样server端就起成功了，整体看下来和lldb差不多，确实lldb也是基于gdb的。

我们在clion中使用<a href="https://www.jetbrains.com/help/clion/remote-debug.html#remote-config">GDB Remote Debug</a>进行配置。然后调试链接时，在之前起server的终端中已经打印:
```
Remote debugging from host 192.168.31.187, port 58287
```

但在clion中报错：
```
com.jetbrains.cidr.execution.debugger.backend.gdb.GDBDriver$GDBCommandException: Remote replied unexpectedly to 'vMustReplyEmpty': PacketSize=47ff;QPassSignals+;QProgramSignals+;QStartupWithShell+;QEnvironmentHexEncoded+;QEnvironmentReset+;QEnvironmentUnset+;QSetWorkingDir+;QCatchSyscalls+;qXfer:libraries-svr4:read+;augmented-libraries-svr4-read+;qXfer:auxv:read+;qXfer:spu:read+;qXfer:spu:write+;qXfer:siginfo:read+;qXfer:siginfo:write+;qXfer:features:read+;QStartNoAckMode+;qXfer:osdata:read+;multiprocess+;fork-events+;vfork-events+;exec-events+;QNonStop+;QDisableRandomization+;qXfer:threads:read+;ConditionalTracepoints+;TraceStateVariables+;TracepointSource+;DisconnectedTracing+;FastTracepoints+;StaticTracepoints+;InstallInTrace+;qXfer:statictrace:read+;qXfer:traceframe-info:read+;EnableDisableTracepoints+;QTBuffer:size+;tracenz+;ConditionalBreakpoints+;BreakpointCommands+;QAgent+;Qbtrace:bts+;Qbtrace-conf:bts:size+;Qbtrace:pt+;Qbtrace-conf:pt:size+;Qbtrace:off+;qXfer:btrace:read+;qXfer:btrace-conf:read+;swbreak+;hwbreak+;qXfer:exec-file:read+;vContSupported+;QThreadEvents+;no-resumed+
```

看日志是server端返回的报文，clion的client端无法识别。查了下push到手机上的gdbserver版本是8.3，而clion用的是10.1。这下得做版本对应。

#### 修改Clion使用的gdb版本
这里最简单的做法是修改Clion的GDB版本。Preferences | Build, Execution, Deployment | Toolchains下修改Debugger的gdb执行文件目录。在engine的编译是会下载整个NDK，前面介绍flutter_gdb server命令做的事情中说道是push /engine/src/third_party/android_tools/ndk/prebuilt/android-arm64/下的gdbserver，而gdb是我们在本机（mac）下运行的，所以应该选择/engine/src/third_party/android_tools/ndk/prebuilt/drawin-x86_64/下的gdb命令。再次开启调试成功连接。


#### 交叉编译android-aarch64下10.1版本的gdbserver并使用过
当然我们也可以自己编译10.1版本的gdbserver，gdbserver是一个针对特定平台的可执行文件，这点在前面说流程的时候有说道，push的是android-arm64下的gdbserver，同时在查看版本的时候如果我们直接在电脑上执行gdbserver --version的话会报：zsh: exec format error: ./gdbserver。



因此只好去下载<a href="https://ftp.gnu.org/gnu/gdb/">gdb10.1版本</a>的源码通过ndk进行编译。

下载好对应版本的源码后，我们进入源码目录，有configure可执行文件，有编译三方库经验的就很清楚，基本流程是通过执行configure命令生成编译配置，然后通过make命令生成编译产物。查看源码中的README,似乎在这个场景下如下执行命令即可：

```
./configure 
make
```

但是在README中有如下内容：
>(If the configure script can't determine your type of computer, give it
>the name as an argument, for instance ``./configure sun4''.  You can
>use the script ``config.sub'' to test whether a name is recognized; if
>it is, config.sub translates it to a triplet specifying CPU, vendor,
>and OS.)

configure脚本是针对单前平台下进行生成编译配置的，然而我们想要的是交叉编译，在mac下编译生成android-arm下可用执行文件。针对交叉编译并没有介绍，如果我们直接执行./configure --target android-arm会出错：
```
checking build system type... x86_64-apple-darwin20.2.0
checking host system type... x86_64-apple-darwin20.2.0
checking target system type... Invalid configuration `android-arm': machine `android-unknown' not recognized
configure: error: /bin/sh ./config.sub android-arm failed
```

因此这里似乎无法直接用过configure命令来进行编译。庆幸的是，gdb相关的在我们NDK中都有支持，我们在/android-ndk-r17c/build/tools目录下能找到build-gdbserver.sh等build脚本用来进行交叉编译。

```
Usage: build-gdbserver.sh [options] <arch> <target-triple> <src-dir> <ndk-dir>

Rebuild the gdbserver prebuilt binary for the Android NDK toolchain.

Where <src-dir> is the location of the gdbserver sources,
<ndk-dir> is the top-level NDK installation path and <toolchain>
is the name of the toolchain to use (e.g. arm-linux-androideabi-4.8).
```

//未完待续


### 使用过Android Studio进行调试
说到底，Clion无法使用过lldb进行调试的原因是没有lldb插件支持，当个Android对NDK的调试就是基于lldb的，在Android studio下完全可以使用过lldb进行调试Native代码。