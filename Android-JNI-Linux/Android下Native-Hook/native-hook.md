## native-hook

### art_method

```c++
class ArtMethod FINAL { 
  protected:
  // Field order required by test "ValidateFieldOrderOfJavaCppUnionClasses".
  // The class we are a part of.
  //对于64位机器：offset=0，length=8.
  GcRoot<mirror::Class> declaring_class_;

  // Access flags; low 16 bits are defined by spec.
  // Getting and setting this flag needs to be atomic when concurrency is
  // possible, e.g. after this method's class is linked. Such as when setting
  // verifier flags and single-implementation flag.
  //对于64位机器：offset=8，length=4.
  std::atomic<std::uint32_t> access_flags_;

  /* Dex file fields. The defining dex file is available via declaring_class_->dex_cache_ */

  // Offset to the CodeItem.
  //对于64位机器：offset=12，length=4.
  uint32_t dex_code_item_offset_;

  // Index into method_ids of the dex file associated with this method.
  //对于64位机器：offset=16，length=4.
  uint32_t dex_method_index_;

  /* End of dex file fields. */

  // Entry within a dispatch table for this method. For static/direct methods the index is into
  // the declaringClass.directMethods, for virtual methods the vtable and for interface methods the
  // ifTable.
  //对于64位机器：offset=20，length=2.
  uint16_t method_index_;

  // The hotness we measure for this method. Not atomic, as we allow
  // missing increments: if the method is hot, we will see it eventually.
  //对于64位机器：offset=22，length=2.
  uint16_t hotness_count_;

  // Fake padding field gets inserted here.

  // Must be the last fields in the method.
  struct PtrSizedFields {
    // Depending on the method type, the data is
    //   - native method: pointer to the JNI function registered to this method
    //                    or a function to resolve the JNI function,
    //   - conflict method: ImtConflictTable,
    //   - abstract/interface method: the single-implementation if any,
    //   - proxy method: the original interface method or constructor,
    //   - other methods: the profiling data.
	//对于64位机器：offset=24，length=8.
    void* data_;

    // Method dispatch from quick compiled code invokes this pointer which may cause bridging into
    // the interpreter.
    //对于64位机器：offset=32，length=8.
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;
}
```

看的是Android 9.0下的ArtMethod结构，这里只列出了属性部分，所有public/private函数都进行了省略。其中GcRoot内部其实是一个指针。

