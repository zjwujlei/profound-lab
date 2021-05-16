kotlin、协程语法及实现
================
### Kotlin常用语法及其实现原理

###### kt文件

和java不同，我们可以直接在KT文件中定义变量(var、val、const val)和函数。
```
var NAME = "NAME"
val NAME1 = "NAME"
const val NAME2 = "NAME"


fun test(){
    Test.name == NAME
}

//kt代码

//字节码 class打开
public final class DomeKTKt {
   @NotNull
   private static String NAME = "NAME";
   @NotNull
   private static final String NAME1 = "NAME";
   @NotNull
   public static final String NAME2 = "NAME";

   @NotNull
   public static final String getNAME() {
      return NAME;
   }

   public static final void setNAME(@NotNull String var0) {
      Intrinsics.checkNotNullParameter(var0, "<set-?>");
      NAME = var0;
   }

   @NotNull
   public static final String getNAME1() {
      return NAME1;
   }

   public static final void test() {
      Intrinsics.areEqual(Test.Companion.getName(), NAME);
   }
}
```

* 在编译的时候会根据kt文件名创建一个类，同时定义静态的属性和函数。
* 对于var和val，生成的都是private变量，同时提供get函数调用。kt中可以直接使用NAME，当在编译之后其实是调用get函数的。
* 对于const val函数，定义的是public static final变量。可以直接使用，在编译后会触发内联，直接使用的值。

###### companion object伴生对象
>companion object 的出现是为了解决 Java static 方法的反面向对象（Anti-OOP）的问题。static 方法无法声明为接口，无法被重写——用更学术的话来说，static 方法没有面向对象的消息传递和延迟绑定特性[参考]。而 Scala 为了完成一切皆对象的使命，以及提高与 Java 的兼容性，提出了伴生对象这个概念来代替 static 方法。随后出身的 Kotlin 也借鉴了这个概念。
>companion 伴生对象是一个对象，它在类初始化时被实例化。 因为伴生对象不再是类的 static 方法，而是某个类的实例化对象，所以它可以声明接口，里面的方法也可以被重写，具备面向对象的所有特点。


```java
//kt代码
class Test (){
    companion object{
        val name:String = "name"
        fun test(){
            val test = name
        }
    }
}

class Test1(){
    fun test(){
        Test.name
        Test.test()
    }
}
//字节码 class打开
public final class Test {
   @NotNull
   private static final String name = "name";
   public static final Test.Companion Companion = new Test.Companion((DefaultConstructorMarker)null);

   public static final class Companion {
      @NotNull
      public final String getName() {
         return Test.name;
      }

      public final void test() {
         String test = ((Test.Companion)this).getName();
      }

      private Companion() {
      }

      // $FF: synthetic method
      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
   }
}

public final class Test1 {
   public final void test() {
      Test.Companion.getName();
      Test.Companion.test();
   }
}


```


* 编译时生成一个名为Test.Companion的静态内部。
* 如果定义了变量，会在Test中定义一个静态变量name，同时在伴生类Test.Companion中生成get函数，如果定义了函数只会在Test.Companion中定义函数。
* 特别的对于变量，对于的变量是在Test中定义的，Test.Companion中定义的get函数会返回Test.name。
* 虽然会在Test中生成name的静态变量，当其他类使用变量时，不是直接使用Test.name，而是Test.Companion.getName()，但如果是在Test类内部，是可以直接使用的。

###### 默认传参
kotlin的函数可以重载，但我们更建议用默认参数的形式来处理需要函数重载的情况，例如构造函数。
```java
//kt代码
class Test (){
    fun test1(){
        val dkt1 = DomeKT("name")
    }
    fun test2(){
        val dkt2 = DomeKT("name","15")
    }
}

class DomeKT (val name:String,val age:String = "10"){
}

//字节码 class打开

public final class Test {
   public final void test1() {
      new DomeKT("name", (String)null, 2, (DefaultConstructorMarker)null);
   }

   public final void test2() {
      new DomeKT("name", "15");
   }
}

public final class DomeKT {
   @NotNull
   private final String name;
   @NotNull
   private final String age;

   @NotNull
   public final String getName() {
      return this.name;
   }

   @NotNull
   public final String getAge() {
      return this.age;
   }

   public DomeKT(@NotNull String name, @NotNull String age) {
      Intrinsics.checkNotNullParameter(name, "name");
      Intrinsics.checkNotNullParameter(age, "age");
      super();
      this.name = name;
      this.age = age;
   }

   // $FF: synthetic method
   public DomeKT(String var1, String var2, int var3, DefaultConstructorMarker var4) {
      if ((var3 & 2) != 0) {
         var2 = "10";
      }

      this(var1, var2);
   }
}

```

