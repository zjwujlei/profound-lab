Flutter调试Engine代码
====================

Flutter Engine部分代码较多，切网上资料较少，学习切入点不好寻找。这里通过调试作为切入点进行学习。

###lldb

<a href="https://lldb.llvm.org/use/remote.html">lldb</a>是一个代码调试工具，是一个底层调试器(low level debugger)Android中NDK下开发调试就是基于lldb。lldb是基于Client-Server架构，Client和Server通过gdb-remote协议进行通信，基于TCP/IP进行传输。

######lldb-server

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

###### lldb

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
add-dsym /Users/wujinglei/github-space/engine/src/out/android_debug_unopt/libflutter.so
#关联源码库和符号目录。
settings set target.source-map /Users/wujinglei/github-space/engine/src/out/android_debug_unopt /Users/wujinglei/github-space/engine/src/
```

至此，lldb client端完成，可以通过命令行进行调试。lldb的调试语法可以参见<a href="https://lldb.llvm.org/use/map.html#id2">vBreakpoint Commands</a>


### 使用clion进行调试 