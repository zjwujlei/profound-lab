JNI下Native代码编译过程
=====================

### cpp编译过程
```cpp
#include <iostream>

int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}

```

对于HelloWorld程序，我们通过gcc编译成可执行文件，进行执行的话就会输出Hello，World！。
```shell script
g++ -o main.out main.cpp
./main

> Hello, World!
```
特别的，对于cpp程序的编译需要用g++而不是gcc。也可以用c++/clang++命令，区别是编译器不同。

整个过程包括1、预处理，2、编译，3、汇编，4、链接。4个步骤。CPP的编译是基于独立的编译单元的，每一个CPP
文件即为一个编译单元。预处理、编译、汇编三步都是基于编译单元的，一一对应生成一个可重定位目标文件（.o文件），这里都是独立进行的。链接就是将这些独立的目标文件链接到一起，生成一个可执行文件。

##### 预处理
```shell script
g++ -E -o main.i main.cpp #生成预处理阶段产物main.i
```
在同目录下生成main.i文件，即为预处理后的结果，可以直接用文本编辑器打开。

预处理阶段是对源码中“# xxx”相关指令的处理。主要包括，1.宏定义（#define）。2.头文件导入（#include）。3.条件编译语句（#ifdef）。

对于宏定义（#define），在预处理阶段会对程序中所有出现的宏定义名的地方，用宏定义字符串进行替换。这里注意是字符串直接替换，并不是语句，因此如果我们宏定义里包含了语句结束符“；”，在使用宏定义的可以不以“；”结尾。

对于头文件导入（#include），在预处理阶段会将头文件直接插入到代码文件中。

对于条件编译语句（#ifdef），在预处理阶段对对条件语句进行判断，符合条件的保留，不符合的去除。条件编译语句一个经典场景是防止头文件重复导入。已经跨平台是也会通过不同的系统进行条件编译。

预处理过程并不涉及到语言的转换，只是代码的复制裁剪。所以这里的include和java的import有本质的区别，java中的import只是一个声明，但include会直接拷贝头文件。

###### 编译
```shell script
g++ -S -o main.s main.cpp #生成编译阶段产物main.s
```

编译是将预处理的结果翻译成汇编语言的过程，最终生成main.s文件，main.s文件里都是汇编代码，用编辑器打开后发现相对于main.i代码量减少了很多。main.i是会将所有的头文件都插入，在这个阶段会进行分析优化保留实际代码。

这里就会有编译原理相关的知识。编译器前端会进行词法分析、语法分析、语义分析。生成中间代码。编译器后端会对代码做优化最终生成汇编语言。
```text
词法分析->语法分析->语义分析->中间码->代码优化->代码生成
```

###### 汇编
```shell script
g++ -o main.o -c main.cpp #生成汇编阶段产物main.o
```

汇编是将汇编语言转化为机器语言的过程，根据汇编代码逐行翻译。生成的main.o文件我们用文本编辑器打开就没法直接预览了，main.o已经是针对当前设备的可执行指令了。

.o文件中包含1.对外导出符号的表，2.未完成寻址的符号的表，3.需要重定向的符号的表。
对于对外导出符号，在.o文件中会为其分配地址，从0x0000开始。
未完成寻址的符号是指不在本目标文件中的部分，需要在链接阶段进行寻址。
需要重定向的部分单独保存在需要重定向符号的表，这是由于每个目标文件的符号地址都是从0x0000开始，多个目标文件会有重复，在链接生成最终可执行文件是会对每个目标文件分配地址和这些符号地址叠加作为最终的地址。

###### 链接
```shell script
g++ -o main.out main.cpp #包含链接阶段完成所有过程
```
实际上对于链接，采用的是link命令
```shell script
link a.o b.o -o main.out
```

链接过程的主要是所有目标文件（库文件、.o文件等）连接起来，生成能被机器执行的格式（ELF等）。
链接过程就是寻址的过程。

* 对每一个目标文件分配地址（在可执行文件中的起始位置）。
* 访问目标文件中需要重定向的符号的表对其中的地址加上做重定向（加上目标文件在可执行文件中的偏移量）
* 访问目标文件中未完成寻址的符号的表，在其他目标文件的导出符号表中确定其在可执行文件中的最终地址。
* 所有地址都确定后，就可以写入生成最终的可执行文件了。