生成两个构造函数
>   public DomeKT(@NotNull String name, @NotNull String age) {}
>   public DomeKT(String var1, String var2, int var3, DefaultConstructorMarker var4) {}

* 对于第二个构造函数，String var1, String var2即为要求的name和age。

* int var3,是默认参数是否有传的标记。对于函数入参，根据在声明中的位置，按顺序分别对应1（1），2（10），4（100），8（1000）... 然后对于传入的var3进行位运算确定外部调用是否有传，没传入赋初始值。

* int var3的传值有编译器在编译的时候确认，例如示例中传了2。

###### apply、let功能函数
```java
//kt代码
DomeKT2("apply").let {
    val (name) = it
    println(name)
}
//字节码 class打开
public final void test1() {
    DomeKT2 var1 = new DomeKT2("apply");
    boolean var2 = false;
    boolean var3 = false;
    int var5 = false;
    String name = var1.component1();
    boolean var6 = false;
    System.out.println(name);
}

```


apply和let都是在编译时new一个对象，然后调用其函数。
###### 扩展函数

在java中，例如要进行px-dp单位转换，我们基本是通过一个Utils类来实现。有了扩展函数的概念后，我们可以将px-dp单位转换的能力绑定到Integer中，就好像内置功能一样方便调用。

```java
//kt代码
fun Int.toPX():Int{
   return  this * 2
}

class Demo2{
    fun test(){
        10.toPX()
    }
}

//字节码 class打开
public final class Demo2 {
   public final void test() {
      DemoKTKt.toPX(10);
   }
}

public final class DemoKTKt {
   public static final int toPX(int $this$toPX) {
      return $this$toPX * 2;
   }
}
```

可以看到根据文件名生成了一个DemoKTKt的类，扩展函数被生成未一个静态函数，入参一个被扩展类对象。调用点代码被编译后都是直接调用这个静态函数。通过这种方式好似”无中生有“了一个函数。

>扩展函数是静态分发的，即他们不是根据接收者类型的虚方法。 这意味着调用的扩展函数是由函数调用所在的表达式的类型来决定的， 而不是由表达式运行时求值结果决定的。

意思就是对于扩展函数不具备多态的性质。简单点说就是，将子类对象赋值给父类引用，调用扩展函数时调用的是父类的扩展函数

###### 扩展属性

可以给现有类添加一个变量。支持添加var、和val。如果是val无法提供set函数。

```java
//kt代码
var String.last:String?
    get() {
        return this.lastOrNull()?.toString()
    }
    set(value) {
        this.substring(0,this.length-1)+value
    }
//字节码 class打开
public final class DemoKTKt {
   @Nullable
   public static final String getLast(@NotNull String $this$last) {
      Intrinsics.checkNotNullParameter($this$last, "$this$last");
      Character var10000 = StringsKt.lastOrNull((CharSequence)$this$last);
      return var10000 != null ? String.valueOf(var10000) : null;
   }

   public static final void setLast(@NotNull String $this$last, @Nullable String value) {
      Intrinsics.checkNotNullParameter($this$last, "$this$last");
      StringBuilder var10000 = new StringBuilder();
      byte var3 = 0;
      int var4 = $this$last.length() - 1;
      StringBuilder var6 = var10000;
      boolean var5 = false;
      String var8 = $this$last.substring(var3, var4);
      Intrinsics.checkNotNullExpressionValue(var8, "(this as java.lang.Strin…ing(startIndex, endIndex)");
      String var7 = var8;
      var6.append(var7).append(value).toString();
   }
}
```

* 特别的与扩展函数的”无中生有“不同，扩展属性是没有幕后字段（field）的，无法像其他属性定义一样使用field。
* 对于变量无法提供初始化器。
* 对于扩展属性，编译后提供了静态get、set函数进行调用。和直接定义的属性不同，无法直接通过属性使用,都是通过这个静态get、set函数进行的。
* 扩展属性可以理解为：并不是对类扩展一个属性，而是对现有属性的扩展。

