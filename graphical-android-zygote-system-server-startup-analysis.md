# Graphical Android - Zygote, System Server startup analysis

Init is the starting point for all Linux programs, and Zygote is on Android, just as its English meaning is the 'incubation pool' of all Java programs \(known to the brothers who have played with the Star Wars\). Can be seen with ps output[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
>adb shell ps | grep -E 'init|926'
 root      1     0     656    372   00000000 0805d546 S /init
 root      926   1     685724 43832 ffffffff b76801e0 S zygote
 system    1018  926   795924 62720 ffffffff b767fff6 S system_server
 u0_a6     1241  926   717704 39252 ffffffff b76819eb S com.android.systemui
 u0_a37    1325  926   698280 29024 ffffffff b76819eb S com.android.inputmethod.latin
 radio     1349  926   711284 30116 ffffffff b76819eb S com.android.phone
 u0_a7     1357  926   720792 41444 ffffffff b76819eb S com.android.launcher
 u0_a5     1523  926   703576 26416 ffffffff b76819eb S com.android.providers.calendar
 u0_a25    1672  926   693716 21328 ffffffff b76819eb S com.android.musicfx
 u0_a17    2040  926   716888 33992 ffffffff b76819eb S android.process.acore
 u0_a21    2436  926   716060 23904 ffffffff b76819eb S com.android.calendar
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

Init is the parent process of zygote, and system\_server and all other applications ending in com.xxx are from zygote fork. This article will illustrate the startup process of Zygote, system server and android application by means of a diagram \(with a small amount of code\).

Less nonsense, open two big pictures to open our Zygote tour. The first picture is the structure diagram of all the classes related to Zygote, and the other is the flow chart started by Zygote.

[![Zyogote class structure diagram](https://images0.cnblogs.com/blog/563439/201309/13121714-eed286f50b5041af9e18fd9275884476.png)](https://images0.cnblogs.com/blog/563439/201309/13121714-eed286f50b5041af9e18fd9275884476.png)

 According to the figure, we decompose the startup process of Zygote according to the serial number in Figure 1.

## 1. App\_Process 

* APP\_Process: An application that launches zygote and other Java programs. The code is located in frameworks/base/cmds/app\_process/app\_main.cpp, specified in init.rc.

```text
#init.rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
```

code show as below[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
...
else if (strcmp(arg, "--zygote") == 0) {
    zygote = true;
    niceName = "zygote";
} else if (strcmp(arg, "--start-system-server") == 0) {
    startSystemServer = true;
} else if (strcmp(arg, "--application") == 0) {
    application = true;
} 
...

if (zygote) {
   runtime.start("com.android.internal.os.ZygoteInit",
        startSystemServer ? "start-system-server" : "");
} else if (className) {
   // Remainder of args get passed to startup class main()
   runtime.mClassName = className;
   ...
   runtime.start("com.android.internal.os.RuntimeInit",
         application ? "application" : "tool");
} else {
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

 As you can see, there are three application types defined in app\_process:

       1.  Zygote:  com.android.internal.os.ZygoteInit

       2. System Server, not started separately, but started by Zygote

       3. Other Java programs that specify the class name, such as the commonly used am. /system/bin/am is actually a shell program, its real implementation is       

```text
exec app_process $base/bin com.android.commands.am.Am "$@"
```

These Java applications are all started with AppRuntime.start\(className\). As you can see from the first big picture, AppRuntime is a subclass of AndroidRuntime. It mainly implements several callback functions, and the start\(\) method is implemented in the AndroidRuntime method class. What is AnroidRuntime? We will start right away.

It should be noted that Zygote is not the first program launched by Init. It can be seen from the PID. Before it, the important System Daemon \(background process\) implemented by Native may be up first, such as ServiceManager \(service DNS service\).

## 2. AndroidRuntime

     First of all, what is Runtime? Take a look at several explanations given by the wiki:      

* [_Run time \(program lifecycle phase\)_](http://en.wikipedia.org/wiki/Run_time_%28program_lifecycle_phase%29)_, the period during which a computer program is executing_
* [_Runtime library_](http://en.wikipedia.org/wiki/Runtime_library)_, a program library designed to implement functions built into a programming language_

     I tend to refer to the latter here and see a further explanation:

     _In_ [_computer programming_](http://en.wikipedia.org/wiki/Computer_programming)_, a **runtime library** is the_ [_API_](http://en.wikipedia.org/wiki/API) _used by a_ [_compiler_](http://en.wikipedia.org/wiki/Compiler) _to invoke some of the behaviors of a_ [_runtime system_](http://en.wikipedia.org/wiki/Runtime_system)_. The runtime system implements the execution model and other fundamental behaviors of a_ [_programming language_](http://en.wikipedia.org/wiki/Programming_language)_. The compiler inserts calls to the runtime library into the executable binary. During execution \(_[_run time_](http://en.wikipedia.org/wiki/Run_time_%28program_lifecycle_phase%29)_\) of that_ [_computer program_](http://en.wikipedia.org/wiki/Computer_program)_, execution of those calls to the runtime library cause communication between the application and the_[_runtime system_](http://en.wikipedia.org/wiki/Runtime_system)_. This often includes functions for input and output, or for memory management._

     The generalization is that Runtime is the base library that supports the running of the program, it is bound to the language. such as:

* C Runtime: It is C standard lib, which is what we often say libc. \(Interestingly, the wiki will automatically redirect the "C runtime" to the "C Standard Library"\).
* Java Runtime: Again, the wiki redirects it to the "Java Virtual Machine", which of course includes the Java support library \(.jar\).
* AndroidRuntime: Obviously, it is the runtime environment required to run the Android app. This environment includes the following:
  * Dalvik VM: Android Java VM, which explains running Java programs in Dex format. Each process runs a virtual machine \(what is running a virtual machine? To put it bluntly, it is some C code, constantly interpreting the Dex format Bytecode, converting them to Machine code, and then executing, Of course, most Java virtual machines now support JIT, which means that bytecode may have been converted to machine code before running, which greatly improves performance. A common understanding in the past is that Java programs are static than C, C++, etc. The compiled language is slow, but with the intervention and development of JIT, this is completely a past tense. The dynamic operation of JIT allows the virtual machine to optimize the generation of machine code according to the runtime environment. In some cases, Java can even It runs faster than C/C++ and has platform-independent features, which is one of the reasons why Java is so popular today.\)
  * Android's Java class libraries, mostly from Apache Hamony, open source Java API implementations, such as java.lang, java.util, java.net. But removed AWT, Swing and other components.
  * JNI: Interface between C and Java intermodulation.
  * Libc: Android also has a lot of C code, naturally, libc, note that Android's libc is called bionic C.

       OK, then let's first take a look at how AndroidRuntime is built.     

[![](https://images0.cnblogs.com/blog/563439/201309/25164254-eadc0c7979b74628aefad305b4b4f75a.png)](https://images0.cnblogs.com/blog/563439/201309/25164254-eadc0c7979b74628aefad305b4b4f75a.png)

      The above figure shows the general flow of Zygote startup. The entry is AndroidRuntime.start\(\). There are two ways to start depending on the parameters passed in. One is "com.android.internal.os.RuntimeInit", the other is " com.android.internal.os.ZygoteInit", corresponding to the two classes RuntimeInit and ZygoteInit, which are represented by green and pink respectively. The main difference between the two classes is the Java side. It is obvious that ZygoteInit does a lot more than RuntimeInit, such as "preload", "gc" and so on. But on the Native side, they all do the same thing, startVM\(\) and startReg\(\), let's start here.

      As you can see from the class diagram, JavaVM and JNIEnv are the only two levels between the link between AndroidRuntim and Dalvik VM. It hides the implementation details in Dalvik. In fact, it is two function pointer structures that provide access to native code. The interface of the Java resource. JNIEnv is relative to the thread. The pointer to JNIEnv can finally correspond to the Thread structure inside Dalvik VM. All calls are completed in this structure context. The JavaVM corresponds to DVMGlobal, a process-only structure. It internally maintains a thread queue threadList, which stores each Thread structure object, as well as a list of objects of various states, and a structure that stores the GC, etc. . This article can't go deeper, just a brief introduction.

*   **JavaVM and JNIENV**
  * [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

    ```text
    Struct _JavaVM {
         const  struct JNIInvokeInterface* functions; // function pointer of C

    #if defined(__cplusplus)    ...
        jint GetEnv(void** env, jint version)
        { return functions->GetEnv(this, env, version); }
    #endif /*__cplusplus*/
    };

    struct JNIInvokeInterface {
        void*       reserved0;
        ...
        jint        (*DestroyJavaVM)(JavaVM*);
        jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);
        jint        (*DetachCurrentThread)(JavaVM*);
        jint        (*GetEnv)(JavaVM*, void**, jint);
        jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);
    };
    ```

    [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

    The most common interface inside is GetEnv\(\), which returns a JNIEnv object corresponding to each DVM thread. The definition of JNIEnv is very long. Interested students can find it in Jni.h. Here we only see how this object gets static jint GetEnv\(JavaVM\* vm, void\*\* env, jint version\) {
* [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  ```text
      Thread* self = dvmThreadSelf() ; //Get the current thread object. 
      If (version < JNI_VERSION_1_1 || version > JNI_VERSION_1_6) {
          Return JNI_EVERSION ;  
      } //Check the version number, Android 4.3 corresponds to 1.6     ... 
           *env = (void*) dvmGetThreadJNIEnv(self) ;   //It's very simple, see the bottom line 
          dvmChangeStatus(self, THREAD_NATIVE) ;
       return (*env ! = NULL) ? JNI_OK : JNI_EDETACHED ;
   }



  INLINE JNIEnv* dvmGetThreadJNIEnv(Thread* self) { return self->jniEnv; }
  ```

  [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  Very simple, the original is to read from the structure object of the current thread, it seems that there is no JavaVM, why is the parameter passed in? I don't know, maybe Google is reserved for future expansion? But no matter what, to get the call to GetEnv, you still need JavaVM. Students who want to write JNI code in the future can refer to the following code to see how to get JavaVM and JniENV.[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  ```text
  JNIEnv * AndroidRuntime :: getJNIEnv ()
  {
      JNIEnv * env;
      JavaVM* vm = AndroidRuntime::getJavaVM();
      assert(vm != NULL);

      if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK)
          return NULL;
      return env;
  }
  ```

  [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  At this point, we know that JavaVM and JNIEnv are native \(C/C++\) code for intermodulation with Java code, and that must be the Java virtual machine and the corresponding Java application on the Java side. What is the Java virtual machine in the end, how is it created? The answer starts with the AndroidRuntime::startVM\(\) function. startVM

* **startVM \(\)**
* [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  ```text
  int AndroidRuntime :: startVm (JavaVM ** pJavaVM, JNIEnv ** pEnv)
  {
      property_get("dalvik.vm.checkjni", propBuf, "");
      ...     
      initArgs.version = JNI_VERSION_1_4;
      ...
       //Create a VM and return JavaVM and JniEnv, pEnv corresponds to the current thread.
      If (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0 ) {
          ALOGE("JNI_CreateJavaVM failed\n");
          goto bail;
      }
      ...
  }

  Jint JNI_CreateJavaVM(JavaVM ** p_vm, JNIEnv** p_env, void * vm_args) {
       memset( &gDvm, 0 , sizeof (gDvm)); /* This is the real VM structure */ 
      JavaVMExt * pVM = (JavaVMExt*) Calloc ( 1 , sizeof (JavaVMExt));
      pVM -> funcTable = & gInvokeInterface; / / initialization function pointer
      pVM->envList = NULL; 
      ...
      gDvmJni.jniVm = (JavaVM* ) pVM; //The JavaVM that the native code contacts is just JniVm.
  ```

  ```text
      JNIEnvExt * pEnv = (JNIEnvExt* ) dvmCreateJNIEnv(NULL); //Create JNIEnv, because the next virtual machine initialization needs to access C/C++ implementation
  ```

  ```text
      /* 开始初始化. */
      gDvm.initializing = true;
      std::string status =
              dvmStartup(argc, argv.get(), args->ignoreUnrecognized, (JNIEnv*)pEnv);
      gDvm.initializing = false;

      dvmChangeStatus(NULL, THREAD_NATIVE);
      *p_env = (JNIEnv*) pEnv;
      *p_vm = (JavaVM*) pVM;
      return JNI_OK;
  ```

  [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  ```text
  std::string dvmStartup(int argc, const char* const argv[],
          bool ignoreUnrecognized, JNIEnv* pEnv)
  {
      /*
       * Check input and prepare initialization parameters
       */
      int cc = processOptions(argc, argv, ignoreUnrecognized);
      ...


      /* The actual initialization begins, initializes each internal module, and creates a series of threads */     
       if (! dvmAllocTrackerStartup()) {
           return  " dvmAllocTrackerStartup failed " ;
      }

      if (!dvmGcStartup()) {
          return "dvmGcStartup failed";
      }

      if (!dvmThreadStartup()) {
          return "dvmThreadStartup failed";
      }

      if (!dvmInlineNativeStartup()) {
          return "dvmInlineNativeStartup";
      }

      if (!dvmRegisterMapStartup()) {
          return "dvmRegisterMapStartup failed";
      }

      if (!dvmInstanceofStartup()) {
          return "dvmInstanceofStartup failed";
      }

      if (!dvmClassStartup()) {
          return "dvmClassStartup failed";
      }

      if (!dvmNativeStartup()) {
          return "dvmNativeStartup failed";
      }

      if (!dvmInternalNativeStartup()) {
          return "dvmInternalNativeStartup failed";
      }

      if (!dvmJniStartup()) {
          return "dvmJniStartup failed";
      }

      if (!dvmProfilingStartup()) {
          return "dvmProfilingStartup failed";
      }

      if (!dvmInitClass(gDvm.classJavaLangClass)) {
          return "couldn't initialized java.lang.Class";
      }

      if (!registerSystemNatives(pEnv)) {
          return "couldn't register system natives";
      }

      if (!dvmCreateStockExceptions()) {
          return "dvmCreateStockExceptions failed";
      }

      if (!dvmPrepMainThread()) {
          return "dvmPrepMainThread failed";
      }

      if (dvmReferenceTableEntries(&dvmThreadSelf()->internalLocalRefTable) != 0)
      {
          ALOGW("Warning: tracked references remain post-initialization");
          dvmDumpReferenceTable(&dvmThreadSelf()->internalLocalRefTable, "MAIN");
      }

      if (!dvmDebuggerStartup()) {
          return "dvmDebuggerStartup failed";
      }

      if (!dvmGcStartupClasses()) {
          return "dvmGcStartupClasses failed";
      }

      if (gDvm.zygote) {
          if (!initZygote()) {
              return "initZygote failed";
          }
      } else {
          if (!dvmInitAfterZygote()) {
              return "dvmInitAfterZygote failed";
          }
      }
      return "";
  }
  ```

  [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  There are too many details about the startup of the Java virtual machine that cannot be expanded here. Here we only need to know that it does the following things:

  1. Read a series of startup parameters from the property.   
  2. Create and initialize the structure global object \(per process\) gDVM, and corresponding to JavaVM and JNIEnv internal structure JavaVMExt, JNIEnvExt.   
  3. Initialize the java virtual machine and create a virtual machine thread. "ps -t", you can find that each Android application has the following threads[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  ```text
  U0_a46 1284 1281 714900 57896 20 0 0 0 fg ffffffff 00000000 S GC //Garbage Collection
   u0_a46    1285  1281  714900 57896 20    0     0     0     fg  ffffffff 00000000 S Signal Catcher   
   U0_a46     1286 1281 714900 57896 20 0 0 0 fg ffffffff 00000000 S JDWP //Java debugging
   u0_a46    1287  1281  714900 57896 20    0     0     0     fg  ffffffff 00000000 S Compiler           //JIT
   u0_a46    1288  1281  714900 57896 20    0     0     0     fg  ffffffff 00000000 S ReferenceQueueD          
   u0_a46    1289  1281  714900 57896 20    0     0     0     fg  ffffffff 00000000 S FinalizerDaemon    //Finalizer监护
   u0_a46    1290  1281  714900 57896 20    0     0     0     fg  ffffffff 00000000 S FinalizerWatchd    //
  ```

  [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  4. Register the JNI of the system through which the Java program accesses the underlying resources.

  ```text
      loadJniLibrary("javacore");
      loadJniLibrary("nativehelper");
  ```

  5. Make final preparations for the startup of Zygote, including setting the SID/UID, and mounting the file system.   
  6. Return the JavaVM to the Native code so that it can access the Java interface up.   
  
  In addition to the system's JNI interface \("javacore", "nativehelper"\), the android framework also has a large number of Native implementations, and Android will do all of these interfaces one-time through start\_reg\(\).

* **startReg \(\)**

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
int AndroidRuntime::startReg(JNIEnv* env){
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc); // Create threads that the JVM can access must pass through a specific interface. 

    Env -> PushLocalFrame( 200 );

    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0)    {
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);
    return 0;
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

            The Android native layer has two ways to create a Thread:[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
#threads.cpp
status_t Thread::run(const char* name, int32_t priority, size_t stack)
{
   ...   
   if (mCanCallJava) {
        res = createThreadEtc(_threadLoop, this, name, priority, stack, &mThread);
   } else {
        res = androidCreateRawThreadEtc(_threadLoop,this, name, priority, stack, &mThread);
    }
   ...
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

          The difference between them is whether they can call Java-side functions. The normal thread is a simple wrapper around pthread\_create.

          [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
int androidCreateRawThreadEtc(android_thread_func_t entryFunction,
                               void *userData,
                               const char* threadName,
                               int32_t threadPriority,
                               size_t threadStackSize,
                               android_thread_id_t *threadId)
{
    ...
    int result = pthread_create(&thread, &attr,android_pthread_entry)entryFunction, userData);
    ...
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

           The thread that can access the Java side needs to be bound with the JVM. The following is the concrete implementation function.          [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
#AndroidRuntime.cpp
int AndroidRuntime::javaCreateThreadEtc(
                                android_thread_func_t entryFunction,
                                void* userData,
                                const char* threadName,
                                int32_t threadPriority,
                                size_t threadStackSize,
                                android_thread_id_t* threadId)
{
     Args[ 0 ] = ( void *) entryFunction; // Precedent entryFunc in args[0] 
     args[ 1 ] = userData;
     args[2] = (void*) strdup(threadName);   
     result =AndroidCreateRawThreadEtc(AndroidRuntime::javaThreadShell, args, threadName, threadPriority, threadStackSize, threadId); //entryFunc变成javaThreadShell.
    return result;
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
int AndroidRuntime::javaThreadShell(void* args) {

    void* start = ((void**)args)[0];
    void* userData = ((void **)args)[1];
    char* name = (char*) ((void **)args)[2];        // we own this storage

    JNIEnv* env;
    /* 跟 VM 绑定 */
    if (javaAttachThread(name, &env) != JNI_OK)
        return -1;

    /* Run the real 'entryFunc' */ 
    result = (* (android_thread_func_t) start)(userData);


    /* unhook us */
    javaDetachThread ();
    ...
    return result;
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

      What does attachVM\(\) do? The space is limited and cannot be expanded. Here you only need to know the following points:

* * There is a Java virtual machine in a process. There are many threads inside the Java virtual machine, such as the GC listed above, FinalizeDaemon, and user-created threads.
  * Each Java thread maintains a JNIEnvExt object that holds a pointer to the DVM internal Thread object. That is, all calls from native to Java will reference this object.
  * All threads created by the JVM will be recorded inside the VM, but currently, we have not entered the Java world. The locally created thread VM is naturally unknown, so we need to notify the VM to create the corresponding internal data structure through attach.

      Take a look at the code below, you know, in fact, one of the important things that Attach\(\) does is to create thread and JNIEnvExt.

            [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
bool dvmAttachCurrentThread(const JavaVMAttachArgs* pArgs, bool isDaemon)

{
    Thread* self = NULL;
    ...
    self = allocThread(gDvm.stackSize);
    ...
    self->jniEnv = dvmCreateJNIEnv(self);
    ...
    gDvm.threadList->next = self;
    ...
    threadObj = dvmAllocObject(gDvm.classJavaLangThread, ALLOC_DEFAULT);
    vmThreadObj = dvmAllocObject(gDvm.classJavaLangVMThread, ALLOC_DEFAULT);
    ...
    self->threadObj = threadObj;
    ...
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

    When you are finished, you will start registering the local JNI interface function - register\_jni\_procs\(\). This function is actually a traversal call to a global array gRegJni\[\]. This array expansion can get the following results.

```text
static const RegJNIRec gRegJNI[] = {
   {register_android_debug_JNITest},
   {register_com_android_internal_os_RuntimeInit}.
    ...
}
```

    Each register\_xxx is a function pointer

```text
int jniRegisterNativeMethods(
      C_JNIEnv* env, 
      const char* className,
      const JNINativeMethod* gMethods, 
      int numMethods);
```

   What happens to RegisterNativeMethods inside the VM? Again, you only need to know the following points: 

   gRegJni \[\]

 Ok, after a lot of hard work, the Android runtime environment is ready, let's review what the AndroidRuntime initialization has done.

1. Created Dalvik VM.
2. Get Native access to Java's two interface objects, JavaVM and JNIENV.
3. Registered a batch \(see gRegJni\[\]\) native interface to the VM.

These operations are relatively time-consuming tasks. If each process does the same work, it will affect the startup speed. This is why we need to create Android applications through Zygote, because the Linux fork copy\_on\_write mechanism allows the child processes to Mapping these initialized memory spaces directly into their own process space does not require repetitive work, which increases the speed at which applications can be launched.

Can it be that the Android system only needs a basic runtime environment? The answer is obviously No. AndriodRuntime only provides the basic support of the language level, we need to quickly incubate and run the application on a multi-tasking, multi-user graphical operating system. This is Zygote, which is why in Figure 2, ZygoteInit does more than RuntimeInit. Then, let us really enter the world of Zygote.

## 3. ZygoteInit

      When the VM is ready, you can run the Java code. The system will enter the Java world for the first time. Remember the parameters of Runtime.start\(\) that are set in app\_main.cpp. That is the Java class we want to run. . Android supports two classes as a starting point, one is ' com.android.internal.os.ZygoteInit ', and the other is 'com.android.internal.os.RuntimeInit'.

     In addition, a ZygoteInit\(\) static method is defined in the Runtime\_Init class. It is created when Zygote creates a new application process, which does the same thing as the main\(\) function of the RuntimeInit class:

* redirectLogStreams\(\): Redirects System.out and System.err output to Android's Log system \(defined in android.util.Log\).
* commonInit\(\): Initializes the system properties. The most important point is to set up a handler for the uncaught exception. When the code has any unknown exceptions, it will execute it. The classmates who have debugged the Android code often see "\*\* \* FATAL EXCEPTION IN SYSTEM PROCESS" Printed here:[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

  ```text
  Runtime_init.java
  ...
  Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
  ...

  private static class UncaughtHandler implements Thread.UncaughtExceptionHandler {
          public void uncaughtException(Thread t, Throwable e) {
              try {
                  // Don't re-enter -- avoid infinite loops if crash-reporting crashes.
                  if (mCrashing) return;
                  mCrashing = true;
                  if (mApplicationObject == null) {
                      Slog.e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
                  } else {
                      Slog.e(TAG, "FATAL EXCEPTION: " + t.getName(), e);
                  }
                  ActivityManagerNative.getDefault().handleApplicationCrash(
                          mApplicationObject, new ApplicationErrorReport.CrashInfo(e));
              } catch (Throwable t2) {
                ...
              } finally {
                  Process.killProcess(Process.myPid());
                  System.exit(10);
              }
          }
      }
  ```

  [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

     Next, RuntimeInit::main\(\) and RuntimeInit::ZygoteInit\(\) call nativeFinishInit\(\) and nativeZygoteInit\(\) respectively, and then start to part ways. RuntimeInit's nativeFinishInit\(\) will eventually call the onStarted\(\) function in app\_main.cpp. , which calls the main\(\) function of the Java class, and then ends the process exit.

    [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
     virtual void onStarted()
     {
         sp<ProcessState> proc = ProcessState::self();
         proc->startThreadPool();
 
         AndroidRuntime * ar = AndroidRuntime :: getRuntime ();
         ar->callMain(mClassName, mClass, mArgC, mArgV);
 
         IPCThreadState::self()->stopProcess();
     }     
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

     RuntimeInit::ZygoteInit\(\) will be transferred to onZygoteInit\(\) of app\_main.cpp[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
virtual void onZygoteInit()
     {
         // Re-enable tracing now that we're no longer in Zygote.
         atrace_set_tracing_enabled(true);
 
         sp<ProcessState> proc = ProcessState::self();
         proc->startThreadPool();
     }
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

    It just starts a ThreadPool, and the rest of the work goes back to the Java side and is done by RuntimeInit::applicationInit\(\).

    So, we might as well understand RuntimeInit::main\(\), RuntimeInit::ZygoteInit\(\), ZygoteInit::main\(\), RuntimeInit's main\(\) method provides a standard Java program runtime, and RuntimeInit's ZygoteInit\(\) It is the method of closing the Android application. It is called when Zygote creates a new application process. This part of the code is implemented in the ZygoteInit class. In addition to the differences described above, the ZygoteInit class has done a few more things, let us analyze them one by one.

1.  registerZygoteSocket\(\);
2.  startSystemServer\(\);
3.  runSelectLoopMode\(\);

###      RegisterZygoteSocket\(\)

       In fact, the simple thing to do is to initialize the socket of the server \(that is, Zygote\). It is worth mentioning that the socket type used here is LocalSocket, which is a package of Android's Local Socket for Linux. Local Socket is a Socket-based interprocess communication method provided by Linux. For the server, the only difference is that bind to a local file descriptor \(fd\) instead of an IP address and port number. In many places in Android, Local Socket is used to communicate between processes. Search init.rc, you will see many such statements:[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
    socket adbd stream 660 system system
    socket vold stream 0660 root mount
    socket netd stream 0660 root system
    socket dnsproxyd stream 0660 root inet
    socket mdns stream 0660 root system
    socket rild stream 660 root radio
    socket rild-debug stream 660 radio system
    socket zygote stream 660 root system
    socket installd stream 600 system system
    socket racoon stream 600 system system
    socket mtpd stream 600 system system
    socket dumpstate stream 0660 shell log
    socket mdnsd stream 0660 mdnsr inet
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

       When init resolves to such a statement, it will do a few things:

            1. Call create\_socket\(\) \(system/core/init/util.c\), create a Socket fd, and put this fd with a file \(/dev/socket/xxx, xxx is the name listed above, for example, zygote\) Bind, set the relevant users, groups and permissions according to the definition in init.rc. Finally return this fd.   
            2. Register the socket name \(with the 'ANDROID\_SOCKET\_' prefix\) \(such as zygote\) and fd into the environment variable of the init process, so that all other processes \(all processes are init subprocesses\) can be obtained by getenv\(name\) To this fd.

       ZygoteInit completes the configuration of the Socket Server side with the following code:

       [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
private static final String ANDROID_SOCKET_ENV = "ANDROID_SOCKET_zygote";
private static void registerZygoteSocket() {

             String env = System.getenv(ANDROID_SOCKET_ENV);
              fileDesc = Integer.parseInt(env);
              ...
              sServerSocket = new LocalServerSocket(
                        createFileDescriptor(fileDesc));
              ...
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

    After the server is created, the corresponding client connection request can be made. As we mentioned earlier, AndroidRuntime a series of complex initialization work can help the child process to simplify the process through fork. Yes, Zygote creates a Socket server to respond to this fork request. Who is the request? Who is the child process of Zygote fork? The answer is ActivityManagerService and Android Application. What is the process like this? The answer is in the startup process of Andriod System Server. 

### Preload

    Preload\(\) does two things:  

```text
    static void preload() {
        preloadClasses();
        preloadResources();
    }
```

    This is the two most time-consuming things in the Android startup process. preloadClassess loads all classes defined by preloaded-classes in framework.jar into memory. Preloaded-classes can be found in framework/base after compiling Android. The preloadResources loads the system's Resource \(not the resource defined in the user apk\) into memory.

    The resource is preloaded into the Zygoted process address space. All fork children will share this space without reloading, which greatly reduces the application startup time, but in turn increases the system startup time. System startup can be accelerated by adjusting the number of preload classes and resources.

### **GC**  

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
  static  void gc () {
         final VMRuntime runtime = VMRuntime.getRuntime ();
        System.gc();
        runtime.runFinalizationSync();
        System.gc();
        runtime.runFinalizationSync();
        System.gc();
        runtime.runFinalizationSync();
    }
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

     Why did you adjust System.gc\(\) and runFinalizationSync\(\) three times? This is because the gc\(\) call simply tells the VM to do garbage collection, whether to recycle, and when to recycle it. GC recycling has a complex state machine control that allows as many resources as possible to be recycled through multiple calls. Gc\(\) must be completed before the fork \(the next StartSystemServer will have a fork operation\), so that the child process that will be copied in the future will have as little garbage memory as possible.

### Start SystemServer

      Think of the zygote parameter in init.rc, "--start-system-server", System Server is the first Java process of Zygote fork, this process is very important, because they have a lot of system threads, provide all cores System service, we can use 'ps -t \|grep &lt;system server pid&gt;' to see which threads are there, exclude the several Java virtual machine threads listed above, and[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
system    1176  1163  774376 51144 00000000 b76c4ab6 S SensorService
system    1177  1163  774376 51144 00000000 b76c49eb S er.ServerThread
system    1178  1163  774376 51144 00000000 b76c49eb S UI
system    1179  1163  774376 51144 00000000 b76c49eb S WindowManager
system    1180  1163  774376 51144 00000000 b76c49eb S ActivityManager
system    1182  1163  774376 51144 00000000 b76c4d69 S ProcessStats
system    1183  1163  774376 51144 00000000 b76c2bb6 S FileObserver
system    1184  1163  774376 51144 00000000 b76c49eb S PackageManager
system    1185  1163  774376 51144 00000000 b76c49eb S AccountManagerS
system    1187  1163  774376 51144 00000000 b76c49eb S PackageMonitor
system    1188  1163  774376 51144 00000000 b76c4ab6 S UEventObserver
system    1189  1163  774376 51144 00000000 b76c4d69 S BatteryUpdateTi
system    1190  1163  774376 51144 00000000 b76c49eb S PowerManagerSer
system    1191  1163  774376 51144 00000000 b76c2ff6 S AlarmManager
system    1192  1163  774376 51144 00000000 b76c4d69 S SoundPool
system    1193  1163  774376 51144 00000000 b76c4d69 S SoundPoolThread
system    1194  1163  774376 51144 00000000 b76c49eb S InputDispatcher
system    1195  1163  774376 51144 00000000 b76c49eb S InputReader
system    1196  1163  774376 51144 00000000 b76c49eb S BluetoothManage
system    1197  1163  774376 51144 00000000 b76c49eb S MountService
system    1198  1163  774376 51144 00000000 b76c4483 S VoldConnector
system    1199  1163  774376 51144 00000000 b76c49eb S CallbackHandler
system    1201  1163  774376 51144 00000000 b76c4483 S NetdConnector
system    1202  1163  774376 51144 00000000 b76c49eb S CallbackHandler
system    1203  1163  774376 51144 00000000 b76c49eb S NetworkStats
system    1204  1163  774376 51144 00000000 b76c49eb S NetworkPolicy
system    1205  1163  774376 51144 00000000 b76c49eb S WifiP2pService
system    1206  1163  774376 51144 00000000 b76c49eb S WifiStateMachin
system    1207  1163  774376 51144 00000000 b76c49eb S WifiService
system    1208  1163  774376 51144 00000000 b76c49eb S ConnectivitySer
system    1214  1163  774376 51144 00000000 b76c49eb S WifiManager
system    1215  1163  774376 51144 00000000 b76c49eb S Tethering
system    1216  1163  774376 51144 00000000 b76c49eb S CaptivePortalTr
system    1217  1163  774376 51144 00000000 b76c49eb S WifiWatchdogSta
system    1218  1163  774376 51144 00000000 b76c49eb S NsdService
system    1219  1163  774376 51144 00000000 b76c4483 S mDnsConnector
system    1220  1163  774376 51144 00000000 b76c49eb S CallbackHandler
system    1227  1163  774376 51144 00000000 b76c49eb S SyncHandlerThre
system    1228  1163  774376 51144 00000000 b76c49eb S AudioService
system    1229  1163  774376 51144 00000000 b76c49eb S backup
system    1233  1163  774376 51144 00000000 b76c49eb S AppWidgetServic
system    1240  1163  774376 51144 00000000 b76c4d69 S AsyncTask #1
system    1244  1163  774376 51144 00000000 b76c42a3 S Thread-64
system    1284  1163  774376 51144 00000000 b76c4d69 S AsyncTask #2
system    1316  1163  774376 51144 00000000 b76c2bb6 S UsbService host
system    1319  1163  774376 51144 00000000 b76c4d69 S watchdog
system    1330  1163  774376 51144 00000000 b76c49eb S LocationManager
system    1336  1163  774376 51144 00000000 b76c2ff6 S Binder_3
system    1348  1163  774376 51144 00000000 b76c49eb S CountryDetector
system    1354  1163  774376 51144 00000000 b76c49eb S NetworkTimeUpda
system    1360  1163  774376 51144 00000000 b76c2ff6 S Binder_4
system    1391  1163  774376 51144 00000000 b76c2ff6 S Binder_5
system    1395  1163  774376 51144 00000000 b76c2ff6 S Binder_6
system    1397  1163  774376 51144 00000000 b76c2ff6 S Binder_7
system    1516  1163  774376 51144 00000000 b76c4d69 S SoundPool
system    1517  1163  774376 51144 00000000 b76c4d69 S SoundPoolThread
system    1692  1163  774376 51144 00000000 b76c4d69 S AsyncTask #3
system    1694  1163  774376 51144 00000000 b76c4d69 S AsyncTask #4
system    1695  1163  774376 51144 00000000 b76c4d69 S AsyncTask #5
system    1791  1163  774376 51144 00000000 b76c4d69 S pool-1-thread-1
system    2758  1163  774376 51144 00000000 b76c4d69 S AudioTrack
system    2829  1163  774376 51144 00000000 b76c49eb S KeyguardWidgetP
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

See the famous WindowManager, ActivityManager? By the way, they are all running in the process of system\_server. There are also a number of "Binder-x" threads that are created by individual services in response to an application remotely invoking a request. In addition, there are many internal threads, such as "UI thread", "InputReader", "InputDispatch", etc. We will analyze these modules in subsequent articles. In this article, we only care about how System Server is created.

## 4. System Server startup process

There are a lot of code in this process, involving many classes. We use a sequence diagram to describe this process. The different colors in the diagram represent running in different threads.

[![](https://images0.cnblogs.com/blog/563439/201309/25154945-40c8b0ec63774e02af485973a4c05c45.png)](https://images0.cnblogs.com/blog/563439/201309/25154945-40c8b0ec63774e02af485973a4c05c45.png)

**1.** ZygoteInit fork A new process, this process is the SystemServer process.

**2.** The child process that forks out starts initialization in **handleSystemServerProcess** . The initialization is divided into two steps, one is done in native, and the other part \(mostly\) is done in Java. The native side works in AppRuntime \(subclass of AndroidRuntime\)::onZygoteInit\(\). One thing to do is to start a Thread, which is the main thread of SystemServer \(the leftmost pink square\), which is responsible for receiving and sending to other processes. The Binder calls the request. code show as below[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName(); //“Binder_1"
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
virtual bool threadLoop() {
```

     IPCThreadState::self\(\)-&gt;joinThreadPool\(mIsMain\); //blocking knows that the Binder driver wakes up  
     return false;   
  }[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

**3. After** nativeZygoteInit\(\) is completed, the Java layer is initialized. This process is long and complicated. We divide it into many steps. The initialization entry is SystemServer's main\(\) function, which calls Native's Init1\(\). Init1 is implemented in com\_android\_server\_SystemServer.cpp, and the final call to the function is system\_init\(\). The implementation of system\_init\(\) is as follows:[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
extern "C" status_t system_init()
{
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    sm->asBinder()->linkToDeath(grim, grim.get(), 0);
    property_get("system_init.startsurfaceflinger", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        // Start the SurfaceFlinger
        SurfaceFlinger::instantiate(); //初始化 SurfaceFlinger
        android_vt = 7;
    }

    property_get("system_init.startsensorservice", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
       // Start the sensor service
       SensorService::instantiate(); // 初始化SensorService.
    }

    ALOGI("System server: starting Android runtime.\n");
    AndroidRuntime* runtime = AndroidRuntime::getRuntime();
    JNIEnv* env = runtime->getJNIEnv();
    ...
    jclass clazz = env->FindClass("com/android/server/SystemServer");
    jmethodID methodId = env->GetStaticMethodID(clazz, "init2", "()V");
    ...
    env->CallStaticVoidMethod(clazz, methodId);
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
return NO_ERROR;
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

 Some points to note:

 A. SurfaceFlinger Service can be run in the System\_Server process, or it can be run in a separate process. If it is the latter, you need to add "setprop system\_init.startsurfaceflinger=1" to init.rc and ensure that service surfaceflinger is not "disable". ”

 B. init2 is implemented in System\_Server.java, which we will cover in detail later.

**4.** system\_init\(\) Finally, join\_threadpool\(\) suspends the current thread and waits for the request from the binder. The name of this thread is "Binder\_1". For the internal mechanism of service and binder, please refer to the article [http://www.cnblogs.com/samchen2009/p/3316001.html](http://www.cnblogs.com/samchen2009/p/3316001.html)

   
**5.** init2: At this point, the native initialization of the system server is completed, and it is back to the Java side. Here, many very important system services will be started. These jobs will start in a new thread with the thread name "android.server.ServerThread", see the green bar below. In ServerThread, SystemServer first creates two threads, UI thread and WindowManager thread. See the orange and peach bars in the figure. The handles of these two threads will be passed to some Service constructors, and some startup work will be done. Distribute to these two Threads.

    Each Thread finally enters the wait loop. Here, Android's Looper mechanism is used. Looper and Handler are the in-process messaging and processing mechanism of Android. We will be in the article  [http://www.cnblogs.com/samchen2009/p](http://www.cnblogs.com/samchen2009/p/3316004.html)  In detail, [/3316004.html](http://www.cnblogs.com/samchen2009/p/3316004.html) , here, we only need to know that Looper sleeps in a thread waiting for messages in the message queue, and then processes the message in a specific Handler. In other words, specify something to be processed in a thread.

**6.** Next, System Server will start a series of services, the most important of which is Acitivity Manager and Window Manager.

As you can see from the figure, Activity Manager has a Looper Thread, AThread. Please pay attention to the difference between Binder Thread and Looper, we will have a special article to introduce them later. A lot of the combination of Binder and Looper is used in Android. One of the important reasons is to solve the complex synchronization problem in multi-threading. Through a Looper and corresponding Message queue, you can serialize Binder calls of different processes in the future without Need to maintain complex and problem-prone locks.

Similar to WindowManager, his Handler Thread is one of the two Handler threads created in the initial stage of the System server we just mentioned, WMThread. His Binder thread will be specified by Kernel's Binder Driver.

In addition to the ActivityManager Service and WindowManager Service, there are many other services that are started one after another. They are not detailed here. You only need to know a few steps required for a Service to start:

       1. Initialize the Service object to get the IBinder object.

       2. Start the background thread and enter Loop to wait.

       3. Register yourself to the Service Manager and let other processes get the IBinder objects that are required to be called remotely by name.

**7.** There is no doubt that there are dependencies between so many services. For example, the ActivityManager Service cannot start an application until the WindowManager Service is initialized. How do you control these priorities? This is done by the System server's startup thread \(the green bar in the image below\) through the SystemReady\(\) interface. Each system service must implement a SystemReady\(\) interface that, when called, indicates that the system is OK, and the service can access \(directly or indirectly\) resources of other services. The last service to be tuned is the ActivyManager Service. AM's SystemReady\(\) is done in another thread via Runnable, and the arrow below the comment 8 in the figure below. The thing to do in this Runnable is to start an application that is currently at the top level - resumeTopActivityLocked\(\). Generally speaking, this is what we often call 'HOME'. Of course, you can specify other applications as Startup applications, such as GoogleTV, can use the TV application as a launcher, so that users can see the program directly after launching, similar to the set-top box at home. In addition, the ActivityManager Service also broadcasts the BOOT\_COMPLETED event to the entire system. In general, the background service of many applications can be monitored and started by registering the Receiver of this Event. The code to start Home is as follows[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
boolean startHomeActivityLocked(int userId) {
        ...
        Intent intent = new Intent(...);
        ... intent.addCategory(Intent.CATEGORY_HOME);
        ...        
        mMainStack.startActivityLocked(null, intent, null, aInfo,null, null, 0, 0, 0, null, 0, null, false, null);
        ...
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

**8.**  Android application startup is more complicated, we will study the details of ActivityManager in a special chapter. Here, we only need to know that ActivityStack is the stack that currently runs Activity, and resumeTopActivityLocked\(\) finds the one to start from. \(In the beginning, the stack was empty because we needed to push 'Home' to the stack via moveTaskFromFrontLocked\(\).\) If the application has never been launched, we need to create a process for it via AcitivyManagerService. note! The process is not created by the ActivityManager, don't forget, we mentioned Zygote is the incubator of all Android applications, right, the ActivityManager just informs Zygote to create. This communication is implemented in Process.java, the specific code is as follows:[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
static LocalSocket sZygoteSocket;
private static ProcessStartResult zygoteSendArgsAndGetResult(ArrayList<String> args)
            throws ZygoteStartFailedEx {
        openZygoteSocketIfNeeded();
        try {
            ...
            sZygoteWriter.write(arg);
          }
          ...
          sZygoteWriter.flush(); // wait after sending 
          ...
          result.pid = sZygoteInputStream.readInt();
          ...
          return result;
        } 
        sZygoteSocket = null ;
    }
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

At this point, the startup of System Server has been completed, and the startup of Zygote has been completed. Next we introduce the only thing in the life of the Zygote process and clone itself.

## _5. Fork_  

 Before Process.java sends a fork request, Zygote is ready for the server side, which we have covered in the previous Zygote Init chapter. Here we briefly analyze the processing of the request received by the Zygote Server. The code is in runSelectLoop\(\) of ZygoteInit.java,

 [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
    private static void runSelectLoop() throws MethodAndArgsCaller {

        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        FileDescriptor [] fdArray = new FileDescriptor [4 ];
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        While ( true ) {
             /* Doing the GC before the fork child process instead of doing it yourself in each child process can improve efficiency, but it can't be done every time because GC is still time consuming. */
            if (loopCount <= 0) {
                gc();
                loopCount = GC_LOOP_COUNT;
            } else {
                loopCount--;
            }

            try {
                fdArray = fds.toArray(fdArray);
                Index = selectReadable(fdArray); //select blocking wait
            } catch (IOException ex) {
               ...
            }

            /* Receive new connection */ 
            else  if (index == 0 ) {
                ZygoteConnection newPeer = acceptCommandPeer();
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                 boolean done;
                 /* complete the fork operation here */ 
                done = peers.get(index).runOnce();
                 if (done) {
                    peers.remove(index);
                    fds.remove(index);
                }
            }
        }
    }
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
        ...
        try {
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            ...
        }

        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;
 
        /* 
        if (parsedArgs.runtimeInit && parsedArgs.invokeWith != null) { 
```

             FileDescriptor\[\] pipeFds = Libcore.os.pipe\(\);   
             childPipeFd = pipeFds\[1\];   
             serverPipeFd = pipeFds\[0\];   
             ZygoteInit.setCloseOnExec\(serverPipeFd, true\);   
         }

```text
try {
            parsedArgs = new Arguments(args);
            applyUidSecurityPolicy(parsedArgs, peer, peerSecurityContext);
            ...

            pid = Zygote.forkAndSpecialize(parsedArgs.uid,parsedArgs.gid, parsedArgs.gids,parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,parsedArgs.niceName);
        } catch (IOException ex) {
            ...
        }

        Try {
             if (pid == 0 ) {
                 // child process, release 
                serverPedFd = null for serverFd ;
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
            } else {
                 //The parent process releases the 
                PipeFd of the unused child process IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

The launch of the Android app is done in handleChildProc:[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
    private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {

        closeSocket (); / / does not require the server socket 
        ZygoteInit.closeServerSocket ();
        ...
        If (parsedArgs.runtimeInit) { //All from Process.java is true 
            if (parsedArgs.invokeWith != null ) {
                WrapperInit.execApplication(parsedArgs.invokeWith,
                        parsedArgs.niceName, parsedArgs.targetSdkVersion,
                        pipeFd, parsedArgs.remainingArgs); // Start the command line program 
            } else {
                RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                        parsedArgs.remainingArgs); // Almost all applications start this way 
            }
        } else {
            ...
            if (parsedArgs.invokeWith != null) {
                WrapperInit.execStandalone(parsedArgs.invokeWith,
                        parsedArgs.classpath, className, mainArgs);
            } else {
                ...
                try {
                    ZygoteInit.invokeStaticMain(cloader, className, mainArgs);
                } catch (RuntimeException ex) {
                    ...
                }
            }
        }
    }
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

Here is the RuntimeInit.ZygoteInit\(\), which is the same as startSystemServer, and finally invokeStaticMain\(,""android.app.ActivityThread",\); invokeStatickMain\(\) function is implemented

```text
    static void invokeStaticMain(ClassLoader loader,String className, String[] argv)     throws zygoteInit.MethodAndArgsCaller {
        ...
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

Careful readers may ask a few questions:

1. Why not directly call the corresponding Java function, but through an exception? MethodAndArgsCaller. Here Android cleverly uses some of the design features of Java Exception. One of the great features of Execption is that when an exception occurs or throws, it can be traced back from the call stack from where the error occurred until the code segment that caught the exception is found. The code to catch the exception is as follows[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
    public static void main(String argv[]) {
        ...
        try {
            runSelectLoop ();
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            Caller.run(); //The real entry is here. 
        } catch (RuntimeException ex) {
            ...
        }
    }
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

Now you understand why you always see the following call stack in the dumpstate file.

```text
   ...
   at java.lang.reflect.Method.invokeNative(Native Method)
   at java.lang.reflect.Method.invoke(Method.java:511)
   at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:793)
   at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:560)
   at dalvik.system.NativeStart.main(Native Method)
```

2. Why do all applications start with "android.app.ActivityThread"? Leave a foreshadowing here, we will introduce the entire process of Android Activity from startup to display at http://www.cnblogs.com/samchen2009/p/3315993.html.   
  
Once the application is launched, Zygote has to go back to sleep and wait for a new application launch request.

## 6. After work

      Isn't it after this, Zygote's work has become very easy, can you raise a good year? Unfortunately, in modern society, which parent can raise the child and leave it alone? Especially for the eldest son who has a heavy social responsibility like the Sytem Server, parents still have to help. Here, Zygote will silently stare at his eldest son in the background. Once he discovers that System Server has been hanged, he will recycle it, then kill himself and start a new life again, which is a pity for the parents. This implementation is in the code: dalvik/vm/native/dalvik\_system\_zygote.cpp,

      [![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
static void Dalvik_dalvik_system_Zygote_forkSystemServer(
        const u4* args, JValue* pResult){
    ...
    pid_t pid;
    pid = forkAndSpecializeCommon(args, true); 
    ...
    if (pid > 0) {
        int status;
        gDvm.systemServerPid = pid; 
         /* WNOHANG will cause waitpid to return immediately, just to prevent the above system from crashing before the assignment statement is completed */ 
        if (waitpid(pid, &status, WNOHANG) == pid) {
            ALOGE("System server process %d has died. Restarting Zygote!", pid);
            kill(getpid(), SIGKILL);
        }
    }
    RETURN_INT(pid);
}


/* Real processing is here */ 
static  void sigchldHandler( int s)
{
    ...
    pid_t pid;
    int status;
    ...
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        ...
        if (pid == gDvm.systemServerPid) { 
            ...
            kill(getpid(), SIGKILL);
        }
    }
    ...
}


```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

```text
static void Dalvik_dalvik_system_Zygote_fork(const u4* args, JValue* pResult)
{
    pid_t pid;
    ...
    setSignalHandler(); //signalHandler Register here
    ...
    pid = fork();
    ...
    RETURN_INT(pid);
}
```

[![Copy code](http://common.cnblogs.com/images/copycode.gif)](javascript:void%280%29;)

    On Unix-like systems, the parent process must wait for the child process to exit with waitpid, otherwise the child process will become a "Zombie" process, not only the system resources will leak, but the system will crash \(no system server, all Android applications are Can not run\). But waitpid\(\) is a blocking function \(except for the WNOHANG parameter\), so the usual practice is to do non-blocking processing in the signal handler, because the system will issue a SIGCHID signal when each child exits. Zygote will kill himself, the father is dead, all the apps are not orphaned? No, because the system will automatically generate a SIGHUP signal to all child processes after the parent process is killed. The default processing of this signal is to kill itself and exit the current process. However, some background processes \(Daemon\) can ignore this signal by setting the SIG\_IGN parameter, so that it can continue to run in the background.

## to sum up

The startup process of Zygote and System Server is finally finished. Let's revisit this process with the complete class diagram above.

1. init runs app\_process according to init.rc and carries the '--zygote' and '--startSystemServer' parameters.

2. JavaVM will be started in AndroidRuntime.cpp::start\(\) and all system-related JNI interfaces will be registered.

3. For the first time into the Java world, run the ZygoteInit.java::main\(\) function to initialize Zygote. Zygote and create the server side of the Socket.

4. Then fork a new process and initialize SystemServer in the new process. Before Fork, Zygote is a commonly used Java class library for preload, as well as the system's resources, while GC\(\) cleans up the memory space, eliminating the need for duplicate work for the child process.

5. SystemServer initializes all system services, including ActivityManager and WindowManager, which are prerequisites for the application to run.

6. At the same time, Zygote listens to the server Socket and waits for a new application to start the request.

7. After the ActivityManager is ready, look for the system's "Startup" Application and send the request to Zygote.

8. After Zygote receives the request, the fork will issue a new process.

9. Zygote listens for and processes the SIGCHID signal from SystemServer and kills itself as soon as System Server crashes. Init will restart Zygote.

