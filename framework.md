

![VmiykV.png](https://s2.ax1x.com/2019/05/28/VmiykV.png)

## ART VS Dalivik

运行时库又分为核心库和ART(5.0系统之后，Dalvik虚拟机被ART取代)。核心库提供了Java语言核心库的大多数功能，这样开发者可以使用Java语言来编写Android应用。相较于JVM，Dalvik虚拟机是专门为移动设备定制的，允许在有限的内存中同时运行多个虚拟机的实例，并且每一个Dalvik 应用作为一个独立的Linux 进程执行。独立的进程可以防止在虚拟机崩溃的时候所有程序都被关闭。而替代Dalvik虚拟机的ART 的机制与Dalvik 不同。在Dalvik下，应用每次运行的时候，字节码都需要通过即时编译器转换为机器码，这会拖慢应用的运行效率，而在ART 环境中，应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用。

## Init

init进程是Android系统中用户空间的第一个进程，作为第一个进程，它被赋予了很多极其重要的工作职责，比如创建zygote(孵化器)和属性服务等。init进程是由多个源文件共同组成的，这些文件位于源码目录system/core/init。本文将基于Android7.0源码来分析Init进程。

### init入口函数

init的入口函数为main.**system/core/init/init.c**

调用了Init.rc文件，其中**init.zygote64.rc**调用了**Zygote服务**

```cpp
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```

其中service用于通知init进程创建名zygote的进程，这个zygote进程执行程序的路径为/system/bin/app_process64，后面的则是要传给app_process64的参数。class main指的是zygote的class name为main，后文会用到它。

```cpp
on nonencrypted    
    # A/B update verifier that marks a successful boot.  
    exec - root -- /system/bin/update_verifier nonencrypted  
    class_start main         
    class_start late_start 
```

init.rc中有这段 `class_start main` 而main就是zygote，所以就调用到了do_class_start（system/core/init/builtins.c）去解析这段,最后会调用到service_start,service_start会调用fork进行创建zygote

```cpp
static void service_start_if_not_disabled(struct service *svc)
{
    if (!(svc->flags & SVC_DISABLED)) {
        service_start(svc, NULL);
    } else {
        svc->flags |= SVC_DISABLED_START;
    }
}
```

```cpp
void service_start(struct service *svc, const char *dynamic_args)
{

    ...
    pid = fork();
    ...
    // 会执行/system/bin/app_process64
  execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
}
```

 执行完/system/bin/app_process64就会进入frameworks/base/cmds/app_process/app_main.cpp的main函数。

## Zygote

在Android系统中，DVM(Dalvik虚拟机)、应用程序进程以及运行系统的关键服务的SystemServer进程都是由Zygote进程来创建的，我们也将它称为孵化器。它通过fork  
(复制进程)的形式来创建应用程序进程和SystemServer进程，由于Zygote进程在启动时会创建DVM，因此通过fork而创建的应用程序进程和SystemServer进程可以在内部获取一个DVM的实例拷贝。

<img title="" src="file:///C:/Users/Lenovo/AppData/Roaming/marktext/images/2024-02-28-10-34-04-image.png" alt="" width="688">

从frameworks/base/cmds/app_process/app_main.cpp开始，会执行`runtime.start("com.android.internal.os.ZygoteInit", args, zygote);`，而runtime就是AndroidRuntime，调用它的start

```cpp
int main(int argc, char* const argv[])
{

    ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
```

**frameworks/base/core/jni/AndroidRuntime.cpp**

```cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{


    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    // 创建虚拟机DVM
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }

    onVmCreated(env);

    // 为DVM注册JNI
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    // className就是从app_main传过来的com.android.internal.os.ZygoteInit
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    // 把类名放入到数组
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    char* slashClassName = toSlashClassName(className);
    // 通过jni找到class对象
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        // 通过反射调用到main
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
   ...
}
```

### ZygoteInit

**frameworks/base/core/java/com/android/internal/os/ZygoteInit.java

![](.\ZygoteInte.main.png)

```java
public static void main(String argv[]) {
        try {
            // 初始化给zygote使用的socket
            registerZygoteSocket(socketName);

            // 加载资源
            preload();

            if (startSystemServer) {
                // fork SS进程 但是要规避SS进程等待AMS所以后续跳到catch
                startSystemServer(abiList, socketName);
            }

            // 等待AMS请求（Zygote进程）
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run(); // SS进程会跳入
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```

#### registerZygoteSocket

创建LocalServerSocket，也就是服务端的Socket。当Zygote进程将SystemServer进程启动后，就会在这个服务端的Socket上来等待**ActivityManagerService请求Zygote进程来创建新的应用程序进程**。

```java
private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                sServerSocket = new LocalServerSocket(
                        createFileDescriptor(fileDesc));
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```

#### startSystemServer

```java
private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        ...
        // args数组
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            // args数组封装成Arguments对象
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            // 执行fork
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

#### runSelectLoop

```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        FileDescriptor[] fdArray = new FileDescriptor[4];

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        int loopCount = GC_LOOP_COUNT;
        // 死循环等待ams发送请求
        while (true) {
            int index;


            if (loopCount <= 0) {
                gc();
                loopCount = GC_LOOP_COUNT;
            } else {
                loopCount--;
            }


            try {
                fdArray = fds.toArray(fdArray);
                index = selectReadable(fdArray);
            } catch (IOException ex) {
                throw new RuntimeException("Error in select()", ex);
            }

            if (index < 0) {
                throw new RuntimeException("Error in select()");
            } else if (index == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDescriptor());
            } else {
                boolean done;
                // 创建新应用程序
                done = peers.get(index).runOnce();

                if (done) {
                    peers.remove(index);
                    fds.remove(index);
                }
            }
        }
    }