###### 操作符
通过重载操作符，可以实现任意对象的+、-、*等操作。操作符的重载最好能够体现业务，在业务上两个对象有明确相加的含义的情况下使用操作符+对相加逻辑进行封装，是可以有效的提升代码开发效率的同事保证其可读性。

```kt
//kt代码
class DemoKT {
    public operator fun plus (value:DemoKT ){
       
    }
}

fun test(){
    var d1 = DemoKT()
    val d2 = DemoKT()
    d1 + d2
}

//字节码 class打开
public final class DemoKT {
   public final void plus(@NotNull DemoKT value) {
      Intrinsics.checkNotNullParameter(value, "value");
   }
}

public final class DemoKTKt {

   public static final void test() {
      DemoKT d1 = new DemoKT();
      DemoKT d2 = new DemoKT();
      d1.plus(d2);
   }
}

```

* 操作符的重载也可以用扩展函数定义。
* 每个操作符对于的重载函数是一一对应的。例如要实现d1 += d2，可以重载plusAssign函数。具体可参阅 <a href="https://www.kotlincn.net/docs/reference/operator-overloading.html">操作符重载</a>
* 不允许自定义操作符

###### 中缀调用 infix
对于操作符无法自定义。但我们可以通过infix实现类似的效果。infix是指中缀调用，例如 Int的and、or等位运算函数等。
```kt
//kt代码
class DemoKT {
    public infix fun merge (value:DemoKT ){
    }
}

fun test(){
    var d1 = DemoKT()
    val d2 = DemoKT()
    d1 merge d2
}

//字节码 class打开

public final class DemoKT {
    public final void merge(@NotNull DemoKT value) {
      Intrinsics.checkNotNullParameter(value, "value");
    }
}

public static final void test() {
    DemoKT d1 = new DemoKT();
    DemoKT d2 = new DemoKT();
    d1.merge(d2);
}
```

* 使用中缀修饰的函数并不是操作符，只是调用形式类似操作符。
* 中缀调用是一个很有意思的知识点，通过中缀调用甚至可以实现 一句话即为一段函数调用。
```kotlin
class DemoKT {
    public infix fun decorate(str:String):DemoKT{
        return this
    }
    
    public infix fun with(str:String):DemoKT{
        return this
    }
    
    fun test(){
        this decorate "abc" with "def"
    }
}
```

###### 解构

解构可以用来快速赋值，可以用来变相实现一个返回多个值，函数返回一个支持解构的对象，然后进行解构赋值。

```
//kt代码
class DemoKT(val name:String,val age:Int) {
    operator fun component1():String{
        return name
    }
    operator fun component2():Int{
        return age
    }
}

fun test(){
    var (name,age) = DemoKT("name",10)

}

//字节码 class打开

public static final void test() {
    DemoKT var2 = new DemoKT("name", 10);
    String var0 = var2.component1();
    int age = var2.component2();
}

public final class DemoKT {
   @NotNull
   private final String name;
   private final int age;

   @NotNull
   public final String component1() {
      return this.name;
   }

   public final int component2() {
      return this.age;
   }

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final int getAge() {
      return this.age;
   }

   public DemoKT(@NotNull String name, int age) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.name = name;
      this.age = age;
   }
}
```

* data 对象本身就支持解构。
* 解构赋值变量的顺序和component1、component2、...的顺序有关。
* 解构赋值是必须按照component1、component2、...的顺序的顺序赋值，如果有不需要赋值的，用"_"下划线代替

###### 委托 by

>委托模式是软件设计模式中的一项基本技巧。在委托模式中，有两个对象参与处理同一个请求，接受请求的对象将请求委托给另一个对象来处理。
Kotlin 直接支持委托模式，更加优雅，简洁。Kotlin 通过关键字 by 实现委托。

委托有类委托，属性委托等。对于类委托很简单，只要在类定义是通过by字段委托
```

```
对于属性委托kotlin 内置了lazy，Observable等委托实现，我们常用的lazy的实现如下：

