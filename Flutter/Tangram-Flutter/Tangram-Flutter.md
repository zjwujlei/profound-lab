Tangram-Flutter
===============

### Tangram-Android

#### 加载模板

使用ViewManager的loadBinBufferSync函数进行加载模板。

###### 编译后.out文件格式
定义的.out文件格式和zip格式类似，header+data两部分组成。

```

对于header部分的格式是固定的

5个字节的固定TAG:ALIVV
2个字节（short）的MajorVersion
2个字节（short）的MinorVersion
2个字节（short）的PatchVersion:在templatelist.properties中设置的PatchVersion
4个字节（int）的uiCodeStart
4个字节（int）的uiCodeLength
4个字节（int）的stringCodeStart
4个字节（int）的stringCodeLength
4个字节（int）的exprCodeStart
4个字节（int）的exprCodeLength
4个字节（int）的extraDataStart
4个字节（int）的extraDataLength
2个字节（short）的pageId
2个字节（short）的dep pages数目
dep pages数目 * 2个字节的depPageIds

内容部分就是我们编写的xml的数据：

UI Block:uiCodeStart为起点的块
    4个字节（int）的Count: 必须是1
    2个字节（short）的Name字符串的长度nameSize
    nameSize个字节的 Name
    2个字节（short）的uiCode字节码的长度uiCodeSize
    uiCodeSize个字节的Code
        1个字节的起始符号：0
        2个字节（short）的组件ID comID
        1个字节的int类型属性数目:attrCount
        attrCount * 2 * 4 个字节的int属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
        1个字节的RP类型属性数目:attrCount
        attrCount * 2 * 4 个字节的RP属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
        1个字节的float类型属性数目:attrCount
        attrCount * 2 * 4 个字节的float属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
        1个字节的float RP类型属性数目:attrCount
        attrCount * 2 * 4 个字节的float RP属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
        1个字节的string类型属性数目:attrCount
        attrCount * 2 * 4 个字节的string属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
        1个字节的expr code类型属性数目:attrCount
        attrCount * 2 * 4 个字节的expr code属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
        1个字节的user var类型属性数目:attrCount
        attrCount * 2 * 4 个字节的user var属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的起始符号：0
            2个字节（short）的组件ID comID
            1个字节的int类型属性数目:attrCount
            attrCount * 2 * 4 个字节的int属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的AP类型属性数目:attrCount
            attrCount * 2 * 4 个字节的AP属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的float类型属性数目:attrCount
            attrCount * 2 * 4 个字节的float属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的float AP类型属性数目:attrCount
            attrCount * 2 * 4 个字节的float AP属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的string类型属性数目:attrCount
            attrCount * 2 * 4 个字节的string属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的expr code类型属性数目:attrCount
            attrCount * 2 * 4 个字节的expr code属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的user var类型属性数目:attrCount
            attrCount * 2 * 4 个字节的user var属性，int类型的key<->value对属性数据。其中key为属性name的hashcode。
            1个字节的起始符号：1

            ...

        1个字节的起始符号：1

String Block：以stringCode为起点的块
    4个字节（int）的字符串个数
        4个字节（int）的该字符串index（该String的hashcode，对应的id）
        2个字节（short）该字符串的长度:length
        length个字节的字符串数据
        ...

Expr Block：以exprCodeStart为起点的块，在xml中写入的逻辑代码
    4个字节（int）的code个数。
        4个字节（int）的该代码片段的id，代码字符串的hashcode
        2个字节（short）该代码片段的长度:length
        length个字节的代码片段
        ...
Extra Data Block
    扩展的额外数据，官方实现中为空。可以做加签


```

###### 编译过程简述

编译过程在ViewCompiler来中，主要流程涉及三个函数。newOutputFile，compile，compileEnd。

newOutputFile函数主要做初始化操作。输出文件创建，StringStore、ExprCodeStore创建，byte数据存储对象RandomAccessMemByte的创建。同时预填header数据。预填时，除了固定TAG（ALIVV）、MajorVersion、MonirVersion、PatchVersion外，其他都以“0”填充。作为站位。

compile函数真正对xml做编码。我们读取XML文件，根据XML的结构来生成编译后的代码。我们使用ConfigParser来做解析，主要是将标签名、属性名、属性值都装换为Int类型的ID。