我们知道c++库分为静态库（.a）和动态库（.so），在链接过程中如果依赖的代码在库中，会有不同处理。
对于静态库，会将代码从其所在地静态链接库中直接拷贝到最终的可执行文件中。这样在执行的时候所有的代码都将被一起加载到进程的虚拟地址中，执行起来更快。
对于动态库，函数的代码会继续保存在动态库中，链接程序会在可执行文件中记录下依赖对象的信息，这样在执行时会先根据记录的信息去加载动态库（动态库全量加载到进程中），再在加载的地址中找到对应的函数代码进行执行。

### Android 平台下的Hello，world！
上面我们生成了main.out文件，在本机上直接执行“./main.out”即可打印出“Hello，World！”。

我们试着将main.out文件拷贝到Android设备中并执行，这里要特别注意需要拷贝到/system/bin/目录下，否则没有执行权限，执行chmod命令也无法添加执行权限。
出现如下报错：
```text

```
这是因为我们使用的都是针对本机（ubuntu 20.0.4）的工具进行编译。如果要生成针对Android平台的可执行文件，就需要用到NDK进行交叉编译。

```shell script
aarch64-linux-android-g++ -std=c++11 -pie -fPIE -o main.out main.cpp
```
我们将main.out推送到/system/bin/目录下进行执行，出现如下错误
```text
/system/bin/sh: ./main.out: No such file or directory
```
首先确认的是在/system/bin/确实有main.out，所以这个不存在的文件并不是main.out。
我这里出这个问题是因为链接的时候链接工具没有，前面说道对于动态库执行时会先根据记录的信息去加载动态库，我们readelf命令查看main.out文件。
```shell script
$ aarch64-linux-android-readelf -l main.out 

Elf file type is DYN (Shared object file)
Entry point 0xce8c
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x0000000000000015 0x0000000000000015  R      1
      [Requesting program interpreter: /system/bin/linker64]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x000000000008ebf4 0x000000000008ebf4  R E    10000
  LOAD           0x000000000008f1b0 0x000000000009f1b0 0x000000000009f1b0
                 0x0000000000004fa8 0x00000000000194a0  RW     10000
  DYNAMIC        0x00000000000935a0 0x00000000000a35a0 0x00000000000a35a0
                 0x0000000000000210 0x0000000000000210  RW     8
  NOTE           0x0000000000000250 0x0000000000000250 0x0000000000000250
                 0x0000000000000098 0x0000000000000098  R      4
  GNU_EH_FRAME   0x0000000000070a28 0x0000000000070a28 0x0000000000070a28
                 0x0000000000003f7c 0x0000000000003f7c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x000000000008f1b0 0x000000000009f1b0 0x000000000009f1b0
                 0x0000000000004e50 0x0000000000004e50  R      1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.android.ident .hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .plt .text .rodata .eh_frame_hdr .eh_frame .gcc_except_table 
   03     .preinit_array .init_array .fini_array .data.rel.ro .dynamic .got .data .bss 
   04     .dynamic 
   05     .note.android.ident 
   06     .eh_frame_hdr 
   07     
   08     .preinit_array .init_array .fini_array .data.rel.ro .dynamic .got 

```
查看段头信息可以看到[Requesting program interpreter: /system/bin/linker64]，但我们的设备中只有/system/bin/linker导致了这个问题。


###### 独立编译工具链
如果之前没有生成过独立编译工具链的话，是没有arm-linux-androideabi-g++这个可执行文件的。
使用ndk目录下/build/tools/make-standalone-toolchain.sh进行生成。例如对于arrch64的命令如下：
```shell script
./make-standalone-toolchain.sh --platform=android-21 --install-dir=$TARGET_DIR/android-arrch64 --arch=arm64
```
其中$TARGET_DIR是你要保存的目标目录。然后将对应的bin文件夹添加到环境变量中即可正常使用。
生成的独立编译工具链除了aarch64-linux-android-g++外还有链接工具aarch64-linux-android-ld、crash时内存地址到源码映射工具aarch64
-linux-android-addr2line、输出elf格式文件工具aarch64-linux-android-readelf等。

在生成过独立编译工具链时，由于是在新配置的ubuntu上操作，出现了ERROR: Failed to create toolchain。的错误，而且没有额外日志。
这个时候通过带入--help参数查看帮助文档，发现可以通过设置--verbose参数来开启verbose模式，这样就有详细信息输出。
```text
./ndk-common.sh: line 122: python: command not found
ERROR: Failed to create toolchain.
```
明显是没有python，我本身是安装了python3，但在/usr/bin/确实没有python命令，且使用which is python也无法找到。
这里直接用ln命令创建一个文件链接。
```shell script
sudo ln -s /usr/bin/python3 /usr/bin/python
```
通过ln命令创建链接其实很常见，包括mac下，包括Android中的目录。