```
//kotlin代码
fun main(){ // this: CoroutineScope
    val demo = DemoMain()
    println(demo.lx)
    println(demo.lx)
    println(demo.a)
    println(demo.b)
}

class DemoMain{
    var a = 0
    var b = 0
    val lx by lazy {
        ++a
        ++b
    }
}
//输出结果
init
lx:1
lx:1
a:1
b:1
//字节码 class打开
public final class DemoMain {
   private int a;
   private int b;
   @NotNull
   private final Lazy lx$delegate = LazyKt.lazy((Function0)(new Function0() {
      // $FF: synthetic method
      // $FF: bridge method
      public Object invoke() {
         return this.invoke();
      }

      public final int invoke() {
         DemoMain var10000 = DemoMain.this;
         var10000.setA(var10000.getA() + 1);
         var10000.getA();
         var10000 = DemoMain.this;
         var10000.setB(var10000.getB() + 1);
         return var10000.getB();
      }
   }));

   public final int getA() {
      return this.a;
   }

   public final void setA(int var1) {
      this.a = var1;
   }

   public final int getB() {
      return this.b;
   }

   public final void setB(int var1) {
      this.b = var1;
   }

   public final int getLx() {
      Lazy var1 = this.lx$delegate;
      Object var3 = null;
      boolean var4 = false;
      return ((Number)var1.getValue()).intValue();
   }
}

public static final void main() {
    DemoMain demo = new DemoMain();
    int var1 = demo.getLx();
    boolean var2 = false;
    System.out.println(var1);
    var1 = demo.getLx();
    var2 = false;
    System.out.println(var1);
    var1 = demo.getA();
    var2 = false;
    System.out.println(var1);
    var1 = demo.getB();
    var2 = false;
    System.out.println(var1);
 }

```


我们可以看到，对于by lazy定义的变量，在Java 字节码中是转化成一个Lazy对象，这个就是被委托处理的对象，具体实现可以看SynchronizedLazyImpl。处理也简单，对于value提供get函数，只有在首次使用value的时候通过传入的Function0进行初始化。Function0里包子的就是我们在by lazy时设置的闭包。再次使用value是，因为不是UNINITIALIZED_VALUE值，就可以直接使用了。

这里的DemoMain就是接受请求的对象。我们在Kotlin代码中定义的是lx变量，在Java字节码中提供getLX函数，在调用点都会替换成getLX。这个点在直接使用变量时都适用，都会替换成get函数。DemoMain的getLx()函数接受到请求后，再调用委托类获取数据。

其他委托实现Observable、NotNull等实现基本类似。


当然我们也可以自定义委托实现，通过操作符函数provideDelegate来自定义属性委托
```
fun demoMain(str:String):DemoMain{
    return DemoMain(str)
}
class Main {
    val demo by demoMain("1234")
}

class DemoMain(var str:String){
    operator fun provideDelegate(ref:Main?,property:KProperty<*>):ReadOnlyProperty<Main,String>{
        return object : ReadOnlyProperty<Main, String> {
            override fun getValue(thisRef: Main, property: KProperty<*>): String {
                return str
            }
        }
    }
}
```

### 协程

首先协程并不是线程，认为协程是Kotlin下的线程是不对的。线程是调度单元，协程可以任务是具体的执行任务。

