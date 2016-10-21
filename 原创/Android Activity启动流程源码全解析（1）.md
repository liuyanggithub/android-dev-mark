## 前言
Activity是Android四大组件的老大，我们对它的生命周期方法调用顺序都烂熟于心了，可是这些生命周期方法到底是怎么调用的呢？在启动它的时候会用到startActivty这个方法，但是这个方法的背后是怎样来实现的呢，来看看源码一探究竟(API23，无关代码省略)
## 应用进程启动activity流程
首先来到startActivity(Intent intent):
```
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
    }
```
内部都是调用的**startActivityForResult**，并且请求码为-1（即没有回调）
```
    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```
首次启动肯定mParent为空，执行Instrumentation(这个类用来做一些Activity的实际操作，**并且Instrumentation将在任何应用程序运行前初始化，可以通过它监测系统与应用程序之间的交互**)的**execStartActivity**方法：
```
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
参数：
- Context who：activity启动的上下文
- IBinder contextThread：上下文所在的主线程
- IBinder token：启动的activity的标识
- Activity target：启动的activity
- Intent intent：传入的intent
- int requestCode：请求码
- Bundle options：额外参数

ActivityMonitor用来监视应用启动activity的操作，最后调用**ActivityManagerNative.getDefault().startActivity**，先看看ActivityManagerNative.getDefault():
```
    static public IActivityManager getDefault() {
        return gDefault.get();
    }
    
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
一个单例用于返回**IActivityManager**对象，调用**asInterface**：
```
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
```
最后返回**ActivityManagerProxy**类，这个类是**ActivityManagerNative**的内部类

