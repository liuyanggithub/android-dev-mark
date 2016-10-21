## 接上之前的分析
### [++Android Activity启动流程源码全解析（1）++](http://blog.csdn.net/ly502541243/article/details/52883198)
### 1.正在运行的Activity调用**startPausingLocked**
一个一个分析，先来看看**startPausingLocked**：

```
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        ......
        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                mService.updateUsageStats(prev, false);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }  
        ......
```
执行Activity所在进程的schedulePauseActivity方法,**prev.app.thread**为**IApplicationThread**，这又是一个**Binder**，真实的执行方法是在**ApplicationThread**中（继承自**IApplicationThread**的子类**ApplicationThreadNative**），这两个类是服务端跟客户端通信，对应客户端和服务端通信是**ActivityManagerProxy和ActivityManagerService**，他们都继承自**ActivityManagerNative**

**ApplicationThread**是**ActivityThread**的私有内部类：

**ApplicationThread.schedulePauseActivity**

```
        public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
        }
        
        private void sendMessage(int what, Object obj, int arg1, int arg2) {
            sendMessage(what, obj, arg1, arg2, false);
        }
        private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            if (async) {
                msg.setAsynchronous(true);
            }
            mH.sendMessage(msg);
        }
```
熟悉的handler发送消息，自然想到处理消息是在handleMessage中：

```
 case PAUSE_ACTIVITY:
     Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
     handlePauseActivity((IBinder)msg.obj, false, (msg.arg1&1) != 0, msg.arg2,
             (msg.arg1&2) != 0);
     maybeSnapshot();
     Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
     break;
 case PAUSE_ACTIVITY_FINISHING:
     Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
     handlePauseActivity((IBinder)msg.obj, true, (msg.arg1&1) != 0, msg.arg2,
             (msg.arg1&1) != 0);
     Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
```

这两个消息都是调用的

**handlePauseActivity**

```
    private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
            performPauseActivity(token, finished, r.isPreHoneycomb());

            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }

            //通知activity manager已经pause
            if (!dontReport) {
                try {
                    ActivityManagerNative.getDefault().activityPaused(token);
                } catch (RemoteException ex) {
                }
            }
            mSomeActivitiesChanged = true;
        }
    }
```
来到**performPauseActivity**：

```
    final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState) : null;
    }
    final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState) {
        if (r.paused) {
            if (r.activity.mFinished) {
                //如果已经finish，直接返回
                return null;
            }
            RuntimeException e = new RuntimeException(
                    "Performing pause of activity that is not resumed: "
                    + r.intent.getComponent().toShortString());
            Slog.e(TAG, e.getMessage(), e);
        }
        if (finished) {
            r.activity.mFinished = true;
        }
        try {
            //保存状态
            if (!r.activity.mFinished && saveState) {
                callCallActivityOnSaveInstanceState(r);
            }
            r.activity.mCalled = false;
            //交给Instrumentation处理
            mInstrumentation.callActivityOnPause(r.activity);
            EventLog.writeEvent(LOG_AM_ON_PAUSE_CALLED, UserHandle.myUserId(),
                    r.activity.getComponentName().getClassName());
            if (!r.activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onPause()");
            }

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                        "Unable to pause activity "
                        + r.intent.getComponent().toShortString()
                        + ": " + e.toString(), e);
            }
        }
        r.paused = true;

        // Notify any outstanding on paused listeners
        ArrayList<OnActivityPausedListener> listeners;
        synchronized (mOnPauseListeners) {
            listeners = mOnPauseListeners.remove(r.activity);
        }
        int size = (listeners != null ? listeners.size() : 0);
        for (int i = 0; i < size; i++) {
            listeners.get(i).onPaused(r.activity);
        }

        return !r.activity.mFinished && saveState ? r.state : null;
    }
```
来到**Instrumentation.callActivityOnPause**：

```
    public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
```
现在已经越来越近了，来到**Activity.performPause**
```
    final void performPause() {
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
        //终于来到了我们熟悉的生命周期方法
        onPause();
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        mResumed = false;
    }
```
到此为止，我们知道了要启动一个新的Activity之前，首先要执行现在正在栈顶运行的Activity的onPause()方法

### 2.如果要启动的Activity不为空且所在的进程存在的话，所在的进程执行**scheduleResumeActivity**启动Activity

再来看看刚才的第二种情况，如果现在这个Activity的所在线程已经存在的话那就执行**next.app.thread.scheduleResumeActivity**，这里的thread同样是**IApplicationThread**，也就是跟上面的情况一样，handler的处理那些就不赘述了，最终来到