ConfigParser加载了config.properties文件中的配置信息。config.properties的介绍可以查看readme。通过ConfigParser我们可以将标签名名称转化为Int类型的ID。

属性有系统属性和自定义属性的区分，对于系统属性：

所有的系统属性我们在StringStore中穷举，我们直接取通过StringStore类可以去到其hashcode作为ID。

对于系统属性值，我们使用ValueParserCenter来做解析，ValueParserCenter也加载了config.properties文件中的配置，对于特定的属性，根据config.properties中配置的特定属性解析器进行解析。例如layoutWidth属性，使用LayoutWidthHeightValueParser进行解析，如果是wrap_content则属性值为-2。特别的我们在解析之前就默认认为是String属性，因此如果没有找到特定的XXXValueParser，该AttrItem的类型即为String类型。

所有的String值，我们都保存在StringStore中。当我们碰到一个String类型的AttrItem时，在StringStore中取一下其对应的ID，如果不存在会添加，其ID即为String的hashcode。StringStore中保存所有String数据，这些数据会作为String Block在后续阶段保存。

特别的对于表达式属性，使用的ExprValueParser进行解析，官方本身提供了用文件外部导入代码块的能力，见ExprCodeStore的mBlockMap相关。但在命令行中没有开放出。直接在XML中写入表达式的话需要使用“${statement}”实现，我们将其中的statement做编码保存在ExprCodeStore中，同时对表达式去hashcode作为表达式属性值的ID。

对于自定义属性也可以分为常量自定义属性和非常量自定义属性

* 常量自定义属性以var_开头的，其格式为var_type_name。type只支持int、float、String。我们通过StringStore渠道对于name的ID，如果不存在会添加。然后对于int、float转化为Int类型ID，对于StringStringStore渠道对于name的ID。

* 非常量自定义属性的处理方式和系统属性一致，只是对于属性名的String，我们要将其保存到StringStore中，并以hashcode作为其ID，而系统属性的属性名可以认为是内置的并固定ID。对于属性值则和系统属性值的解析一致，来获取其ID。

compile函数到这里就完成了一个xml标签的解析。我们调用writeFile函数将其写入，即为out格式分析中的一整块。

xml文件中有很多的标签，且其存在嵌套关系。因此compile函数本身是一个while循环，循环体内就是上述的解析实现。对于整个XML解析来说，我们在解析到XmlPullParser.START_TAG时，写入起始符号“0”，在碰到XmlPullParser.END_TAG标签的时候写入结束符号“1”。同时有parentParser的概念，子节点可调用父节点的解析器，但在官方提供的原始的中也不会到用到。

compileEnd函数最后调用。在newOutputFile函数中我们都是预填”0“的，通过compile函数解析完成后我们就知道了相关的值，可以进行填入。对于StringStore和ExprCodeStore中保存的String和代码片段，我们也在这里进行写入。

至此一个XML模板的解析就完成了，我们生成了对应的out文件。

###### 加载模板

模板的加载入口在ViewManager.loadBinBufferSync函数中。主要通过BinaryLoader、ExprCodeLoader、StringLoader和UiCodeLoader来实现 。函数调用链到达BinaryLoader.loadFromBuffer后对byte数组进行解析。按照”编译后.out文件格式“进行解析，在校验TAG(ALIVV)和版本之后开始调用对应的Loader进行加载。

UiCodeLoader会通过mTypeToCodeReader和mTypeToPos两个map来保存改模板卡片的数据。需要获取对应模板的代码时只需要调用getcode函数来获取。

StringLoader也通过map来保存所有的String和其ID的关系。和编译out文件时一样，这里会区分系统字符串和其他字符串。系统字符串即为原始支持的属性名称，在编译过程简述部分可以知道是直接内置的。

ExprCodeLoader也通过map来保存所有的代码和ID的关系，特别的把代码封装到了ExprCode中，里面包括了bytes数组和起始位置，长度。

通过这些Loader加载的数据，我们都关联到一个ViewManager中。ViewManager中注意到有setPageContext（VafContext context）函数，我们有看到VafContext有成对出现的init(Context context)和uninit函数。因此我们在多页面使用Tangram方案实现时，我们是可以共用ViewManager的。在提供的demo里面，ViewManager、VafContext都是在Application中创建共用的。



###### 注册模板