```
//kt代码
fun main() = runBlocking { // this: CoroutineScope
    launch {
        delay(200L)
        println("Task from runBlocking")
    }

    println("Task in runBlocking")
}
//class文件打开
public final class MainKt {
   public static final void main() {
      BuildersKt.runBlocking$default((CoroutineContext)null, (Function2)(new Function2((Continuation)null) {
         private CoroutineScope p$;
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
            switch(this.label) {
            case 0:
               ResultKt.throwOnFailure($result);
               CoroutineScope $this$runBlocking = this.p$;
               BuildersKt.launch$default($this$runBlocking, (CoroutineContext)null, (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
                  private CoroutineScope p$;
                  Object L$0;
                  int label;

                  @Nullable
                  public final Object invokeSuspend(@NotNull Object $result) {
                     Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
                     CoroutineScope $this$launch;
                     switch(this.label) {
                     case 0:
                        ResultKt.throwOnFailure($result);
                        $this$launch = this.p$;
                        this.L$0 = $this$launch;
                        this.label = 1;
                        if (DelayKt.delay(200L, this) == var5) {
                           return var5;
                        }
                        break;
                     case 1:
                        $this$launch = (CoroutineScope)this.L$0;
                        ResultKt.throwOnFailure($result);
                        break;
                     default:
                        throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
                     }

                     String var3 = "Task from runBlocking";
                     boolean var4 = false;
                     System.out.println(var3);
                     return Unit.INSTANCE;
                  }

                  @NotNull
                  public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
                     Intrinsics.checkParameterIsNotNull(completion, "completion");
                     Function2 var3 = new <anonymous constructor>(completion);
                     var3.p$ = (CoroutineScope)value;
                     return var3;
                  }

                  public final Object invoke(Object var1, Object var2) {
                     return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
                  }
               }), 3, (Object)null);
               String var3 = "Task in runBlocking";
               boolean var4 = false;
               System.out.println(var3);
               return Unit.INSTANCE;
            default:
               throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }
         }

         @NotNull
         public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
            Intrinsics.checkParameterIsNotNull(completion, "completion");
            Function2 var3 = new <anonymous constructor>(completion);
            var3.p$ = (CoroutineScope)value;
            return var3;
         }

         public final Object invoke(Object var1, Object var2) {
            return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
         }
      }), 1, (Object)null);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }
}
```

可以看到并没有线程切换。通过launch等个创建一个协程时，实际上是BuildersKt.runBlocking$default()函数，将协程内执行代码包装成Function2。

我们看下BuildersKt的实现，在kotlinx-coroutines-core-jvm-1.3.9.jar中。这里有个点，在AS中直接打开BuildersKt没法看到所有代码。这个是反编译工具的问题。这里可以通过dadx-gui打开。也可以去github上个查看源码。


###### 作用域CoroutineScope
作用域，用来创建和管理一定范围内的协程。协程构建相关函数（launch、async等）都是CoroutineScope的扩展函数，需要通过CoroutineScope调用。在CoroutineScope中创建的协程，都会继承CoroutineScope的上下文（CoroutineContext），基于相同的上下文（并不是通一个实例，是基于代理的实现功能一致的CoroutineContext），可以实现对所有子协程的管理。

###### 上下文CoroutineContext

CoroutineContext在协程创建时创建，每个协程都有自己CoroutineContext对象，用于持有当前协程的一些列配置、协程全局对象等。

在理解上，我们可以认为CoroutineContext本身不提供协程处理的能力，具体能力通过Element提供（虽然Element声明了CoroutineContext接口）。CoroutineContext定义了get操作符函数，同时实现了plus操作室函数。因此我们可以通过CoroutineContext[Key]来获取类型为Element子类的实例，同时我们可以通过CoroutineContext + CoroutineContext的方式来添加CoroutineContext。

通过这两个函数，我们使CoroutineContext具有了Map集合的概念，一个CoroutineContext中可以包含多个CoroutineContext。如果让我们自己来实现，我们可能直接Map数据结构来实现，但为了更加高效，协程内对这个抽象集合通过get、plus操作符和单向链表来实现。其中链表对象定义为CombinedContext，拥有left和element，其中left为CoroutineContext，element为Element。

我们最常用的CoroutineContext有CoroutineDispatcher、Job。

CoroutineDispatcher用来指明当前协程运行的线程情况，通过Dispatchers获取：
* Dispatchers.Default : 默认实现，底层实现了一个线程池，每次都会基于这个线程池获取一个新的线程来执行协程。
* Dispatchers.Main : 在Android平台上所有使用Dispatchers.Main的都会在UI主线程里执行，当然只有一个线程。
* Dispatchers.Unconfined : 不做任何线程切换，在单前线程执行。
* Dispatchers.IO : 底层实现了一个专门做IO处理线程的线程池，将协程跑在专门的IO线程中。

特别的，Dispatchers.Default的默认是对CoroutineScope来说的。对于一个协程构建器函数（launch等），如果不设置时不做线程切换使用的是所在的CoroutineScope中指定的线程（类似于Dispatchers.Unconfined）。

