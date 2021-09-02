kotlin 反射
==========

java中反射需要获取使用java.lang.reflect包下的类，对于kotlin，反射只需要使用到"::"。

### 类引用 KClass

```
//kotlin, 
val c:KClass<MyClass> = MyClass::class
//java
Class c = MyClass.class;
```

Java中有Class.fromName()的函数来获取类的引用。kotlin中没有直接的函数获取，但是可以通过Class.fromName().kotlin获得。
```
Class.forName("MyClass").kotlin
```

### 函数引用 KFunction

```
fun main() {
    
    fun isGood(x:Int):Boolean = x%2 == 0

    fun check(checker:(x:Int)->Boolean):Boolean{
        return  checker(listOf(1,2,3,4).random())
    }

    check(::isGood)
}
```

通过“::functionName”获取函数引用，对应的是KFunction。kotlin使用闭包会很方便开发，对于闭包的函数就可以通过函数引用来使用，

在Java中，去的Java类引用后可以通过getDeclaredMethod()函数根据不同的函数签名来获取对应函数。在kotlin中也可以通过kotlin的类引用来获取函数。

```
val myclass = Class.forName("MyClass").kotlin
myclass.declaredFunctions.find {
    it.name == "abc"
}?.call()
```

### 扩展函数

### 属性引用 KProperty

```
class MyClass {

    val x = 1

    fun main(){
        ::x.name
        ::x.get()
    }
}
```

通过“::name”获取属性引用，对于的KProperty。在kotlin中同样也可以通过kotlin的类引用来获取。
