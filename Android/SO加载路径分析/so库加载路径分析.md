so库加载路径分析:
================
### so库加载路径分析
无论是系统应用，还是第三方应用，调用so库都是通过System.loadLibrary（文件名）或System.load（绝对路径）的方式调用，后一种方式以及确定了so库，我们代码内使用的是System.loadLibrary（文件名）这种方式。
通过System.loadLibrary的时候，入参是libname，只需要传入文件名，经过搜索确定so所在的绝对路径。
方法调用链：
System.loadLibrary ->Runtime.loadLibrary0->ClassLoader.findLibrary->DexPathList.findLibrary

DexPathList.findLibrary中通过for循环遍历所有可能的路径进行查找
```java
	public String findLibrary(String libraryName) {
        String fileName = System.mapLibraryName(libraryName);

        for (Element element : nativeLibraryPathElements) {
            String path = element.findNativeLibrary(fileName);

            if (path != null) {s
                return path;
            }
        }

        return null;
    }
```
所有可能的路径nativeLibraryPathElements是在DexPathList初始化时确定。

```java
        this.nativeLibraryDirectories = splitPaths(libraryPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
        allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);

        this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories, null,suppressedExceptions);

```


可以看到，所有可能的路径allNativeLibraryDirectories的List中，先通过apk的本地库 径创建，后添加了系统本地库的路径，最后通过makePathElements 法，创建出上面for循环遍历的nativeLibraryPathElements。

因此这里可以确定在搜索so库的先搜索apk的本地库径，后搜索系统的本地库的路径

系统本地库的路径System.getProperty("java.library.path")是系统属性，经过打印是/vendor/lib，/system/lib。

apk本地库路径由DexPathList创建是传入的libraryPath决定，DexPathList存储的是apk安装后所有资源的路径。

该值最初是在创建BaseDexClassLoader的时候传入的，其参数注释是: @param libraryPath the list of directories containing native libraries, delimited by {@code File.pathSeparator}; may be {@code null}
具体创建过程可能在c层创建，搜索到的相关代码如下:

```c++
 // Create DexPathList.
Handle<mirror::Object> h_dex_path_list = hs.NewHandle( dex_elements_field->GetDeclaringClass()->AllocObject(self));
DCHECK(h_dex_path_list.Get() != nullptr);
// Set elements.
dex_elements_field->SetObject<false>(h_dex_path_list.Get(), h_dex_elements.Get());
  // Create PathClassLoader.
Handle<mirror::Class> h_path_class_class = hs.NewHandle( soa.Decode<mirror::Class*>(WellKnownClasses::dalvik_system_PathClassLoader));
Handle<mirror::Object> h_path_class_loader = hs.NewHandle( h_path_class_class->AllocObject(self));
DCHECK(h_path_class_loader.Get() != nullptr); 
// Set DexPathList.
ArtField* path_list_field = soa.DecodeField(WellKnownClasses::dalvik_system_PathClassLoader_pathList);
DCHECK(path_list_field != nullptr); path_list_field->SetObject<false>(h_path_class_loader.Get(), h_dex_path_list.Get());
```
这里我的理解是，android的每个apk都跑在独立的虚拟机上，所有的的虚拟机都是从Zygote中fork出来的，创建BaseDexClassLoader等过程应该是统一的，这个值相对于应用来说，应该是固定的。

参考加载不存在的so时报错输出如下信息：

java.lang.UnsatisfiedLinkError: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/${packageName}- 2/base.apk"],nativeLibraryDirectories=[/data/app/${packageName}-2/lib/arm, /data/app/${packageName}- 2/base.apk!/lib/armeabi, /vendor/lib, /system/lib]]] couldn't find "lib333locSDK7.so"

与DexPathList的toString，上面的nativeLibraryPathElements确定逻辑相对照

```java
@Override 
public String toString() {
    List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories); 
    allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);
    File[] nativeLibraryDirectoriesArray = allNativeLibraryDirectories.toArray(new File[allNativeLibraryDirectories.size()]); 
    return "DexPathList[" + Arrays.toString(dexElements) + ",nativeLibraryDirectories=" + Arrays.toString(nativeLibraryDirectoriesArray) + "]";
}
```
可以得出，so的所有搜索路径allNativeLibraryDirectories是 [/data/app/${packageName}-2/lib/arm, /data/app/${packageName}- 2/base.apk!/lib/armeabi, /vendor/lib, /system/lib]

其中${packageName}-2是当前版本apk安装时生成的目录，这个过程中会把上次安装${packageName}-1的目录删除，这个变化通过adb命令可以查看。


### 预转系统应用

预装应用加载SO的路径如下：nativeLibraryDirectories=[/system/app/${app}/lib/arm, /system/app/${app}/${app}.apk!/lib/armeabi-v7a, /system/lib, /system/vendor/lib, /system/lib, /system/vendor/lib]。

系统应用在安装时，PackageManaer并不会自动抽离释放APK内部的so文件。系统应用的SO是在ROM包打包的时候进行释放到同级目录下的lib目录下（对应so加载路径”/system/app/${app}/lib/arm“）。这步释放需要在ROM包的packages/apps/${app}下创建lib目录放置so文件并在Android.mk文件中进行配置。


###### 内置应用so处理方案



方案一：ROM包制作过程中入手，根据ROM包的buid规范实现SO自动释放，但不确定能否解决。

方案二：直接将我们的SO拷贝到/system/lib下，该方案可以解决，当需要和厂商进行协调，且会被作为公共SO。
  