```

### Zygote总结

Zygote启动流程就讲到这，Zygote进程共做了如下几件事：  
1.创建AppRuntime并调用其start方法，启动Zygote进程。  
2.创建DVM并为DVM注册JNI.  
3.通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层。  
4.通过registerZygoteSocket函数创建服务端Socket，并通过runSelectLoop函数等待ActivityManagerService的请求来创建新的应用程序进程。  
5.启动SystemServer进程。

## fork出的System Server工作流程

fork出system server后调用handleSystemServerProcess

### handleSystemServerProcess

```java
private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {
        // 关闭Fork Zygote出现的socket
        closeServerSocket();


        if (parsedArgs.invokeWith != null) {

        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
                Thread.currentThread().setContextClassLoader(cl);
            }
            // 初始化
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }
    }
```

**frameworks/base/core/java/com/android/internal/os/RuntimeInit.java**

```java
   public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        redirectLogStreams();

        commonInit();
        // 调用native代码调用JNI 本质是启动binder线程池
        nativeZygoteInit();
        // 通过反射获取SystemService类的main方法Method对象
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```

#### nativeZygoteInit

##### 启动Binder线程池

接着我们来查看nativeZygoteInit函数对应的JNI声明。  
**frameworks/base/core/jni/AndroidRuntime.cpp**

```cpp
static const JNINativeMethod gMethods[] = {  
 { "nativeFinishInit", "()V",  
 (void*) com_android_internal_os_RuntimeInit_nativeFinishInit },  
 { "nativeZygoteInit", "()V",  
 (void*) com_android_internal_os_RuntimeInit_nativeZygoteInit },  
 { "nativeSetExitWithoutCleanup", "(Z)V",  
 (void*) com_android_internal_os_RuntimeInit_nativeSetExitWithoutCleanup },  
};
```

**frameworks/base/cmds/app_process/app_main.cpp**

RuntimeInit.java的nativeZygoteInit函数主要做的就是启动Binder线程池，这样SyetemServer进程就可以使用Binder来与其他进程进行通信了

```cpp
virtual void onZygoteInit()
   {
       sp<ProcessState> proc = ProcessState::self();
       ALOGV("App process: starting thread pool.\n");
       proc->startThreadPool();// 启动线程池
   }