Job是真正在协程中执行的任务（类似于线程中的Runnable），拥有start、join、cancel等函数。我们使用协程构建器函数（launch等）时就返回一个Job。Job是有状态的
```java
>  * A job has the following states:
>  *
>  * | **State**                        | [isActive] | [isCompleted] | [isCancelled] |
>  * | -------------------------------- | ---------- | ------------- | ------------- |
>  * | _New_ (optional initial state)   | `false`    | `false`       | `false`       |
>  * | _Active_ (default initial state) | `true`     | `false`       | `false`       |
>  * | _Completing_ (transient state)   | `true`     | `false`       | `false`       |
>  * | _Cancelling_ (transient state)   | `false`    | `false`       | `true`        |
>  * | _Cancelled_ (final state)        | `false`    | `true`        | `true`        |
>  * | _Completed_ (final state)        | `false`    | `true`        | `false`       |

> * 
> *                                       wait children
> * +-----+ start  +--------+ complete   +-------------+  finish  +-----------+
> * | New | -----> | Active | ---------> | Completing  | -------> | Completed |
> * +-----+        +--------+            +-------------+          +-----------+
> *                  |  cancel / fail       |
> *                  |     +----------------+
> *                  |     |
> *                  V     V
> *              +------------+                           finish  +-----------+
> *              | Cancelling | --------------------------------> | Cancelled |
> *              +------------+                                   +-----------+
> * 
```

我们可以通过CoroutineScope来取消该作用域下的所有协程，这个就是通过job实现的。使用构建器函数（launch等）创建一个协程时，会将该协程的job添加到父Job中，函数调用链如下：
```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
    initParentJob()
    start(block, receiver, this)
}

internal fun initParentJob() {
    initParentJobInternal(parentContext[Job])
}
internal fun initParentJobInternal(parent: Job?) {
    assert { parentHandle == null }
    if (parent == null) {
        parentHandle = NonDisposableHandle
        return
    }
    parent.start() // make sure the parent is started
    @Suppress("DEPRECATION")
    val handle = parent.attachChild(this)
    parentHandle = handle
    // now check our state _after_ registering (see tryFinalizeSimpleState order of actions)
    if (isCompleted) {
        handle.dispose()
        parentHandle = NonDisposableHandle // release it just in case, to aid GC
    }
}

```

构建了这个Job关系后，调用CoroutineScope的cancel后就能取消所有子协程。

### Flow
Flow 就是 Kotlin 协程与响应式编程模型结合的产物。和RxJava实现的功能非常像。
###### 冷流
要创建一个Flow，可以用个简单的flow函数创建。

```kotlin
val intFlow = flow {
  println("flow")
    (1..3).forEach {
        emit(it)
        delay(100)
    }
}
intFlow.collect { 
    println(it)
}
intFlow.collect { 
    println(it)
}

```

这样创建的都是冷流，对于冷流，一个使用场景是定时器。通过冷流实现的定时器可以多次复用，每次调用collect都会重新产生数据。

这里有一点，在多次调用collect时，如果过前一次flow的数据还没emit完成，既还在工作状态，如果我们再次调用collect，是collect前一个为发送完成的flow还是一个新的flow呢？这里答案是一个新的flow。我们基于协程将两个collect并发，println("flow")也会打印两次。

这里我们可以从collect函数中确认答案：
```kotlin
//collect.kt
public suspend inline fun <T> Flow<T>.collect(crossinline action: suspend (value: T) -> Unit): Unit =
    collect(object : FlowCollector<T> {
      override suspend fun emit(value: T) = action(value)
  })

//Flow.kt
 @FlowPreview
public abstract class AbstractFlow<T> : Flow<T> {

    @InternalCoroutinesApi
    public final override suspend fun collect(collector: FlowCollector<T>) {
        val safeCollector = SafeCollector(collector, coroutineContext)
        try {
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }

  
    public abstract suspend fun collectSafely(collector: FlowCollector<T>)
} 

//SafeCollector.kt
private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```

最终collector.block()调用就是执行block函数的意思。我们用flow函数创建流时传入的也是FlowCollector，整个flow相当于多个FlowCollector的串联，由上游发送数据，下流处理输入，如果还有下游的话继续传递。

我们使用RxJava的时候有操作符的概念，Flow同样有，这里看下常用的Map操作符：

```
public inline fun <T, R> Flow<T>.map(crossinline transform: suspend (value: T) -> R): Flow<R> = transform { value ->
   return@transform emit(transform(value))
}

```

这里的transform在源码中跟下去的话也是FlowCollector。这里也和前面说的上游、下游的情况对应。

###### 热流

