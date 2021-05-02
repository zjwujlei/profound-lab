ScriptEngine-Android动态逻辑JS下发实现
==========================

在Java1.6之后提供了ScriptEngine类可以用来运行脚本语言。我们这里时候用1.8之后最新的实现，来实现下使用JavaScript动态下发逻辑。


### ScriptEngine使用说明

使用ScriptEngine-Java执行JS代码本身是否简单。

```kotlin
	val engine = ScriptEngineManager().getEngineByName("javascript")

    val invoke = engine as Invocable

    fun execute(){
        engine.eval("function sayName(){return \"Hello Java!\";}");
        println("----->${invoke.invokeFunction("sayName",null).toString()}")
    }

```

拿到engine对象后可以调用eval函数加载JS代码，然后使用invokeFunction函数执行JS内定义的函数。

对于复杂一些的情况我们可以通过binding向JS传递原生变量、可以通过get函数获取JS的变量、可以将JS返回的变量强转为ScriptObjectMirror进行使用。

特别的，在Android中没法直接使用javax.script中的代码。我们需要通过额外导入库实现。


### 