DefaultResolverRegistry：用来注册所有的cell和card。我们加载了模板后，需要将其进行注册。我们编写的一个视图模板，即可作为一个Card（recycleview的一个item），也可以作为一个基础的cell来组成Card。因此我们在注册的时候会同时调用为Card和 Cell。我们主要通过registerVirtualView、registerCell和registerCard进行注册。当然Tangram提供的API中我们通过TangramBuilder.InnerBuilder来调用。

registerCard就是组成recycleview的一个item的样式，框架内部在创建Tangram时已经为我们注册了很多内置的Card(TangramBuilder.InnerBuilder的installDefaultRegistry)。对于我们自定义的动态下发的样式，我们无法像预置Card一样提前定义好对应的Card实现类（例如Banner对应内置的BannerCard），我们可以使用VVCard类，VVCard是一种只有一个原生的Card，当然我们也可以自己实现，我们将所有支持的Card都保存到CardResolver中。


registerCell就是注册Cell了，框架同时也注册了部分内置Cell。我们基本使用“registerCell(String type, final @NonNull Class<V> viewClz) ”进行注册。其中viewClz为该类型Cell对应的View实现，对于自定义的动态下发的样式如果使用registerCell注册，我们同样无法一一预先定义，我们可以使用SimpleEmptyView，当然我们也可以定义一个义统一的实现（WYTangramView）。我们将相关数据保存在BaseCellBinderResolver和MVHelper中在后续使用。MVHelper主要用来更具JSON数据还原出元素树，BaseCellBinderResolver则用来创建View。


registerVirtualView会同时注册Card和Cell。注册Card时就用VVCard进行注册，和registerCard完全一致。注册Cell时和registerCell就有一定区别，registerVirtualView不会提供具体viewClz，不会将type<->viewClz的关系保存到MVHelper中。的两个函数实现中BaseCellBinder不一致。

###### 数据渲染

调用TangramEngine.setData来渲染数据。Tangram内部都是用RecycleView承载视图，这个setData内部其实就是Adapter的setData。

setData中会调用mDataParser.parseGroup，返回值为List<C> cards。我们可以将其理解为一个元素树，描述了每个Card/Cell的属性、样式、位置信息等。对于parseGroup函数框架内部实现了PojoDataParser。简单的来说就是解析Json，根据我们注册的模板（CardResolver、MVHelper）构建出元素树。

* 我们根据CardResolver保存的type<->Class的关系，通过反射创建出对于的Card对象。如果不存在就使用WrapCellCard，WrapCellCard是单Cell的card，只会显示第一个Cell。
* Card对象确认后，调用parseWith函数解析JSON数据，除了Card本身属性的赋值（包括对Style的解析）外还会解析内部Cell。
* 如果设置了“header“，解析构建header对应的Cell添加到Card中。但并不是所有的Card都支持header，Card基类本身没有实现parseHeaderCell函数，部分内置Card（例如BannerCard有实现）。对于自定义Card,由于在注册模板时使用VVCard类，也没有parseHeaderCell函数。当然我们可以在注册时使用自己实现的一个Card子类。
* 根据card对于JSON中的”items“数组，循环调用createCell函数创建Cell。根据是否指定了viewClz(通过registerCell注册)、是否是Card、是否是VirtualView（通过registerVirtualView注册）有不同的创建过程，当最终都会调用resolver.parseCell(cell, data);进行属性赋值和addCellInternal(cell, false)添加到parent（Card中）
* 如果设置了“footer“，解析构建footer对应的Cell添加到Card中。逻辑和header就完全一样。

至此我们将已经将数据JSON完全解析完成保存为List< Card >这个元素树中。同时这个list也是RecycleView的数据源，会设置到Adapter中。


视图控件树的构建，依托于vlayout，需要对vlayout有所了解。简单的我们可以通过使用DelegateAdapter的方式了解，vlayout提供了DelegateAdapter类来创建Item，DelegateAdapter是一个聚合的Adapter，可以通过add函数添加很多子Adapter（DelegateAdapter.Adapter）。通过数据的类型使用不同的DelegateAdapter.Adapter，即可实现不同布局。但Tangram是通过自定义VirtualLayoutAdapter子类（GroupBasicAdapter）的方式实现的。

###### 构建VirtualView



###### 事件处理


### Tangram-Flutter

### 模板数据解析

我们要做byte数据解析，在java里读取的数据直接是byte[]，而在dart中对应的是Uint8List类型。