既然有冷流，就会有对应的热流。热流的话就是哪怕没有使用collect，也会发送数据。使用热流对优化我们业务代码很有帮助。特别是在异步的场景，我们可以不使用回调，通过suspendCancellableCoroutine（触发一次后就作废）和callbackFlow（能一直发送数据）将异步功能封装成流使用。

例如callbackFlow，一般的用法如下：
```
suspend fun initFlow() = callbackFlow<String> {
    
    //创建监听
    val listener = object : Listener<String> {
        override fun onSuccess(result: String) {
            //发送数据
            offer(result)
        }
    }
    //异步功能注册回调
    SDK.registerListener(listener)
    awaitClose {
        //销毁，取消监听等处理
    }
}

```

initFlow返回的是一个Flow对象，后续的操作符、collect等都和冷流一致。


###### 线程调度
在RxJava中，subscribeOn指定的调度器影响前面的预约逻辑线程，observeOn影响的是后面的观察逻辑，在kotlin的flow中，线程的调度使用flowOn来切换上游执行的的线程，相当于subscribeOn。collect函数调用所在的额协程对于的线程就是观察逻辑线程，相当于observeOn。


###### 背压
背压是指消费者处理消息的速率低于生产者产生消息的速率的情况。这里好比我们的页面的掉帧，系统屏幕硬件发出的vsync就是生成者生产消息，CPU+GPU处理排版绘制上屏逻辑就是消费者，当排版绘制上屏逻辑逻辑耗时草果16ms，就会形成背压导致掉帧。

解决背压我们有三种方式可以选择。

第一种是通过缓冲来解决背压，
```
flow {
    
    (1..3).forEach {
        emit(it)
        delay(100)
    }
}.buffer().collect {
    println(it)
}
```
通过buffer()创建缓冲区，同时可以设定缓冲区大小。当这种方式只能缓解和延迟问题的爆发，本质上没有解决问题。

第二种是使用conflate函数，意思是只处理最新的消息。当前一个消息还没被消费者消费完成时，后面多次接受到的消息还会进行覆盖。想过就是好像是跳过了中间消息一样。

第三种是使用collectLatest函数，也是只处理最新的消息，但区别是收到新消息是，消费者会进行中断当前的消费，快速进行下一个消息的处理。这里的中断是基于挂起（suspend）函数的，例如delay等，并不是任何地方都能够进行中断。

当然这些方法都是取巧或者补救的措施，真正能解决背压是优化消费者的消费速率，不产生背压问题。


### 协程异常处理
协程中，我们可以正常使用try-catch捕获异常，也可以使用runCatching、catch等函数处理异常。

>协程构建器有两种形式：自动传播异常（launch 与 actor）或向用户暴露异常（async 与 produce）。 当这些构建器用于创建一个根协程时，即该协程不是另一个协程的子协程， 前者这类构建器将异常视为未捕获异常，类似 Java 的 Thread.uncaughtExceptionHandler， 而后者则依赖用户来最终消费异常，例如通过 await 或 receive（produce 与 receive 的相关内容包含于通道章节）。

###### CoroutineExceptionHandler
Java中有Thread.uncaughtExceptionHandler来做全局的异常捕获，协助中也可以时候用类似的实现。CoroutineExceptionHandler时候一个CoroutineContextCoroutineContext，我们在创建CoroutineScope的时候可以进行设置，这样所有未被捕获的异常都可以处理。

###### CancellationException
当我们取消协程是，协程内部使用CancellationException来进行取消，理论上我们不应该处理这个异常，否则可能引起CoroutineScope下协程的异常。


###### CoroutineScope下巧用异常

一般情况下创建的CoroutineScope，取消是在协程的整个层次结构中传播的双向关系。有一个子协程发生异常的时候，会引起整个CoroutineScope的取消，因此包括父协程，兄弟协程都会取消。对于相互关联较大的大型功能下的小项功能，可以如此使用。

当我们要实现单向传播时，可以使用SupervisorJob和supervisorScope。相对于Job，SupervisorJob 的取消只会向下传播。supervisorScope 是用来替代coroutineScope ，它只会单向的传播并且当作业自身执行失败的时候将所有子作业全部取消。作业自身也会在所有的子作业执行结束前等待， 就像 coroutineScope 所做的那样。

