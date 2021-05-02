Android JNI使用三方库入门
======================
### 说明
我们在做JNI开发的时候，经常会使用到一些三方库，我们基本是以预构建库的形式引入的。预构建库的介绍可以看<a href="https://developer.android.google.cn/ndk/guides/prebuilts?hl=zh_cn">android developer</a>。本文记录入引入OpenSSL库作为加解密实现的整个过程。

### 源码处理
openssl库使用c实现，保证平台兼容性，我们从<a href="https://www.openssl.org/">官网</a>上下载1.1.1版本<a href="http://artfiles.org/openssl.org/snapshot/">快照</a>"，也可以从github上下载。

openssl库源码是c实现，无法直接为Android平台使用，因此我们需要用NDK编译打包成Android对应cpu架构平台上的静态库或者动态库。

######NOTE
对于OpenSSL的源码，通过NOTES.ANDROID、NOTES.DJGPP、NOTES.WIN等文件来说明针对对应平台应该如何进行编译。我们看下NOTES.ANDROID中的介绍：

```txt

 NOTES FOR ANDROID PLATFORMS
 ===========================

 Requirement details
 -------------------

 Beside basic tools like perl and make you'll need to download the Android
 NDK. It's available for Linux, macOS and Windows, but only Linux
 version was actually tested. There is no reason to believe that macOS
 wouldn't work. And as for Windows, it's unclear which "shell" would be
 suitable, MSYS2 might have best chances. NDK version should play lesser
 role, the goal is to support a range of most recent versions.

 Configuration
 -------------

 Android is a naturally cross-compiled target and you can't use ./config.
 You have to use ./Configure and name your target explicitly; there are
 android-arm, android-arm64, android-mips, android-mip64, android-x86
 and android-x86_64 (*MIPS targets are no longer supported with NDK R20+).
 Do not pass --cross-compile-prefix (as you might be tempted), as it will
 be "calculated" automatically based on chosen platform. Though you still
 need to know the prefix to extend your PATH, in order to invoke
 $(CROSS_COMPILE)clang [*gcc on NDK 19 and lower] and company. (Configure
 will fail and give you a hint if you get it wrong.) Apart from PATH
 adjustment you need to set ANDROID_NDK_HOME environment to point at the
 NDK directory. If you're using a side-by-side NDK the path will look
 something like /some/where/android-sdk/ndk/<ver>, and for a standalone
 NDK the path will be something like /some/where/android-ndk-<ver>.
 Both variables are significant at both configuration and compilation times.
 The NDK customarily supports multiple Android API levels, e.g. android-14,
 android-21, etc. By default latest API level is chosen. If you need to
 target an older platform pass the argument -D__ANDROID_API__=N to Configure,
 with N being the numerical value of the target platform version. For example,
 to compile for Android 10 arm64 with a side-by-side NDK r20.0.5594570

	export ANDROID_NDK_HOME=/home/whoever/Android/android-sdk/ndk/20.0.5594570
	PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin:$PATH
	./Configure android-arm64 -D__ANDROID_API__=29
	make

 Older versions of the NDK have GCC under their common prebuilt tools directory, so the bin path
 will be slightly different. EG: to compile for ICS on ARM with NDK 10d:

    export ANDROID_NDK_HOME=/some/where/android-ndk-10d
    PATH=$ANDROID_NDK_HOME/toolchains/arm-linux-androideabi-4.8/prebuilt/linux-x86_64/bin:$PATH
    ./Configure android-arm -D__ANDROID_API__=14
    make

 Caveat lector! Earlier OpenSSL versions relied on additional CROSS_SYSROOT
 variable set to $ANDROID_NDK_HOME/platforms/android-<api>/arch-<arch> to
 appoint headers-n-libraries' location. It's still recognized in order
 to facilitate migration from older projects. However, since API level
 appears in CROSS_SYSROOT value, passing -D__ANDROID_API__=N can be in
 conflict, and mixing the two is therefore not supported. Migration to
 CROSS_SYSROOT-less setup is recommended.

 One can engage clang by adjusting PATH to cover same NDK's clang. Just
 keep in mind that if you miss it, Configure will try to use gcc...
 Also, PATH would need even further adjustment to cover unprefixed, yet
 target-specific, ar and ranlib. It's possible that you don't need to
 bother, if binutils-multiarch is installed on your Linux system.

 Another option is to create so called "standalone toolchain" tailored
 for single specific platform including Android API level, and assign its
 location to ANDROID_NDK_HOME. In such case you have to pass matching
 target name to Configure and shouldn't use -D__ANDROID_API__=N. PATH
 adjustment becomes simpler, $ANDROID_NDK_HOME/bin:$PATH suffices.

 Running tests (on Linux)
 ------------------------

 This is not actually supported. Notes are meant rather as inspiration.

 Even though build output targets alien system, it's possible to execute
 test suite on Linux system by employing qemu-user. The trick is static
 linking. Pass -static to Configure, then edit generated Makefile and
 remove occurrences of -ldl and -pie flags. You would also need to pick
 API version that comes with usable static libraries, 42/2=21 used to
 work. Once built, you should be able to

    env EXE_SHELL=qemu-<arch> make test

 If you need to pass additional flag to qemu, quotes are your friend, e.g.

    env EXE_SHELL="qemu-mips64el -cpu MIPS64R6-generic" make test
```