```
class ActivityManagerProxy implements IActivityManager
{
    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }

    public IBinder asBinder()
    {
        return mRemote;
    }

    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    ......
}
```
这里就涉及到了android的**Binder**机制，android的进程通信是使用**Binder**来进行**进程间**的通信（例外：**Zygote和SystemService是使用socket**，《[Android应用进程启动流程（Zygote进程与SystemServer进程）](http://blog.csdn.net/ly502541243/article/details/52639830)》），简单来说就是这里的**ActivityManagerProxy**作为客户端调用了**startActivity**方法，其实具体的实现是在服务端（也有一个对应的startActivity方法），这里对应的类为**ActivityManagerService**（同样继承自**ActivityManagerNative**）。
## 系统进程启动activity流程
首先来到**ActivityManagerService**的

**startActivity**
```
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }
```
**startActivityAsUser**：
```
    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```
设置了userId，并且再传入mStackSupervisor的startActivityMayWait，来到**ActivityStackSupervisor**类（所有activity的栈操作都通过这个类）

**startActivityMayWait**：
```
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
        ......
        int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);
        ......
}
```
在这个函数里主要进行了**Intent**的检查，Activity信息的获取还有重量级进程的判断，然后调用

**startActivityLocked**：

```
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
            ......
            ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, this, container, options);
            ......
            err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
            ......
}
```
**ActivityRecord**是server端Activity的对应类，包含了Activity的信息，每一个进程也对应一个**ProcessRecord**类来描述其信息，再继续调用：

**startActivityUncheckedLocked**：

```
......
final boolean launchSingleTop = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP;
final boolean launchSingleInstance = r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE;
final boolean launchSingleTask = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK;
......
ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
targetStack.mLastPausedActivity = null;
targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
......
```
首先判断了启动模式，然后再调用了startActivityLocked（不同于刚才的ActivityStackSupervisor.startActivityLocked，这里是ActivityStack类），上面的步骤完成了Activity的目标栈的判断(targetStack)

**ActivityStack.startActivityLocked**:

```
    final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
        ......
        task = r.task;
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());
        task.addActivityToTop(r);
        task.setFrontOfTask();

        r.putInHistory();
       if (!isHomeStack() || numActivities() > 0) {
            boolean showStartingIcon = newTask;
            ProcessRecord proc = r.app;
            if (proc == null) {
                proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
            }
            if (proc == null || proc.thread == null) {
                showStartingIcon = true;
            }
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else {
                mWindowManager.prepareAppTransition(newTask
                        ? r.mLaunchTaskBehind
                                ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                                : AppTransition.TRANSIT_TASK_OPEN
                        : AppTransition.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            mWindowManager.addAppToken(task.mActivities.indexOf(r),
                    r.appToken, r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                    r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
            boolean doShow = true;
            if (newTask) {
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && new ActivityOptions(options).getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {
               
                mWindowManager.setAppVisibility(r.appToken, true);
                ensureActivitiesVisibleLocked(null, 0);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                
                ActivityRecord prev = mResumedActivity;
                if (prev != null) {
                    // 满足判断则不需要复用之前已经启动的任务
                    // (1) 当前activity在不同的task
                    if (prev.task != r.task) {
                        prev = null;
                    }
                    // 当前activity不可见
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                mWindowManager.setAppStartingWindow(
                        r.appToken, r.packageName, r.theme,
                        mService.compatibilityInfoForPackageLocked(
                                r.info.applicationInfo), r.nonLocalizedLabel,
                        r.labelRes, r.icon, r.logo, r.windowFlags,
                        prev != null ? prev.appToken : null, showStartingIcon);
                r.mStartingWindowShown = true;
            }
        } else {
            // 如果是第一个activity，不需要动画
            mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                    r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                    r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
            ActivityOptions.abort(options);
            options = null;
        }
        if (VALIDATE_TOKENS) {
            validateAppTokensLocked();
        }

        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```
将Activity添加到了栈顶，初始化了WindowManager，并且又回到了StackSupervisor，调用

**StackSupervisor.resumeTopActivitiesLocked**：
```
    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // Already started above.
                    continue;
                }
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
```
判断目标任务栈是否在前面，还是调用**ActivityStack**类的

**ActivityStack.resumeTopActivityLocked**：

```
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            return false;
        }

        boolean result = false;
        try {
            mStackSupervisor.inResumeTopActivity = true;
            //锁屏相关
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```
确保当前栈顶的Activity是否resumed，然后调用

**ActivityStack.resumeTopActivityInnerLocked**（*前方代码过长预警*）：

```
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        if (!mService.mBooting && !mService.mBooted) {
            // AMS未初始化
            return false;
        }    
        ActivityRecord parent = mActivityContainer.mParentActivity;
        if ((parent != null && parent.state != ActivityState.RESUMED) ||
                !mActivityContainer.isAttachedLocked()) {
            //如果parent没有resume，直接返回
            return false;
        }
        cancelInitializingActivities();

        // 寻找第一个栈顶没有结束的Activity
        final ActivityRecord next = topRunningActivityLocked(null);
        
        //用户离开标志
        final boolean userLeaving = mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving = false;

        final TaskRecord prevTask = prev != null ? prev.task : null;
        
        //如果next为null，启动Launcher界面
        if (next == null) {
            // There are no more activities!
            final String reason = "noMoreActivities";
            if (!mFullscreen) {
                //如果当前栈没有遮盖整个屏幕，将焦点移到下一个可见的任务栈的正在运行的Activity
                final ActivityStack stack = getNextVisibleStackLocked();
                if (adjustFocusToNextVisibleStackLocked(stack, reason)) {
                    return mStackSupervisor.resumeTopActivitiesLocked(stack, prev, null);
                }
            }
            // Let's just start up the Launcher...
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: No more activities go home");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            // Only resume home if on home display
            final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                    HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
            return isOnHomeDisplay() &&
                    mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, reason);
        }
        next.delayedResume = false;

        
        /** mResumedActivity--系统当前激活的Activity组件
        *   mLastPausedActivity--上一次被中止的Activity组件
        *   mPausingActivity--正在被中止的Activity组件
        **/
        
        //如果栈顶的Activity已经是resumed，不做任何事
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed " + next);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
        //判断下个任务，上一个任务，决定是否调转到home
        final TaskRecord nextTask = next.task;
        if (prevTask != null && prevTask.stack == this &&
                prevTask.isOverHomeStack() && prev.finishing && prev.frontOfTask) {
            if (DEBUG_STACK)  mStackSupervisor.validateTopActivitiesLocked();
            if (prevTask == nextTask) {
                prevTask.setFrontOfTask();
            } else if (prevTask != topTask()) {
                final int taskNdx = mTaskHistory.indexOf(prevTask) + 1;
                mTaskHistory.get(taskNdx).setTaskToReturnTo(HOME_ACTIVITY_TYPE);
            } else if (!isOnHomeDisplay()) {
                return false;
            } else if (!isHomeStack()){
                if (DEBUG_STATES) Slog.d(TAG_STATES,
                        "resumeTopActivityLocked: Launching home next");
                final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                        HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
                return isOnHomeDisplay() &&
                        mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, "prevFinished");
            }
        }
        //如果系统要进入关机或者休眠状态，并且下一个Activity是上一次已经终止的Activity，直接返回
        if (mService.isSleepingOrShuttingDown()
                && mLastPausedActivity == next
                && mStackSupervisor.allPausedActivitiesComplete()) {
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Going to sleep and all paused");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        } 
        //如果不确定是哪个用户打开的Activity，直接返回
        if (mService.mStartedUsers.get(next.userId) == null) {
            Slog.w(TAG, "Skipping resume of top activity " + next
                    + ": user " + next.userId + " is stopped");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
        //如果当前启动的Activity正在准备stop，终止这个操作
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mStackSupervisor.mWaitingVisibleActivities.remove(next);
        ......
        //如果有正正在pause的Activity，直接返回
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    "resumeTopActivityLocked: Skip resume: some activity pausing.");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }
        ......
        boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
        //表示当前的Activity正在运行
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            //*********************让当前运行的Activity进入pause状态*********************
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
        if (pausing) {
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG_STATES,
                    "resumeTopActivityLocked: Skip resume: need to start pausing");
            //将即将启动的Activity放到LRU list，因为直接杀死的话会造成资源浪费
            if (next.app != null && next.app.thread != null) {
                mService.updateLruProcessLocked(next.app, true, null);
            }
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        } 
        // If the most recent activity was noHistory but was only stopped rather
        // than stopped+finished because the device went to sleep, we need to make
        // sure to finish it as we're making a new activity topmost.
        if (mService.isSleeping() && mLastNoHistoryActivity != null &&
                !mLastNoHistoryActivity.finishing) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "no-history finish of " + mLastNoHistoryActivity + " on new resume");
            requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                    null, "resume-no-history", false);
            mLastNoHistoryActivity = null;
        }
        //
        if (prev != null && prev != next) {
            if (!mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                    && next != null && !next.nowVisible) {
                mStackSupervisor.mWaitingVisibleActivities.add(prev);
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Resuming top, waiting visible to hide: " + prev);
            } else {
                //隐藏上一个Activity的界面，必须在finish状态，不然报错
                if (prev.finishing) {
                    mWindowManager.setAppVisibility(prev.appToken, false);
                    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                            "Not waiting for visible to hide: " + prev + ", waitingVisible="
                            + mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                            + ", nowVisible=" + next.nowVisible);
                } else {
                    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                            "Previous already visible but still waiting to hide: " + prev
                            + ", waitingVisible="
                            + mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                            + ", nowVisible=" + next.nowVisible);
                }
            }
        }
        //加载Activity，核实用户id是否正确
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    next.packageName, false, next.userId); 
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + next.packageName + ": " + e);
        }
        //打开动画，并且通知WindowManager关闭上一个Activity
        boolean anim = true;
        if (prev != null) {
            if (prev.finishing) {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        "Prepare close transition: prev=" + prev);
                if (mNoAnimActivities.contains(prev)) {
                    anim = false;
                    mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
                } else {
                    mWindowManager.prepareAppTransition(prev.task == next.task
                            ? AppTransition.TRANSIT_ACTIVITY_CLOSE
                            : AppTransition.TRANSIT_TASK_CLOSE, false);
                }
                mWindowManager.setAppWillBeHidden(prev.appToken);
                mWindowManager.setAppVisibility(prev.appToken, false);
            } else {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        "Prepare open transition: prev=" + prev);
                if (mNoAnimActivities.contains(next)) {
                    anim = false;
                    mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
                } else {
                    mWindowManager.prepareAppTransition(prev.task == next.task
                            ? AppTransition.TRANSIT_ACTIVITY_OPEN
                            : next.mLaunchTaskBehind
                                    ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                                    : AppTransition.TRANSIT_TASK_OPEN, false);
                }
            }
            if (false) {
                mWindowManager.setAppWillBeHidden(prev.appToken);
                mWindowManager.setAppVisibility(prev.appToken, false);
            }
        } else {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION, "Prepare open transition: no previous");
            if (mNoAnimActivities.contains(next)) {
                anim = false;
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_ACTIVITY_OPEN, false);
            }
        } 
        ......
        ActivityStack lastStack = mStackSupervisor.getLastStack();
        //如果Activity所在的进程已经存在
        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next);

            // 显示Activity
            mWindowManager.setAppVisibility(next.appToken, true);

            // 开始计时
            next.startLaunchTickingLocked();

            ActivityRecord lastResumedActivity =
                    lastStack == null ? null :lastStack.mResumedActivity;
            ActivityState lastState = next.state;

            mService.updateCpuStats();   
            
            //设置Activity状态
            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next + " (in existing)");
            next.state = ActivityState.RESUMED;
            mResumedActivity = next;
            next.task.touchActiveTime();
            mRecentTasks.addLocked(next.task);
            mService.updateLruProcessLocked(next.app, true, null);
            updateLRUListLocked(next);
            mService.updateOomAdjLocked();
            //根据Activity设置屏幕方向
            boolean notUpdated = true;
            if (mStackSupervisor.isFrontStack(this)) {
                Configuration config = mWindowManager.updateOrientationFromAppTokens(
                        mService.mConfiguration,
                        next.mayFreezeScreenLocked(next.app) ? next.appToken : null);
                if (config != null) {
                    next.frozenBeforeDestroy = true;
                }
                notUpdated = !mService.updateConfigurationLocked(config, next, false, false);
            }
            //确保Activity保持在栈顶
            if (notUpdated) {
                ActivityRecord nextNext = topRunningActivityLocked(null);
                if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                        "Activity config changed during resume: " + next
                        + ", new next: " + nextNext);
                if (nextNext != next) {
                    // Do over!
                    mStackSupervisor.scheduleResumeTopActivities();
                }
                if (mStackSupervisor.reportResumedActivityLocked(next)) {
                    mNoAnimActivities.clear();
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return false;
            }
            try {
                ......
                if (next.newIntents != null) {
                    next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
                }

                ......
                //是否睡眠
                next.sleeping = false;
                //兼容模式提示框
                mService.showAskCompatModeDialogLocked(next);
                //清理UI资源
                next.app.pendingUiClean = true;
                //设置处理状态
                next.app.forceProcessStateUpTo(mService.mTopProcessState);
                //清理最近的选择
                next.clearOptionsLocked();
                //*********************resume目标Activity*********************
                next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                        mService.isNextTransitionForward(), resumeAnimOptions);

                mStackSupervisor.checkReadyForSleepLocked();
                } catch (Exception e) {
                    ......
            }
            //设置可见
            try {
                next.visible = true;
                completeResumeLocked(next);
            } catch (Exception e) {
                //出现意外，放弃操作，尝试下一个Activity
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
            next.stopped = false;

        }else {//Activity所在的进程不存在的情况
            // 重启Activity
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.logo, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            //*********************启动Activity*********************
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
```
总的来说就是：

- 首先让现在正在运行的Activity调用**startPausingLocked**进入pause状态
- 如果要启动的Activity不为空且所在的进程存在的话，所在的进程执行**scheduleResumeActivity**启动Activity
- 如果Activity为空，所在的进程不存在，执行**ActivityStackSupervisor.startSpecificActivityLocked**

## 接下来继续分析

### [++Android Activity启动流程源码全解析（2）++](http://blog.csdn.net/ly502541243/article/details/52883212)