```

#### applicationInit

最重要的就是调用了invokeStaticMain

```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {

        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```

```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
             // 获取到com.android.server.SystemServer的类加载器
             cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            // 得到它的main方法
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        // 通过异常的方式把它抛出去
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

在ZygoteInit的main方法中接住异常

```java
    public static void main(String argv[]) {
     ...
          closeServerSocket();
      } catch (MethodAndArgsCaller caller) {
          caller.run(); // 接住异常调用run
      } catch (RuntimeException ex) {
          Log.e(TAG, "Zygote died with exception", ex);
          closeServerSocket();
          throw ex;
      }
  }
```

```java
        public void run() {
            try {
              // 这个mMethod是SystemServer的main方法
              mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
```

到SystemServer的main方法中

**frameworks/base/services/java/com/android/server/SystemServer.java**

```java
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```

```java
    private void run() {
       ...
           System.loadLibrary("android_servers");//1
       ...
           mSystemServiceManager = new SystemServiceManager(mSystemContext);//2
           LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
       ...    
        try {
           Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
           startBootstrapServices();//3
           startCoreServices();//4
           startOtherServices();//5
       } catch (Throwable ex) {
           Slog.e("System", "******************************************");
           Slog.e("System", "************ Failure starting system services", ex);
           throw ex;
       } finally {
           Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
       }
       ...
   }
```

run函数代码很多，关键就是在注释1处加载了libandroid_servers.so。接下来在注释2处创建SystemServiceManager，它会对系统的服务进行创建、启动和生命周期管理。启动系统的各种服务，在注释3中的startBootstrapServices函数中**用SystemServiceManager启动了ActivityManagerService、PowerManagerService、PackageManagerService等服务**。在注释4处的函数中则启动了BatteryService、UsageStatsService和WebViewUpdateService。注释5处的startOtherServices函数中则启动了CameraService、AlarmManagerService、VrManagerService等服务，这些服务的父类为SystemService。从注释3、4、5的函数可以看出，官方把系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，其中其他服务为一些非紧要和一些不需要立即启动的服务。系统服务大约有80多个

| 引导服务                        | 作用                                          |
| --------------------------- | ------------------------------------------- |
| Installer                   | 系统安装apk时的一个服务类，启动完成Installer服务之后才能启动其他的系统服务 |
| ActivityManagerService      | 负责四大组件的启动、切换、调度。                            |
| PowerManagerService         | 计算系统中和Power相关的计算，然后决策系统应该如何反应               |
| LightsService               | 管理和显示背光LED                                  |
| DisplayManagerService       | 用来管理所有显示设备                                  |
| UserManagerService          | 多用户模式管理                                     |
| SensorService               | 为系统提供各种感应器服务                                |
| PackageManagerService       | 用来对apk进行安装、解析、删除、卸载等等操作                     |
| **核心服务**                    |                                             |
| BatteryService              | 管理电池相关的服务                                   |
| UsageStatsService           | 收集用户使用每一个APP的频率、使用时常                        |
| WebViewUpdateService        | WebView更新服务                                 |
| **其他服务**                    |                                             |
| CameraService               | 摄像头相关服务                                     |
| AlarmManagerService         | 全局定时器管理服务                                   |
| InputManagerService         | 管理输入事件                                      |
| WindowManagerService        | 窗口管理服务                                      |
| VrManagerService            | VR模式管理服务                                    |
| BluetoothService            | 蓝牙管理服务                                      |
| NotificationManagerService  | 通知管理服务                                      |
| DeviceStorageMonitorService | 存储相关管理服务                                    |
| LocationManagerService      | 定位管理服务                                      |
| AudioService                | 音频相关管理服务                                    |
| …                           | ….                                          |

**frameworks/base/services/core/java/com/android/server/SystemServiceManager.java**

```java
  public <T extends SystemService> T startService(Class<T> serviceClass) {
  ...
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);//1
            } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
            }
...
            // Register it.
            mServices.add(service);//2
            // Start it.
            try {
                service.onStart();
            } catch (RuntimeException ex) {
                throw new RuntimeException("Failed to start service " + name
                        + ": onStart threw an exception", ex);
            }
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }     }
        return service;
    }
```

注释1处的代码用来创建SystemService，这里的SystemService是PowerManagerService，在注释2处将PowerManagerService添加到mServices中，这里mServices是一个存储SystemService类型的ArrayList。接着调用PowerManagerService的onStart函数启动PowerManagerService并返回，这样就完成了PowerManagerService启动的过程。

除了用mSystemServiceManager的startService函数来启动系统服务外，也可以通过如下形式来启动系统服务，以PackageManagerService为例

```java
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
               mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
```

直接调用了PackageManagerService的main函数：  
**frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**

```java
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    // Self-check for initial settings.
    PackageManagerServiceCompilerMapping.checkProperties();
    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);//1
    m.enableSystemUserPackages();
    // Disable any carrier apps. We do this very early in boot to prevent the apps from being
    // disabled after already being started.
    CarrierAppUtils.disableCarrierAppsUntilPrivileged(context.getOpPackageName(), m,
            UserHandle.USER_SYSTEM);
    ServiceManager.addService("package", m);//2
    return m;
}
```

注释1处直接创建PackageManagerService并在注释2处将PackageManagerService注册到**ServiceManager**(第一种方式不是注册到ServiceManager，是注册到SystemServerManager)中，ServiceManager用来管理系统中的各种Service，用于系统C/S架构中的Binder机制通信：Client端要使用某个Service，则需要先到ServiceManager查询Service的相关信息，然后根据Service的相关信息与Service所在的Server进程建立通讯通路，这样Client端就可以使用Service了。还有的服务是直接注册到**ServiceManager**中的，如下所示。



### 总结SyetemServer进程

SyetemServer在启动时做了如下工作：  
1.启动Binder线程池，这样就可以与其他进程进行通信。  
2.创建SystemServiceManager用于对系统的服务进行创建、启动和生命周期管理。  
3.启动各种系统服务。

## Binder

Binder是RPC框架，RPC是基于IPC

### IPC

在 Linux 中，每个进程都有自己的**虚拟内存地址空间**。虚拟内存地址空间又分为了用户地址空间和内核地址空间。

![](https://cdn.jsdelivr.net/gh/zzh0838/MyImages@main/img/20230703172935.png)

A进程访问B进程数据，需要B进程通过copy_from_user到B内核态，然后内核态数据是共享的，A进程在通过copy_to_user到A的用户态空间，使用。

为了优化两次cp，可以使用mmap将A进程的用户地址空间与内核态的空间进行映射，指向同一块相同的物理空间。

![](https://cdn.jsdelivr.net/gh/zzh0838/MyImages@main/img/20230703173006.png)

### RPC

Binder的RPC是通过设定相同的协议包格式进行传输

- 在 A 进程中按照固定的规则打包数据，这些数据包含了：
  - 数据发给哪个进程，Binder 中是一个整型变量 Handle
  - 要调用目标进程中的哪个函数，Binder 中用一个整型变量 Code 表示
  - 目标函数的参数
  - 要执行具体什么操作，也就是 Binder 协议
- 进程 B 收到数据，按照固定的格式解析出数据，调用函数，并使用相同的格式将函数的返回值传递给进程 A。

![](https://cdn.jsdelivr.net/gh/zzh0838/MyImages@main/img/20230703173016.png)

### Binder驱动对外提供使用接口

Binder 是一个字符驱动，对应的设备文件是 `/dev/binder`，和其他驱动一样，是通过 Linux 的文件访问系统调用对外提供具体功能的：

- open()，用于打开 Binder 驱动，返回 Binder 驱动的文件描述符

- mmap()，用于在内核中申请一块内存，并完成应用层与内核层的虚拟地址映射

- ioctl()，在应用层调用 ioctl ，从应用层向内核层发送数据或者读取内核层发送到应用层的数据：

- ```c
  ioctl(文件描述符,ioctl命令,数据)
  ```
  
  文件描述符是在调用 open 时的返回值，ioctl 命令和第三个参数"数据"的类型是相关联的，具体如下：

| ioctl命令                | 数据类型                      | 函数动作                  |
| ---------------------- | ------------------------- | --------------------- |
| BINDER_WRITE_READ      | struct binder_write_read  | 应用层向内核层收发数据           |
| BINDER_SET_MAX_THREADS | size_t                    | 设置最大线程数               |
| BINDER_SET_CONTEXT_MGR | int or flat_binder_object | 设置当前进程为ServiceManager |
| BINDER_THREAD_EXIT     | int                       | 删除 binder 线程          |
| BINDER_VERSION         | struct binder_version     | 获取 binder 协议版本        |

### Binder 应用层工作流程

- 通常，系统中的服务很多，我们需要一个管家来管理它们，**服务管家（ServiceManager）** 是 Android 系统启动时，启动的一个用于管理 **Binder 服务（Binder Service）** 的进程。通常，**服务（Service）** 需要事先注册到**服务管家（ServiceManager）**，其他进程向**服务管家（ServiceManager）** 查询服务后才能使用服务。

![](https://cdn.jsdelivr.net/gh/zzh0838/MyImages@main/img/20230703173026.png)

![](C:\Users\Lenovo\AppData\Roaming\marktext\images\2024-02-28-10-11-47-image.png)
