### android启动流程

    0、android架构

    1、android系统启动流程

    2、APP启动流程

    3、activity启动流程

    4、view树绘制流程

    5、16.6ms刷新机制

	  6、Android 采用 Binder 作为 IPC 机制（进程间的通信）
	https://www.zhihu.com/question/39440766/answer/89210950
	https://www.zhihu.com/question/44329366/answer/97794649
	https://www.zhihu.com/question/34652589/answer/90344494（主）同问题8

	 7、Android的Handler采用管道，Handler 机制（同一进程中不同线程间的通信）

	 8、Android中为什么主线程不会因为Looper.loop()方法造成阻塞？

##### android架构，分层及功能（从上往下）

[android架构图](/img/android.png)

    1、应用层

    2、Java框架层

    3.1、android运行时环境

    3.2、系统native库

    4、HAL（硬件抽象层）

    6、Linux内核层

[android系统启动流程](/img/android-boot.jpg)

[图片来源](http://gityuan.com/android/)

    各模块在启动过程的功能：

    Loader层：
      Boot ROM: 当手机处于关机状态时，长按Power键开机，引导芯片开始从固化在ROM里的预设代码开始执行，然后加载引导程序到RAM；
      Boot Loader：这是启动Android系统之前的引导程序，主要是检查RAM，初始化硬件参数等功能。

    Linux内核层：
      Android平台的基础是Linux内核，Linux内核的安全机制为Android提供相应的保障，也允许设备制造商为内核开发硬件驱动程序。
      启动内核初始化进程，用于初始化进程管理、内存管理，加载Display, Camera Driver，Binder Driver等相关工作（驱动）

    硬件抽象层 (HAL)：
      硬件抽象层 (HAL) 提供标准接口，HAL包含多个库模块，其中每个模块都为特定类型的硬件组件实现一组接口，比如WIFI/蓝牙模块，当框架API请求访问设备硬件时，Android系统将为该硬件加载相应的库模块

    Android Runtime & 系统库：
      每个应用都在其自己的进程中运行，都有自己的虚拟机实例。ART通过执行DEX文件可在设备运行多个虚拟机，DEX文件是一种专为Android设计的字节码格式文件，经过优化，使用内存很少。ART主要功能包括：预先(AOT)和即时(JIT)编译，优化的垃圾回收(GC)，以及调试相关的支持。

      这里的Native系统库主要包括init孵化来的用户空间的守护进程、HAL层以及开机动画等。启动init进程(pid=1),是Linux系统的用户进程，init进程是所有用户进程的鼻祖。

      init进程会孵化出ueventd、logd、healthd、installd、adbd、lmkd等用户守护进程；
      init进程还启动servicemanager(binder服务管家)、bootanim(开机动画)等重要服务
      init进程孵化出Zygote进程，Zygote进程是Android系统的第一个Java进程(即虚拟机进程)，Zygote是所有Java进程的父进程，Zygote进程本身是由init进程孵化而来的。

      Framework层：
        Zygote进程，是由init进程通过解析init.rc文件后fork生成的，Zygote进程主要包含：
        加载ZygoteInit类，注册Zygote Socket服务端套接字
        加载虚拟机
        提前加载类preloadClasses
        提前加载资源preloadResouces
        System Server进程，是由Zygote进程fork而来，System Server是Zygote孵化的第一个进程，System Server负责启动和管理整个Java framework，包含ActivityManager，WindowManager，PackageManager，PowerManager等服务。
        Media Server进程，是由init进程fork而来，负责启动和管理整个C++ framework，包含AudioFlinger，Camera Service等服务。

      App层：
        Zygote进程孵化出的第一个App进程是Launcher，这是用户看到的桌面App；
        Zygote进程还会创建Browser，Phone，Email等App进程，每个App至少运行在一个进程上。
        所有的App进程都是由Zygote进程fork生成的。

      Native与Kernel之间有一层系统调用(SysCall)层
      Java层与Native(C/C++)层之间的纽带JNI


##### android系统启动流程

    1、点击开机按键，引导芯片代码开始从预定义的地方（固化在ROM）开始执行。加载引导程序Bootloader到RAM，然后执行。
    2、引导程序Bootloader
    引导程序是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。
    3.linux内核启动
    内核启动时，设置缓存、被保护存储器、计划列表，加载驱动。当内核完成系统设置，它首先在系统文件中寻找”init”文件，然后启动root进程或者系统的第一个进程。
    4.init进程启动
    Init进程来启动zygote进程

    总结起来init进程主要做了三件事：
    1.创建一些文件夹并挂载设备
    2.初始化和启动属性服务
    3.解析init.rc配置文件并启动zygote进程

    Zygote启动流程就讲到这，Zygote进程共做了如下几件事：
    1.创建AppRuntime并调用其start方法，启动Zygote进程。
    2.创建DVM并为DVM注册JNI.
    3.通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层。
    4.通过registerZygoteSocket函数创建服务端Socket，并通过runSelectLoop函数等待ActivityManagerService的请求来创建新的应用程序进程。
    5.启动SystemServer进程。

    SyetemServer在启动时做了如下工作：
    1.启动Binder线程池，这样就可以与其他进程进行通信。
    2.创建SystemServiceManager用于对系统的服务进行创建、启动和生命周期管理。
    3.启动各种系统服务。

    ===========总结==========

    1.启动电源以及系统启动
    当电源按下时引导芯片代码开始从预定义的地方（固化在ROM）开始执行。加载引导程序Bootloader到RAM，然后执行。
    2.引导程序BootLoader
    引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。
    3.Linux内核启动
    内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置，它首先在系统文件中寻找init.rc文件，并启动init进程。
    4.init进程启动
    初始化和启动属性服务，并且启动Zygote进程。
    5.Zygote进程启动
    创建JavaVM并为JavaVM注册JNI，创建服务端Socket，启动SystemServer进程。
    6.SystemServer进程启动
    启动Binder线程池和SystemServiceManager，并且启动各种系统服务。
    7.Launcher启动
    被SystemServer进程启动的ActivityManagerService会启动Launcher，Launcher启动后会将已安装应用的快捷图标显示到界面上。


##### APP启动流程

    a、android系统启动后会ActivityManagerService会启动Launcher，Launcher是一个桌面程序，就是我们看到的桌面，将已安装的应用的快捷图标显示到界面上

    b、点击图标会启动对应的APP，实际也是调用的startActivity等相关方法

    c、相关流程
    1、启动APP时，会先通过Binder IPC从服务端（ActivityManagerService）获取一个Activity Manager然后通过ActivityManagernative封装成ActivityManagerProxy

    2、ActivityManagerProxy是作为APP进程与服务进程通信的客户端的角色

    3、然后ActivityManagerProxy会通知ActivityManagerService来启动activity

    4、ActivityManagerService会先找到要启动的activity（默认第一次启动的是清单文件里面注册的activity）

    5、然后Zygote进程会进行fork操作，来生成一个进程来运行APP，并且会创建进程的主入口，默认是ActivityThread，然后会调用ActivityThread的main方法

    6、在ActivityThread的main方法中初始化了主线程的looper，并创建了ActivityThread对象，最后会启动消息循环

[app与system通信](app-system.jpg)

    7、通过Binder IPC机制进行通信，系统进程的ActivityManagerService作为服务端接收APP进程的ActivityManagerProxy作为客户端的请求，APP进程中的ApplicationThread作为服务端来接收系统进程的ApplicationThreadProxy作为客户端发来的回复

    8、APP进程内部通过Handler机制进行消息传递，activity的生命周期方法的创建都是通过Handler来进行消息的分发和处理
