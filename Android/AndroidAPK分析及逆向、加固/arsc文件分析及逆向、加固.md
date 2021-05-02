arsc文件分析及逆向、加固
===================

###  arsc文件的数据结构

arsc文件的数据结构定义在 fameworks/base/incllude/androidfw/ResourceTypes.h中。
arsc文件由一个个chunk组成，有写chunk是嵌套的，有写chunk是平级的。

* RES_TABLE_TYPE的chunk 数量为1
    * RES_STRING_POOL_TYPE 的chunk 数量为1
    * RES_TABLE_PACKAGE_TYPE 的chunk 数量为1
        * 资源类型字符串池
        * 资源项名称字符串池
        * RES_TABLE_TYPE_SPC_TYPE 的chunk 数量为N
        * RES_TABLE_TYPE_TYPE 的chunk 数量为N


缩进代表嵌套。

每一个chunk都有头部信息和数据块组成。对于chunk头部信息至少包含了type（2个字节）：chunk类型、headerSize(2个字节)：头部信息字节大小、size（4个字节）：当前chunk大小，这三个信息。

特别的，arsc文件采用的是小端字节结构，既对于字节码，位数小的反而在前面。


###### RES_TABLE_TYPE的chunk
 头信息的数据结构
```kotlin

    var type:Short = 0 //类型，2
    var headerSize:Short = 0 //头部大小， 12
    var size:Int = 0 //当前chunk大小，对于RES_TYPE_TABLE来说就是文件大小。
    var packageCount:Int = 0 //package数，1

```


###### RES_STRING_POOL_TYPE的chunk

这个chunk是一个字符串池，包含了所有在资源包里面定义的资源项的值字符串

头部信息数据结构

```kotlin

    var type:Short = 0 //类型，值2
    var headerSize:Short = 0 //头部大小， 值28
    var size:Int = 0 //当前chunk大小
    var stringCount:Int = 0 //字符串数
    var styleCount:Int = 0 //style数
    var flag:Int = 0 //flag，取值：0x000(UTF-16)、0x001(字符串经过排序)、0x100(UTF-8)或组合
    var stringStart:Int = 0 //字符串起始位置，特别注意是相对于当前chunk的起始位置。相对于文件位置需要加上ResTypeTable chunk的头部信息大小（12）
    var styleStart:Int = 0 //style起始位置，特别注意是相对于当前chunk的起始位置。

```

对于字符串块的结构，有两点需要特别注意。
* 前两个字节表示字符串的长度。但我们取一个字节就可以了。Android里面字符串资源长度不能够超过256，一个字节即可保存，这里的两个字节都保存了相同的值（例如对于长度为6的字符串，值为 0x0606）。
* 字符串结束之后有结束分割符，对于UTF-8，分割符为0x00，对于UTF-16，分割符为0x000.



