KTNative落地分析
======================================

### 简介


Kotlin/Native 是一种将 Kotlin 代码编译为无需虚拟机就可运行的原生二进制文件的技术。 它是一个基于 LLVM 的 Kotlin 编译器后端以及 Kotlin 标准库的原生实现。

当前前支持一下平台:

>iOS（arm32、 arm64、 模拟器 x86_64）

>MacOS（x86_64）

>Android（arm32、arm64）

>Windows（mingw x86_64、x86）

>Linux（x86_64、 arm32、 MIPS、 MIPS 小端次序、树莓派）

>WebAssembly（wasm32）

详见<a href="http://www.kotlincn.net/docs/reference/native-overview.html">官网翻译</a>

更准确的表述也可查阅<a href="https://kotlinlang.org/docs/reference/native-overview.html">英文官网</a>

### 实现过程
在官网的Kotlin/Native 开发动态库部分，提供了针对Linux、Macos、Windows提供了详细的教程。针对上述三个平台，会分别生成.h头文件和对应的动态库（.so、.dylib、.dll）。理论上讲我们使用Linux相关的配置即可生成需要的动态库。但实际操作下发现需要在Linux机器上能进行编译出.so.

```
Target 'native' for platform linux_x64 is ignored during build on this macos_x64 machine. You can build it with a linux_x64 host.

```


针对Android平台，官网上没有详细介绍，在<a href="https://www.kotlincn.net/docs/tutorials/native/mpp-ios-android.html">多平台项目: iOS 与 Android</a>部分中，对应的Android实现上并未将kotlin代码编译成动态库使用。

对着最多PC端的配置，和通过网上查询资料，对Kotlin/Native For Android的完整gradle配置如下：

```
import org.jetbrains.kotlin.gradle.plugin.mpp.NativeBuildType

plugins {
    kotlin("multiplatform") version "1.3.21"
    id("com.android.library")
}

val jniLibDir = File(project.buildDir, arrayOf("generated", "jniLibs").joinToString(File.separator))

kotlin {
    androidNativeArm32 {
        binaries {
            sharedLib("knlib")
        }
    }
}

android {
    compileSdkVersion(26)

    defaultConfig {
        minSdkVersion(23)
        targetSdkVersion(26)

        ndk {
            abiFilters("armeabi-v7a")
        }
    }

    sourceSets {
        val main by getting {
            jniLibs.srcDir(jniLibDir)
        }
    }
}

```


### 剩余问题

* 复杂的功能开发的实现。
* Kotlin/Native工程对其依赖库（kotlin源码依赖、jar库依赖）是否能够处理。
* 当前打包默认输出的是armeabi-v7a。其他cpu的结果能否输出或者能否直接用。
* Kotlin/Native工程是通过linkReleaseShared打包产物的，能否和APP打包流程进行串联。



### 缺点

* Kotlin/Native 只能用来实现逻辑，具体依赖其他aar库是不好处理。
* Kotlin/Native 国内资料比较少。

### 优势

* Kotlin/Native 工程也是基于Gradle来实现构建过程，完全可以作为一个Android的一个library工程，使用方便。
* 使用kotlin开发，开发过度成本低。


### 资料

<a href="https://www.kotlincn.net/docs/tutorials/native/dynamic-libraries.html">Kotlin/Native 开发动态库</a>

<a href="https://github.com/enbandari/hello-kni">hello-kni demo</a>