这里对NDK等的环境配置不做讨论。

###### 生成预编译库
在介绍中最重要的是Configure步骤，在源码中有Configure的可执行文件，就是通过这个文件进行。由于我用的是NDK17，根据本机环境及不同的编译CPU Target，执行进行如下命令
```
#export ANDROID_NDK_HOME=/some/where/android-ndk-10d//设置NDK_HOME，已在环境变量中设置
export PATH=$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin:$PATH//将对于平台的编译工具目录设置到PATH中。arm-linux-androideabi-4.9对于的是android-arm。在mac环境下相关工具是在darwin-x86_64目录下，用NOTES.ANDROID中看介绍是linux机器上的配置，可以在NDK中找到具体目录，
./Configure android-arm -D__ANDROID_API__=21 --prefix=${output}//通过-D__ANDROID_API__设置Android API，本次生成用21，可以通过通过--prefix=直接设置输出目录。configure后会生成makefile。
make //用make命令进行生成。
```

android支持armeabi-v7a armeabi arm64-v8a x86 x86_64 mips mips64的不同CPU架构，除了armeabi-v7a armeabi对应的工具链目录相同外，其他都有对应的工具链，可以分别调用生成。同时Configure命令传入的arch_name也不同，armeabi-v7a armeabi对应都是android-arm，其他各部相同。

### 使用预编译库

```
#设置include目录，我们代码中要使用三方库，从该目录导入头文件。
#特别注意include目录不要有’‘，直接设置，否则clang++命令-I设置的目录会错，-I"../../../../'目录'"。 
#include_directories('${CMAKE_SOURCE_DIR}/src/main/cpp/include')
include_directories(${CMAKE_SOURCE_DIR}/src/main/cpp/include)
# 添加第三方库，我们可以直接使用对应的源码通过add_library添加，这时我们可以用多个cmake文件编写，可以在主cmake文件中用 ADD_SUBDIRECTORY( "相对目录" )添加其他cmake文件。也可以直接使用预构建库（.a或者.so）。这里使用预构建库集成。

#添加一个预构建静态库
add_library(crypto STATIC IMPORTED)
#设置crypto预构建静态库的本地导入目录属性
set_target_properties(crypto
        PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libcrypto.a)
```


通过cmake使用预构建库需要add_library和set_target_properties配合使用。如果用Android.mk文件，参考<a href="https://developer.android.google.cn/ndk/guides/prebuilts?hl=zh_cn">预构建库的介绍</a>


###OPENSSL
MD5<a href="https://www.openssl.org/docs/manmaster/man3/MD5.html">使用文档</a>
SHA1<a href="https://www.openssl.org/docs/manmaster/man3/SHA1.html">使用文档</a>
RSA_public_decrypt<a href="https://www.openssl.org/docs/manmaster/man3/RSA_public_decrypt.html">使用文档</a>
RSA_public_encrypt<a href="https://www.openssl.org/docs/manmaster/man3/RSA_public_encrypt.html">使用文档</a>
RSA_private_decrypt<a href="https://www.openssl.org/docs/manmaster/man3/RSA_private_decrypt.html">使用文档</a>
RSA_private_encrypt<a href="https://www.openssl.org/docs/manmaster/man3/RSA_private_encrypt.html">使用文档</a>

这种方式的使用，在文档里注明了会在3.0版本之后废弃，例如使用公钥加密，<a href="https://www.openssl.org/docs/manmaster/man3/EVP_PKEY_encrypt.html">EVP_PKEY_encrypt</a>。
```
 #include <openssl/evp.h>

 int EVP_PKEY_encrypt_init(EVP_PKEY_CTX *ctx);
 int EVP_PKEY_encrypt_init_ex(EVP_PKEY_CTX *ctx, const OSSL_PARAM params[]);
 int EVP_PKEY_encrypt(EVP_PKEY_CTX *ctx,
                      unsigned char *out, size_t *outlen,
                      const unsigned char *in, size_t inlen);
```


### 踩坑

###### 问题一

```
A/libc: Fatal signal 4 (SIGILL), code 1 (ILL_ILLOPC), fault addr 0xc4207e10 in tid 11501 (mo.apikeyverify), pid 11501 (mo.apikeyverify)
```

定义了有返回值的函数但是没有进行return值。编译是有如下警告：
```
warning: control reaches end of non-void function [-Wreturn-type]
```

###### 问题二

```c++
    jclass signature_clazz = env->GetObjectClass(signature);
    jmethodID signature_toByteArray = env->GetMethodID(signature_clazz,
                                                       "toByteArray", "()[B");
    jbyteArray sig_bytes = (jbyteArray) env->CallObjectMethod(
            signature, signature_toByteArray);
    jbyte *jchars = env->GetByteArrayElements(sig_bytes, 0);
    uint32_t len = env->GetArrayLength(sig_bytes);
    char *chars = new char[len + 1];
    memset(chars, 0, len + 1);
    memcpy(chars, jchars, len);
    chars[len] = 0;
```

使用toByteArray获取byte数组，然后拷贝成char*。取到的SHA1值不正确。

byte和char是不同的。在c++中要做hex转化。可以使用toChars取到jcharArray就可以。