**Instrumentation.callActivityOnResume**：
```
    public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
```
简单明了，调用了Activity的生命周期**onResume()**,在Activity的线程已经创建过，Activity重新回到可见状态时的必经方法

### 3.如果Activity为空，所在的进程不存在，执行**ActivityStackSupervisor.startSpecificActivityLocked**

**ActivityStackSupervisor.startSpecificActivityLocked**
```
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        //应用进程是否运行
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);

        if (app != null && app.thread != null) {//app不为空
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

        }
        //如果进程没有启动的话，启动进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```
这里app为空，需要启动进程，调用

**ActivityManagerService.startProcessLocked**
```
    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting,
            boolean isolated, boolean keepIfLarge) {
        return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
                hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
                null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
                null /* crashHandler */);
    }
```

**ActivityManagerService.startProcessLocked**:

```
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
            ......
            startProcessLocked(
                app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
            ......
}
```
继续调用重载方法

**ActivityManagerService.startProcessLocked**:
```
private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
            ......
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            checkTime(startTime, "startProcess: returned from zygote!");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            ......
            
}
```
调用**Process.start**方法，并且传入类名**android.app.ActivityThread**
```
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
```
继续调用**startViaZygote**
```
    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ......
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);                          
    }
}
```
**zygoteSendArgsAndGetResult**:

```
    private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                if (arg.indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx(
                            "embedded newlines not allowed");
                }
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();
            result.pid = inputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```
最后通过Zygote孵化出了我们刚才传入的ActivityThread，也就是我们很熟悉的应用入口进程,来到它的**main**方法
```
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        AndroidKeyStoreProvider.install();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
其他Looper相关的不多说，主要看看**thread.attach**:
```
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {//刚才传入的参数为false
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
        }
        ......
```
有涉及到了Binder，mgr.attachApplication(mAppThread)实际是调用的ActivityManagerService的同名方法：
**ActivityManagerService.attachApplication**
```
    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
**ActivityManagerService.attachApplicationLocked**
```
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        ......
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        ......
    }
```
**ActivityStackSupervisor.attachApplicationLocked**
```
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0);
        }
        return didSomething;
    }
```
两个循环Activity的显示和任务栈，最后调用
**ActivityStackSupervisor.realStartActivityLocked**

```
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
            .......
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);            
            .......
    }
```
主要是调用了线程的scheduleLaunchActivity方法，看到这个方法，知道跟上面的是一样的，都是BInder，来到ApplicationThread

**ApplicationThread.scheduleLaunchActivity**
```
        @Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
                ......
                sendMessage(H.LAUNCH_ACTIVITY, r);
}
```
看看**handleMessage**
```
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
```
交给handleLaunchActivity处理

**ActivityThread.handleLaunchActivity**
```
        Activity a = performLaunchActivity(r, customIntent);
```
**ActivityThread.performLaunchActivity**
```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //获取Activity信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        //Activity组件
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
        
        //创建Activity实例
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }  
        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            //创建上下文
            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                //初始化
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    //调用Activity的OnCreate生命周期方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    //调用Activity的OnStart生命周期方法
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {//恢复Activity
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```
这里面就看到了对Activity生命周期方法的调用了
**Instrumentation.callActivityOnCreate**
```
    public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
        prePerformCreate(activity);
        activity.performCreate(icicle, persistentState);
        postPerformCreate(activity);
    }
```
**Activity.performCreate**
```
    final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
```
第一个生命周期方法**onCreate**出现！继续回到ActivityThread.performLaunchActivity，在调用了Instrumentation.callActivityOnCreate之后又有activity.performStart()这一句：

**activity.performStart()**
```
    final void performStart() {
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
        mFragments.noteStateNotSaved();
        mCalled = false;
        mFragments.execPendingActions();
        mInstrumentation.callActivityOnStart(this);
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onStart()");
        }
        mFragments.dispatchStart();
        mFragments.reportLoaderStart();
        mActivityTransitionState.enterReady(this);
    }
```
**Instrumentation.callActivityOnStart**
```
    public void callActivityOnStart(Activity activity) {
        activity.onStart();
    }
```
又看到了第二个生命周期方法**onStart**，后面的生命周期方法就不深入看，有兴趣的可以自己跟踪一下源码

## 总结
经过了应用进程和系统进程一系列操作，完成Activity的启动，并且也找到了Activity的生命周期方法到底是怎么调用的，只是很多方法太长，其中的一些逻辑还不是很理解，后面有时间再仔细看看