在9.0系统上我们获取到的ART_METHOD_SIZE是28(后面函数替换部分，使用的是32为的armeabi-v7a。如果是64位的arm64-v8a的话是40，一个指针占8位。），与上面的结构定义也是匹配的。

### epic

对于epic，其实现原理可以通过查阅作者的<a href="http://weishu.me/2017/11/23/dexposed-on-art/">我为Dexposed续一秒——论ART上运行时 Method AOP实现</a>。对于经典的hook基本实现例如：

```kotlin
fun startHook(){
    DexposedBridge.findAndHookMethod(
        Class.forName("com.profound.epic.AppLogic"),
        "info",
        String::class.java,
        object : XC_MethodHook() {
            override fun afterHookedMethod(param: MethodHookParam?) {
                super.afterHookedMethod(param)
                println("afterHooked")
            }
        }

    )
}
```

核心就是调用框架内部的findAndHookMethod函数：

```java
public static XC_MethodHook.Unhook findAndHookMethod(Class<?> clazz, String methodName, Object... parameterTypesAndCallback) {
    ...
    Method m = XposedHelpers.findMethodExact(clazz, methodName, parameterTypesAndCallback);
    XC_MethodHook.Unhook unhook = hookMethod(m, callback);
    ...
}
```

核心就这两行代码，findMethodExact就是找到要hook的Method对象。parameterTypesAndCallback这个对象数组最后一个是XC_MethodHook对象，之前的都是参数类型对象。findMethodExact内部会生成一个key（ clazz.getName() + '#' + methodName + getParametersString(parameterTypes) + "#exact"），然后找到的Method对象作为value保存到methodCache中。然后调用hookMethod函数：

```java
public static XC_MethodHook.Unhook hookMethod(Member hookMethod, XC_MethodHook callback) {
    //Member是java反射中的元素统一接口。
    if (!(hookMethod instanceof Method) && !(hookMethod instanceof Constructor<?>)) {
        throw new IllegalArgumentException("only methods and constructors can be hooked");
    }

    boolean newMethod = false;
    //CopyOnWriteSortedSet是框架内部实现的一个线程安全的集合，内部用数组保存元素。
    //这里主要是处理如果同一个函数被多次hook的逻辑。
    //将所有的callbacks保存到CopyOnWriteSortedSet中并用hookedMethodCallbacks（HashMap）存储。
    CopyOnWriteSortedSet<XC_MethodHook> callbacks;
    synchronized (hookedMethodCallbacks) {
        callbacks = hookedMethodCallbacks.get(hookMethod);
        if (callbacks == null) {
            callbacks = new CopyOnWriteSortedSet<XC_MethodHook>();
            hookedMethodCallbacks.put(hookMethod, callbacks);
            newMethod = true;
        }
    }

    Logger.w(TAG, "hook: " + hookMethod + ", newMethod ? " + newMethod);

    callbacks.add(callback);
    if (newMethod) {
        //对于ART虚拟机，调用Epic.hookMethod进行hook。
        if (Runtime.isArt()) {
            if (hookMethod instanceof Method) {
                Epic.hookMethod(((Method) hookMethod));
            } else {
                Epic.hookMethod(((Constructor) hookMethod));
            }
        } else {
            //非ART虚拟机我们就不看了。
            ...
        }
    }
    return callback.new Unhook(hookMethod);
}
```

对于Epic.hookMethod函数，这里要讲一下ArtMethod，这个只是Epic的一个定义，内部存储了Member、真正Art_method对象的指针等信息。

```java
private static boolean hookMethod(ArtMethod artOrigin) {
	
    //用MethodInfo保存artOrigin信息并放在artOrigin map中。
    MethodInfo methodInfo = new MethodInfo();
    ...
    originSigs.put(artOrigin.getAddress(), methodInfo);

    if (!artOrigin.isAccessible()) {
        artOrigin.setAccessible(true);
    }
	//在原理介绍中有说道:
    //static函数是lazy resolve的，在方法没有被调用之前，static函数的入口地址是一个跳板函数，
    //名为 art_quick_resolution_trampoline，这个跳转函数做的事情就是去resvole原始函数，
    //然后进行真正的调用逻辑；因此没有被调用的static函数的entrypoint都是一样的。
    //ensureResolved做的就是如果是static函数，就进行一次调用。函数入参直接传null即可。
    artOrigin.ensureResolved();
	//上面处理了静态函数，在原理介绍中还有说道：
    //Android N以上，APK安装的时候，默认是不会触发AOT编译的；
    //因此如果刚安装完你去看apk生成的OAT文件，会发现里面的code都是空。
    //在这些方法被resolve的时候，如果ART发现code是空，会把entrypoint设置为解释执行的入口；
    //接下来如果此方法被执行会直接进入到解释器。所以，Android N上未被编译的所有方法入口地址都相同。
    //originEntry就是获取ArtMethod（c++）中的entry_point_from_quick_compiled_code_。
    //如果originEntry是跳板art_quick_interpreter_bridge说明未编译，手动触发JIT编译。
    long originEntry = artOrigin.getEntryPointFromQuickCompiledCode();
    if (originEntry == ArtMethod.getQuickToInterpreterBridge()) {
        Logger.i(TAG, "this method is not compiled, compile it now. current entry: 0x" + Long.toHexString(originEntry));
        //手动触发编译
        boolean ret = artOrigin.compile();
        if (ret) {
            originEntry = artOrigin.getEntryPointFromQuickCompiledCode();
            Logger.i(TAG, "compile method success, new entry: 0x" + Long.toHexString(originEntry));
        } else {
            Logger.e(TAG, "compile method failed...");
            return false;
            // return hookInterpreterBridge(artOrigin);
        }
    }
	//backup可以理解为clone，通过反射创建一个Method对象，并将信息完全拷贝，包括native中ArtMetjod对象指针
    ArtMethod backupMethod = artOrigin.backup();

    Logger.i(TAG, "backup method address:" + Debug.addrHex(backupMethod.getAddress()));
    Logger.i(TAG, "backup method entry :" + Debug.addrHex(backupMethod.getEntryPointFromQuickCompiledCode()));
	//如果已经hook过了，backupList不为null，可以复用。否则调用setBackMethod进行存储
    ArtMethod backupList = getBackMethod(artOrigin);
    if (backupList == null) {
        setBackMethod(artOrigin, backupMethod);
    }
	
    //这里通过锁来做多线程安全，通过，从原理中可知，是使用originEntry来标记同一函数。
    //这里通过originEntry关联的EntryLock对象做锁。
    final long key = originEntry;
    final EntryLock lock = EntryLock.obtain(originEntry);
    //noinspection SynchronizationOnLocalVariableOrMethodParameter
    synchronized (lock) {
        //如果之前没有hook过的，将originEntry对于的跳板实现Trampoline进行保存。
        //这里在原理部分有说道，如果函数逻辑一样，其originEntry也会一样。所以这里可能会有重复的。
        if (!scripts.containsKey(key)) {
            scripts.put(key, new Trampoline(ShellCode, originEntry));
        }
        //这个Trampoline类可以打开看一下，原理里说的核心实现就在这里。
        //
        Trampoline trampoline = scripts.get(key);
        //install，就是将原理里说的 跳板函数指令写入到originEntry只想的函数code中。
        boolean ret = trampoline.install(artOrigin);
        // Logger.d(TAG, "hook Method result:" + ret);
        return ret;
    }
}
```

通过java侧代码，对大致流程有了了解。然后我们来看一下细节，这中间有用到很多native函数，我们阅读下核心C++代码。

假设我们是在arm64-v8a，9.0手机上运行。9.0上ArtMethod（C++）结构在最前面已经给出，包括各自的offset和length。在epic中也有维护这部分，是在Offset类中，这里全面的给出了64位（arm64-v8a）和32位（thumb2），在Android4.4~11上ArtMethod结构各个属性的offset和length。但epic支持的是5.0以上，4.4的可以忽略用不到，

对于artOrigin.getEntryPointFromQuickCompiledCode()函数，其实就是去读取ArtMethod结构（C++）结构中的entry_point_from_quick_compiled_code_，是offset=32开始读取8位byte，然后转成long类型。这里注意是小端排序结构，这里看下读取的函数：

```java
public static long read(long base, Offset offset) {
    //base就是指向ArtMethod（C++）结构的指针，加上offset.offset就指向了我们要读取的块的开头。
    long address = base + offset.offset;
    //调用JNI函数读取内存，offset.length.width对于64位是8，32位是4。一个指针的大小。
    //这个JNI函数就不去展开了，就一个读取到ByteArray的过程。
    byte[] bytes = EpicNative.get(address, offset.length.width);
    if (offset.length == BitWidth.DWORD) {
        //对于32位机器，做 &0xFFFFFFFFL 操作，0xFFFFFFFFL就是4个字节长度全是1的二进制的16进制。
        return ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN).getInt() & 0xFFFFFFFFL;
    } else {
        //对于32位机器，直接用Long类型即可。ByteOrder.LITTLE_ENDIAN即小端字节顺序，低位在前的顺序。
        return ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN).getLong();
    }
}
```

对于ArtMethod.getQuickToInterpreterBridge()函数，是获取函数未被编译时entry_point_from_quick_compiled_code_的值。这里在epic原理部分的Android N 解释执行有说道，但没有完全针对这个函数说明白。

```java
public static long getQuickToInterpreterBridge() {
    //对于7.0以下即5.0、6.0。采用的AOT模式，即安装的时候就做了代码编译生成oat文件。
    //因此不会存在代码未被编译的情况。
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {
        return -1L;
    }
    //对于7.0即以上，采用的是AOT+JIT的模式，引入热点代码的概念。安装的时候不会编译代码。
    //在运行是采用JIT模式进行编译，对于热点代码会进行AOT模式编译生成art文件。
    //这里的原理是如果一个从来没有被用到的函数，在7.0以上是不会被编译的。
    //其entry_point_from_quick_compiled_code_的值就是QuickToInterpreterBridge（解释执行的入口）。
    final Method fake = XposedHelpers.findMethodExact(NeverCalled.class, "fake", int.class);
    return ArtMethod.of(fake).getEntryPointFromQuickCompiledCode();
}
```

然后就是artOrigin.compile()函数，手动触发编译，这个其原理在epic原理文档里已经说明了，这里看下C++的代码实现：

```c++
jboolean epic_compile(JNIEnv *env, jclass, jobject method, jlong self) {
    LOGV("self from native peer: %p, from register: %p", reinterpret_cast<void*>(self), __self());
    //取得method对于的art_method结构指针。
    jlong art_method = (jlong) env->FromReflectedMethod(method);
    if (art_method % 2 == 1) {
        //JniIdManager_DecodeMethodId_这是一个函数，初始话为null，
        //在api>30上是通过dlsym获取的_ZN3art3jni12JniIdManager14DecodeMethodIdEP10_jmethodID函数
        //源码注释： Android R would not directly return ArtMethod address but an internal id
        //这里还有很多动态调用系统库函数的兼容可以自己看一下。
        art_method = reinterpret_cast<jlong>(JniIdManager_DecodeMethodId_(ArtHelper::getJniIdManager(), art_method));
    }
    bool ret;
    //下面就是原理里说的调用系统库的函数来做代码编译。
    //现在Android 12马上出来了，如果我们需要做12的兼容，
    //需要通过查看源码和通过nm/objdump来查看系统库中的函数名来确定兼容逻辑
    if (api_level >= 30) {
      void* current_region = JitCodeCache_GetCurrentRegion(ArtHelper::getJitCodeCache());
      ret = ((JIT_COMPILE_METHOD3)jit_compile_method_)(jit_compiler_handle_, reinterpret_cast<void*>(self),
          reinterpret_cast<void*>(current_region),
          reinterpret_cast<void*>(art_method), false, false);
    } else if (api_level >= 29) {
        ret = ((JIT_COMPILE_METHOD2) jit_compile_method_)(jit_compiler_handle_,
                                                          reinterpret_cast<void *>(art_method),
                                                          reinterpret_cast<void *>(self), false, false);
    } else {
        ret = ((JIT_COMPILE_METHOD1) jit_compile_method_)(jit_compiler_handle_,
                                                          reinterpret_cast<void *>(art_method),
                                                          reinterpret_cast<void *>(self), false);
    }
    return (jboolean)ret;
}
```

最后我们来看一下函数：

```java
public boolean install(ArtMethod originMethod){
    //这里就是避免重复执行
    boolean modified = segments.add(originMethod);
    if (!modified) {
        // Already hooked, ignore
        Logger.d(TAG, originMethod + " is already hooked, return.");
        return true;
    }
	//这个page就是原理里说的二次跳板函数的代码的机器码字节数值。
    //安装epic原理里说的会根据原始函数的entry_point_from_quick_compiled_code_不同做分发。
    //可以理解为switch-case。调用具体的hook函数，
    byte[] page = create();
    //trampolineAddress就是我们二次跳板函数的地址。
    //我们通过mmap在内存中申请了一块内存用来存储page这段可执行代码然后返回其指针。
    EpicNative.put(page, getTrampolineAddress());

    int quickCompiledCodeSize = Epic.getQuickCompiledCodeSize(originMethod);
    int sizeOfDirectJump = shellCode.sizeOfDirectJump();
    //如果原函数过短，小于跳板函数，则直接将原函数的EntryPoint改为二次跳转函数的地址。
    //原理上说这种场景是无法hook的，包括下面说“绝对不能改EntryPoint”，有点疑问。
    if (quickCompiledCodeSize < sizeOfDirectJump) {
        Logger.w(TAG, originMethod.toGenericString() + " quickCompiledCodeSize: " + quickCompiledCodeSize);
        originMethod.setEntryPointFromQuickCompiledCode(getTrampolinePc());
        return true;
    }
    // 这一行是原始注释：这里是绝对不能改EntryPoint的，碰到GC就挂(GC暂停线程的时候，遍历所有线程堆栈，如果被hook的方法在堆栈上，那就GG)
    // source.setEntryPointFromQuickCompiledCode(script.getTrampolinePc());
    
    //调用了c++实现讲跳板函数的机器码写入到原函数entry_point_from_quick_compiled_code_指向的内存中。
    return activate();
}
```



### 函数替换

函数替换最早了解到的是AndFix热修复，是基于ART_METHOD结构的，但存在适配问题。后来阿里推出了Sophix热修复框架，当时有一篇PDF文档说明，其中说道通过ART_METHOD的整体替换，这样就不存在适配问题。基于这里说的方案，我们可以实现函数替换。

```java
public class ArtMethodMeasure {

    public void sizeCompute1(){
        //计算ArtMethod结构信息辅助方法1.
    }

    public void sizeCompute2(){
        //计算ArtMethod结构信息辅助方法2.
    }
}
```

在ArtMethodMeasure类中只定义两个函数，由于这两个函数的ART_METHOD结构在内存中是依次排列的，因此我们可以通过这两个指针相减得到ART_METHOD结构体的大小。

```c++
//相减得到ART_METHOD_SIZE
void JNICALL Java_com_profound_hook_MethodHook_initHook
        (JNIEnv *env, jclass clazz, jobject method1, jobject method2 ){
    jmethodID id1 = env->FromReflectedMethod(method1);
    jmethodID id2 = env->FromReflectedMethod(method2);
    ART_METHOD_SIZE = (size_t)((jlong)id2-(jlong)id1);
    LOGE("ART_METHOD_SIZE:%d",ART_METHOD_SIZE);
}
```

然后对于想要replace的函数，只要进行内存拷贝即可。

```c++
void JNICALL Java_com_profound_hook_MethodHook_replace
        (JNIEnv *env, jclass clazz, jobject origin, jobject hook){
    LOGE("ART_METHOD_SIZE:%d",ART_METHOD_SIZE);
    jmethodID origin_pointer = env->FromReflectedMethod(origin);
    jmethodID hook_pointer = env->FromReflectedMethod(hook);
    memcpy(origin_pointer,hook_pointer,ART_METHOD_SIZE);
}
```

按照Sophix热修复框架的介绍这样操作即可。对于热修复场景，我们的origin函数和hook函数是在同一个类中，这样确实没问题。但对于hook场景，origin函数和hook函数不是在同一个类，例如我替换BaseActivity中的setContentView函数为SnowflakeActivity中的实现，此时就会报错：

```tex
java.lang.VerifyError: Verifier rejected class com.zjwujlei.snowflake.MainActivity: void com.zjwujlei.snowflake.MainActivity.onCreate(android.os.Bundle) failed to verify: void com.zjwujlei.snowflake.MainActivity.onCreate(android.os.Bundle): [0x16] 'this' argument 'Reference: com.zjwujlei.snowflake.MainActivity' not instance of 'Reference: com.profound.snowflake.SnowflakeActivity' (declaration of 'com.zjwujlei.snowflake.MainActivity' appears in /data/app/com.zjwujlei.snowflake-MPgUX58_BbxhTkjT-esXgg==/base.apk!classes2.dex)
        at java.lang.Class.newInstance(Native Method)
        at android.app.AppComponentFactory.instantiateActivity(AppComponentFactory.java:69)
        at android.app.Instrumentation.newActivity(Instrumentation.java:1216)
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2865)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3082)
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:78)
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:108)
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:68)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1832)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loop(Looper.java:207)
        at android.app.ActivityThread.main(ActivityThread.java:6820)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:547)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:876)
```

这里我们查阅源码，是在method_verifier.cc中。如果没有校验过会调用 VerifyInvocationArgsFromIterator函数中对于非静态函数会对this和method中的class信息做校验，如果不匹配就会抛出这个问题，相关代码片段如下：

```c++
ArtMethod* MethodVerifier::VerifyInvocationArgsFromIterator(
    T* it, const Instruction* inst, MethodType method_type, bool is_range, ArtMethod* res_method) {
 	...
    /*
       * Check the "this" argument, which must be an instance of the class that declared the method.
       * For an interface class, we don't do the full interface merge (see JoinClass), so we can't do a
       * rigorous check here (which is okay since we have to do it at runtime).
       */
    if (method_type != METHOD_STATIC) {
		...
        if (!res_method_class->IsAssignableFrom(adjusted_type, this)) {
            Fail(adjusted_type.IsUnresolvedTypes()
                 ? VERIFY_ERROR_NO_CLASS
                 : VERIFY_ERROR_BAD_CLASS_SOFT)
                << "'this' argument '" << actual_arg_type << "' not instance of '"
                << *res_method_class << "'";
            // Continue on soft failures. We need to find possible hard failures to avoid problems in
            // the compiler.
            if (have_pending_hard_failure_) {
                return nullptr;
            }
        }
    }
    ...
}

```

可以看到对于非statc函数会进行校验。确实如果我们使用静态函数进行replace不会有这个问题。

通过VerifyInvocationArgsFromIterator函数我们一直往上寻找调用点，可以发现初始是从VerifyClass开始。这里分析一下重点的有意义的调用流程。VerifyClass(static)->VerifyMethods(static)->VerifyMethod(static)->Verify->CodeFlowVerifyMethod->···->VerifyInvocationArgsFromIterator。

```c++
FailureKind MethodVerifier::VerifyClass(Thread* self,
                                        mirror::Class* klass,
                                        CompilerCallbacks* callbacks,
                                        bool allow_soft_failures,
                                        HardFailLogMode log_level,
                                        std::string* error) {
  if (klass->IsVerified()) {
    return FailureKind::kNoFailure;
  }
    ...
}
```

对于VerifyClass函数，有一个IsVerified的判断。确实，如果我们已经进入过这个类做过Verify在做replace，就算是非静态函数也能成功替换，不会有java.lang.VerifyError错误。

```c++
MethodVerifier::FailureData MethodVerifier::VerifyMethods(Thread* self,
                                                          ClassLinker* linker,
                                                          const DexFile* dex_file,
                                                          const DexFile::ClassDef& class_def,
                                                          ClassDataItemIterator* it,
                                                          Handle<mirror::DexCache> dex_cache,
                                                          Handle<mirror::ClassLoader> class_loader,
                                                          CompilerCallbacks* callbacks,
                                                          bool allow_soft_failures,
                                                          HardFailLogMode log_level,
                                                          bool need_precise_constants,
                                                          std::string* error_string) {
  DCHECK(it != nullptr);

  MethodVerifier::FailureData failure_data;

  int64_t previous_method_idx = -1;
  while (HasNextMethod<kDirect>(it)) {
    ...
    ArtMethod* method = linker->ResolveMethod<ClassLinker::ResolveMode::kNoChecks>(
        method_idx, dex_cache, class_loader, /* referrer */ nullptr, type);
    ...
    MethodVerifier::FailureData result = VerifyMethod(self,
                                                      method_idx,
                                                      dex_file,
                                                      dex_cache,
                                                      class_loader,
                                                      class_def,
                                                      it->GetMethodCodeItem(),
                                                      method,
                                                      it->GetMethodAccessFlags(),
                                                      callbacks,
                                                      allow_soft_failures,
                                                      log_level,
                                                      need_precise_constants,
                                                      &hard_failure_msg);
    ...
    it->Next();
  }

  return failure_data;
}
```

这里看到，在类加载的时候会对所有函数进行进行VerifyMethod操作。

```c++
bool MethodVerifier::CodeFlowVerifyMethod() {
  const uint16_t* insns = code_item_accessor_.Insns();
  const uint32_t insns_size = code_item_accessor_.InsnsSizeInCodeUnits();
}
```

CodeFlowVerifyMethod这个函数里看到很熟悉的Insns，是每个函数代码块的信息，我们做字节码抽离加固的方案就是抽离函数的这个信息。

我们实际上用static函数做replace的时候，也会报错“java.lang.NoSuchMethodError: No static method xxx”。

```tex
java.lang.NoSuchMethodError: No static method create(Landroid/app/Activity;Landroidx/appcompat/app/AppCompatCallback;)Landroidx/appcompat/app/AppCompatDelegate; in class Landroidx/appcompat/app/AppCompatDelegate; or its super classes (declaration of 'androidx.appcompat.app.AppCompatDelegate' appears in /data/app/com.profound.snowflake-1jv9sL5Sl7sgNq7rs7TtMg==/base.apk)
        at androidx.appcompat.app.AppCompatActivity.getDelegate(AppCompatActivity.java:554)
        at androidx.appcompat.app.AppCompatActivity.attachBaseContext(AppCompatActivity.java:107)
        at android.app.Activity.attach(Activity.java:7137)
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2907)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3082)
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:78)
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:108)
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:68)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1832)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loop(Looper.java:207)
        at android.app.ActivityThread.main(ActivityThread.java:6820)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:547)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:876)
2021-08-19 22:49:19.533 26013-26013/com.profound.snowflake I/Process: Sending signal. PID: 26013 SIG: 9

```

这个在上面Epic中有说道，对于静态函数初始的时候entrypoint指向的都是一个默认位置，需要先手动调用下使其resolve。这里给出hook AppCompatDelegate.create函数（用来做setContentView函数拦截）时的完整代码：

```java
try {
    MethodHook.startup();
    AppCompatActivity param = null;
    try {
        AppCompatDelegate.create(param,param);
    }catch (Throwable thr){
        thr.printStackTrace();
    }
    try {
        SnowflakeHook.create(param,param);
    }catch (Throwable thr){
        thr.printStackTrace();
    }

    Method origin  = AppCompatDelegate.class.getDeclaredMethod("create",Activity.class, AppCompatCallback.class);
    Method hook  = SnowflakeHook.class.getDeclaredMethod("create",Activity.class, AppCompatCallback.class);
    origin.setAccessible(true);
    hook.setAccessible(true);
    MethodHook.replaceStaticMethod(origin,hook);
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}
```

最后，其实发现在9.0上我只要调用下AppCompatDelegate下的任意函数或者访问任意属性也能成功hook。