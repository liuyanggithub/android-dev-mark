## 应用启动流程
Android系统是基于Linux的，所以它的所有应用也是基于Linux的Init进程创建出来的，首先Init进程启动Zygote（受精卵）进程，然后再fork出其他进程（包括SystemServer），最后开启各种应用进程。也就是流程如下：

> Init进程-->Zygote进程-->SystemServer进程-->应用进程

**SystemServer进程中启动系统的各种服务（ActivityManagerService、PackageManagerService、WindowManagerService...）**

## Zygote进程启动流程（API 23）
Init进程启动Zygote进程时会首先来到**ZygoteInit**类的main方法：
```
    public static void main(String argv[]) {
        try {
            RuntimeInit.enableDdms();
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    socketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            registerZygoteSocket(socketName);
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            preload();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());

            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            gcAndFinalize();

            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }

            Log.i(TAG, "Accepting command socket connections");
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```
- 第一句**enableDdms()** 打开DDMS（Dalvik Debug Monitor Service虚拟机调试服务）
- **SamplingProfilerIntegration.start()** 开始分析Zygote
- 一个循环解析main方法的参数列表，判断是否开启系统服务，获取ABI列表，socket名称 
- **registerZygoteSocket(socketName)** 注册socket（**android间的进程通信是使用Binder,唯独Zygote和SystemService是使用socket**）
- **preload()** 初始化

```
    static void preload() {
        Log.d(TAG, "begin preload");
        preloadClasses();
        preloadResources();
        preloadOpenGL();
        preloadSharedLibraries();
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        WebViewFactory.prepareWebViewInZygote();
        Log.d(TAG, "end preload");
    }
```
- preloadClasses()初始化相关的类
- preloadResources()初始化资源
- preloadOpenGL()初始化OpenGL
- preloadSharedLibraries()初始化系统的Lib
- preloadTextResources()初始化文字资源
- WebViewFactory.prepareWebViewInZygote()初始化webview