### Android Studio下编译Native工程
有了手工编译执行C++程序的经验下，我们再来看看Android Studio下是如何编译Native工程生成动态库的。

###### cmake
早期我们使用的是ndk-build，当前官方推荐的方式是cmake，而且现在AS上可以认为是强制使用cmake的。
个人理解上cmake（包括ndk-build）只是一个编译配置生成工具。上面我们手动编译Hello world！程序时代码十分简单，因此我们使用的编译命令也很简单。
实际场景上我们编译过程会复杂很多，例如c/c++混合开发是我们要同时指定c/c++的编译器，例如使用库是我们要设置额外的include目录。
以及实际场景下我们的编译单元不会和Hello world！程序一样只有一个。
这些都使得编译是一个很复杂的过程，直接来处理编译过程是很难度的一件事情。
cmake就是来做这件事情的工具，程序员通过在CMakeLists.txt中声明配置，cmake工具会根据这些配置来生成编译过程。

###### ninja

>Ninja 是Google的一名程序员推出的注重速度的构建工具，一般在Unix/Linux上的程序通过make/makefile来构建编译，而Ninja通过将编译任务并行组织，大大提高了构建速度。

如果cmake对于的是早期使用的ndk-build，ninja就是对应早期使用的make。
早期使用ndk-build，通过输入的Android.mk和Application.mk文件生成MakeFile文件，然后通过make执行。
同样的，cmake也会为Ninja生成build.ninja文件。

###### gradle tasks
不同的AGP版本，task会有所不同但做的事情基本一致，这里是使用AGP 4.2.2，gradle 6.7.1下的情况：
```text
:libnative:generateJsonModelRelease SKIPPED
:libnative:externalNativeBuildRelease SKIPPED
:libnative:mergeReleaseJniLibFolders SKIPPED
:libnative:mergeReleaseNativeLibs SKIPPED
:libnative:stripReleaseDebugSymbols SKIPPED
:libnative:copyReleaseJniLibsProjectAndLocalJars SKIPPED
```

对于generateJsonModelRelease，我们打印其类名为：com.android.build.gradle.tasks.ExternalNativeBuildJsonTask_Decorated。
当前的AGP都改成用KT实现了，尴尬的是使用了KT之后，我们直接通过 implementation 'com.android.tools.build:gradle:4.2.2'添加依赖查看源码是，很多代码都是 /* compiled code */ 无法查看。
这里直接从googlesorce上<a href="">studio-4.2.0</a>查看源码。

看下这个任务的doTaskAction函数：
```kotlin
override fun doTaskAction() {
    IssueReporterLoggingEnvironment(DefaultIssueReporter(LoggerWrapper(logger))).use {
        val generator =
            createCxxMetadataGenerator(
                configurationModel,
                analyticsService = analyticsService.get()
            )
        for (future in generator.getMetadataGenerators(ops, false, null)) {
            future.call()
        }
    }
}
```
调用createCxxMetadataGenerator函数CxxMetadataGenerator的具体实例，根据variant.module.buildSystem（ndk-build/cmake）及其版本确定。
这里使用cmake 4.10.2，对应的实例是CmakeServerExternalNativeJsonGenerator。
CmakeServerExternalNativeJsonGenerator为我们生成了编译是的各种配置及环境，主要包括：

1、各种.cmake文件，例如CMakeCXXCompiler.cmake文件，是对c++编译的配置，进行了编译器、链接器等设置。
2、build_command.txt文件，对cmake执行的参数配置。
例如-DCMAKE_TOOLCHAIN_FILE=/home/xxx/Android/android-ndk-r17c/build/cmake/android.toolchain.cmake参数指定了toolchain。
android.toolchain.cmake这个配置本身是在NDK中维护的，因此它知道各个参与编译的工具在NDK中的目录。
之前我们使用make-standalone-toolchain.sh构建了独立编译工具链，在构建时我们要传入交叉编译的目标平台。
这里也一样，android.toolchain.cmake内部会根据当前的机器平台和目标平台（ABI）来设置一系列的环境参数用来获取对于的编译工具文件地址。
3、各种.json文件。
例如compile_commands.json文件，是对编译单元（需要参与编译的cpp文件）的配置。
4、rules.ninja和build.ninja。是ninja命令文件，和MakeFile一样，记录了整个编译过程的各个单步的命令。

相关环境配置都收集完毕后，externalNativeBuildRelease任务就是用来执行命令生成最终的动态库SO的。
















