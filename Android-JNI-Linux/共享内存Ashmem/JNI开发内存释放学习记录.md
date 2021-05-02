JNI开发内存释放学习笔记
====================

### c++内存相关
静态存储区域：内存在程序编译的时候就已经分配好，这块内存在程序的整个运行期间都存在。例如全局变量，static变量。
栈：函数调用过程中产生的本地变量和调用数据都可以在栈中分配，包括函数入参。栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。
堆：动态分配的内存。用new，malloc等分配内存，可以分配任意多的内存，但需要手动释放。在手动释放后建议立即将指针置位NULL，避免产生野指针。
###### malloc
malloc是c++中的内存分配函数，在进程堆中动态分配内存。需要调用free函数进行释放。
```
char *result = static_cast<char *>(malloc(1024));
free(result); //调用free释放
```

调用malloc函数后，系统会在内存空闲内存链表中寻找一块足以满足申请大小的区域，将区域分为两块，一块等于申请大小并将内存地址返回，一块是剩余区域拼接会空闲内存链表中。调用free时也是将内存区域返回空闲内存链表中。但空闲内存链表中的内存块无法满足分配大小时就会触发GC，进行内存整理。

###### new 
new 关键只创建的是对象，也是在在进程堆中动态分配的内存。对于对象需要调用delete函数进行释放。
```
Obj* obj = new Obj("123");
obj->say();
delete obj; //删除对象
```

###### 栈中分配内存
函数中的局部变量是在栈中分配的内存，当退出这个函数作用域是，会进行*栈展开*操作。栈展开是会自动调用析构函数。因此局部变量不需要释放。
```
Obj obj("234");//局部变量在栈上申请内存。这个时候obj是引用并不是指针。
obj.say();
```

由于存在栈张开自动释放内存，我们不能将局部变量当做函数返回值。

### JNI相关

###### 基础类型，jint、jchar、jlong等
JNI 基本数据类型是不需要释放的

###### 引用类型 string
```c++
jclass context_class = env->GetObjectClass(ctx);
    // step 1：获取packageName。context.getPackageName()
    jmethodID get_package_name = env->GetMethodID(context_class, "getPackageName",
            "()Ljava/lang/String;");
auto package_name = (jstring)(env->CallObjectMethod(ctx, get_package_name));
const char* package_name_chars = env->GetStringUTFChars(package_name,0);

env->ReleaseStringUTFChars(package_name,package_name_chars);
env->DeleteLocalRef(package_name);

```
调用ReleaseStringUTFChars释放内存，同时调用DeleteLocalRef删除引用。

###### 数组相关，jcharArray、jbyteArray等
```c++
auto chars_array = (jcharArray) env->CallObjectMethod(signature, to_chars);
env->DeleteLocalRef(chars_array);
```

调用DeleteLocalRef删除引用

###### 对象数组相关 GetCharArrayElements、GetByteArrayElements等

```c++
auto chars_array = (jcharArray) env->CallObjectMethod(signature, to_chars);
jchar *jchars = env->GetCharArrayElements(chars_array, 0);

env->ReleaseCharArrayElements(chars_array,jchars,0);
env->DeleteLocalRef(chars_array);
```

调用ReleaseCharArrayElements释放，调用DeleteLocalRef删除。

###### 对象相关，jobject、jclass等
```c++
jclass context_class = env->GetObjectClass(ctx);
env->DeleteLocalRef(context_class);
```

调用DeleteLocalRef删除引用

###### jmethodId

直接用DeleteLocalRef进行删除编译报错，错误如下：
```
Android jni Cannot initialize a parameter of type 'jobject' (aka '_jobject *') with an lvalue of type 'jmethodID' (aka '_jmethodID *')
```

可以看到jmenthodID是左值，在栈空间分配内存会在栈展开的时候析构，猜测不用手动释放。查看网上的资料如下：

>DeleteLocalRef 一般不需要手动调用，shared_ptr<T> 对象（jni的引用，_jobject _jmethodid是其具体实现）出作用域，就会析构，同时--refcount.除非你这个函数会执行很久。
>对于FindClass 返回的一定需要调用DeleteLocalRef，还有jbyteArray 类型的变量需要DeleteLocalRef。
NewString/ NewStringUTF/NewObject/ GetObjectField生成的需要DeleteLocalRef。
以上返回的类型变量是malloc出来的，不是栈变量。出作用域不会被释放，需要手动。设计的真是令人发指！