然后调用**startSystemServer**方法：
```
    private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_BLOCK_SUSPEND,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
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
在调用**Zygote.forkSystemServer**方法后创建了SystemServer进程
## SystemServer进程启动流程
在Zygote进程fork出了SystemServer进程后，来到SystemServer的main方法：
```
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```
就只有一句（还有注意下官方的注释），调用了run方法（删了一些无关代码）：

```
    private void run() {
        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }
        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
            final String languageTag = Locale.getDefault().toLanguageTag();

            SystemProperties.set("persist.sys.locale", languageTag);
            SystemProperties.set("persist.sys.language", "");
            SystemProperties.set("persist.sys.country", "");
            SystemProperties.set("persist.sys.localevar", "");
        }
        SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            mProfilerSnapshotTimer = new Timer();
            mProfilerSnapshotTimer.schedule(new TimerTask() {
                @Override
                public void run() {
                    SamplingProfilerIntegration.writeSnapshot("system_server", null);
                }
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        }

        VMRuntime.getRuntime().clearGrowthLimit();

        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

        Build.ensureFingerprintProperty();

        Environment.setUserRequired(true);


        BinderInternal.disableBackgroundScheduling(true);


        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
        android.os.Process.setCanSelfBackground(false);
        Looper.prepareMainLooper();


        System.loadLibrary("android_servers");


        performPendingShutdown();


        createSystemContext();


        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

        // Start services.
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        if (StrictMode.conditionallyEnableDebugLogging()) {
            Slog.i(TAG, "Enabled StrictMode for system server main thread.");
        }

        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
- 首先判断系统的当前时间，如果小于1970.就设置为1970
- 使用SystemProperties设置系统的语言、虚拟机库、内存。。。
- 初始化主Looper（**返回的Looper为单例**）
- 调用createSystemContext()创建上下文（activityThread.getSystemContext()返回的Context为单例），并且设置了系统主题

```
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }

```
关于Context相关的可以传送[《 从getApplicationContext和getApplication再次梳理Android的Application正确用法》](http://blog.csdn.net/ly502541243/article/details/52105466)

接下来这两句：
```
mSystemServiceManager = new SystemServiceManager(mSystemContext);
LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
```
创建了SystemServiceManager并且加到LocalServices(内部是个Map)来保存，来到最后：
```
startBootstrapServices();
startCoreServices();
startOtherServices();
```

```
private void startBootstrapServices() {
        // Wait for installd to finish starting up so that it has a chance to
        // create critical directories such as /data/user with the appropriate
        // permissions.  We need this to complete before we initialize other services.
        Installer installer = mSystemServiceManager.startService(Installer.class);

        // Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        // Power manager needs to be started early because other services need it.
        // Native daemons may be watching for it to be registered so it must be ready
        // to handle incoming binder calls immediately (including being able to verify
        // the permissions for those calls).
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // Now that the power manager has been started, let the activity manager
        // initialize power management features.
        mActivityManagerService.initPowerManagement();

        // Manages LEDs and display backlight so we need it to bring up the display.
        mSystemServiceManager.startService(LightsService.class);

        // Display manager is needed to provide display metrics before package manager
        // starts up.
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        // We need the default display before we can initialize the package manager.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // Only run "core" apps if we're encrypting the device.
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        // Start the package manager.
        Slog.i(TAG, "Package Manager");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();

        Slog.i(TAG, "User Service");
        ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

        // Initialize attribute cache used to cache resources from packages.
        AttributeCache.init(mSystemContext);

        // Set up the Application instance for the system process and get started.
        mActivityManagerService.setSystemProcess();

        // The sensor service needs access to package manager service, app ops
        // service, and permissions service, therefore we start it after them.
        startSensorService();
    }
```
首先调用mSystemServiceManager.startService(Installer.class)，这个服务是系统的安装应用服务，需要最先启动，来看看SystemServiceManager的startService方法：
```
    public <T extends SystemService> T startService(Class<T> serviceClass) {
        final String name = serviceClass.getName();
        Slog.i(TAG, "Starting " + name);

        // Create the service.
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service constructor threw an exception", ex);
        }

        // Register it.
        mServices.add(service);

        // Start it.
        try {
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + name
                    + ": onStart threw an exception", ex);
        }
        return service;
    }
```
通过反射创建实例，然后放到mServices数组中
```
private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();
```
最后调用service的onStart方法，刚才我们传入的是Installer，来到Installer的onStart方法：
```
private final InstallerConnection mInstaller;
...
public void onStart() {
        Slog.i(TAG, "Waiting for installd to be ready.");
        mInstaller.waitForConnection();
    }
```
来到InstallerConnection的waitForConnection方法:


```
    public void waitForConnection() {
        for (;;) {
            if (execute("ping") >= 0) {
                return;
            }
            Slog.w(TAG, "installd not ready");
            SystemClock.sleep(1000);
        }
    }
```
一个无限循环通过ping命令连接Zygote进程，连接成功后才开始启动服务

继续看其他服务的启动：

```
mActivityManagerService = mSystemServiceManager.startService(
        ActivityManagerService.Lifecycle.class).getService();
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
mActivityManagerService.setInstaller(installer);
```
接下里启动了**ActivityManagerService**，这个服务用来管理Android的四大组件(Activity,Service,Broadcast,ContentProvider)，来到**ActivityManagerService.Lifecycle**类：

```
public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }
```
它的onStart调用了ActivityManagerService的start方法：

```
    private void start() {
        Process.removeAllProcessGroups();
        mProcessCpuThread.start();

        mBatteryStatsService.publish(mContext);
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    }
```
这里面开启了操作CPU相关的子线程和电池状态相关的服务，应用操作相关的AppOpsService，最后添加到SystemService的存储数组中，具体的代码太复杂，就不赘述了

再继续看startBootstrapServices方法：
```
mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
```
启动了PowerManagerService，电源管理服务，这个的作用就不多说了吧，来看看PowerManagerService的onStart方法:
```
    public void onStart() {
        publishBinderService(Context.POWER_SERVICE, new BinderService());
        publishLocalService(PowerManagerInternal.class, new LocalService());

        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
    }
```
这几个方法就不详细看了，继续回到startBootstrapServices：

```
mActivityManagerService.initPowerManagement();
```
还记得刚才在ActivityManagerService启动了电池状态相关服务么？现在有了电源管理服务，在ActivityManagerService进行一下初始化：
```
    public void initPowerManagement() {
        mStackSupervisor.initPowerManagement();
        mBatteryStatsService.initPowerManagement();
        mLocalPowerManager = LocalServices.getService(PowerManagerInternal.class);
        PowerManager pm = (PowerManager)mContext.getSystemService(Context.POWER_SERVICE);
        mVoiceWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "*voice*");
        mVoiceWakeLock.setReferenceCounted(false);
    }
```
继续：
```
        mSystemServiceManager.startService(LightsService.class);
```
开启LightsService，闪关灯LED相关的服务，看看onStart方法：
```
    @Override
    public void onStart() {
        publishLocalService(LightsManager.class, mService);
    }
```
然后开启显示相关的服务DisplayManagerService：

```
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
```
也是来到DisplayManagerService的onStart方法，没什么特别之处：
```
    @Override
    public void onStart() {
        mHandler.sendEmptyMessage(MSG_REGISTER_DEFAULT_DISPLAY_ADAPTER);

        publishBinderService(Context.DISPLAY_SERVICE, new BinderService(),
                true /*allowIsolated*/);
        publishLocalService(DisplayManagerInternal.class, new LocalService());
    }
```
再回到startBootstrapServices，启动PackageManagerService：

```
mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);//设置默认的显示界面

mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();
```
这个的启动方法有所不同，直接调用了PackageManagerService.main：

```
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        ServiceManager.addService("package", m);
        return m;
    }
```
通过构造方法new出了PackageManagerService

最后在startBootstrapServices方法中启动UserManagerService和SensorService，用户管理服务和传感器服务：

```
ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());
......
startSensorService();//JNI方法
```
到了这里startBootstrapServices方法就执行完了，在SystemServer的run方法中接下来调用startCoreServices()：

```
    private void startCoreServices() {
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);

        // Tracks application usage stats.
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        // Update after UsageStatsService is available, needed before performBootDexOpt.
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

        // Tracks whether the updatable WebView is in a ready state and watches for update installs.
        mSystemServiceManager.startService(WebViewUpdateService.class);
    }
```
在这里启动了一些其他核心服务，流程跟刚才的启动流程一样，都是调用相关服务的onStart方法来启动，就不细看了，注意这里的BatteryService不同于刚才在ActivityManagerService的BatteryStatsService

最后调用**startOtherServices()** 启动一些其他服务：振动（VibratorService），网络管理（NetworkManagementService），网络状态（NetworkStatsService），窗口管理（WindowManagerService）。。。。。。
这个方法代码太长，就不贴过来了
## 总结
1. Zygote进程是所有进程的父进程，它fork出了SystemServer进程
2. SystemServer进程的main方法中调用run方法进行初始化
3. 在run方法中创建了SystemServiceManager，并且依次调用**startBootstrapServices();startCoreServices();startOtherServices();** 开启系统服务（三类服务：BootstrapServices-->CoreServices-->OtherServices）
4. 在启动服务时调用服务类的**onStart**方法来初始化

---

附上系统服务启动顺序：
#### Installer-->ActivityManagerService-->PowerManagerService-->
#### ActivityManagerService-->DisplayManagerService-->PackageManagerService-->
#### UserManagerService-->SensorService-->BatteryService-->
#### UsageStatsService-->WebViewUpdateService-->OtherServices
