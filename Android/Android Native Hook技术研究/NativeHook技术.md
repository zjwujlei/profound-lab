NativeHook技术
=============


### sophix运行时方法替换方案

native层对方法进行整体替换。

``` C++
    jmethodID origin_pointer = env->FromReflectedMethod(origin);
    jmethodID hook_pointer = env->FromReflectedMethod(hook);
    memcpy(origin_pointer,hook_pointer,ART_METHOD_SIZE);
```

通过同一类中唯二函数指针的查值计算ART_METHOD_SIZE。

``` C++
    jmethodID id1 = env->FromReflectedMethod(method1);
    jmethodID id2 = env->FromReflectedMethod(method2);
    ART_METHOD_SIZE = (size_t)((jlong)id2-(jlong)id1);
```


#### hook场景

类结构无法变更

#### 热修复场景


