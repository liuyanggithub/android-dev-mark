## 前言
在上篇文章[从getApplicationContext和getApplication再次梳理Android的Application正确用法](http://blog.csdn.net/ly502541243/article/details/52105466)中，我提到
> 但是我们知道了mApplication和context是两个不同的东西，所以严格意义上来说getApplicationContext和getApplication是**不一样**的，虽然很多时候他们返回的都是同一个对象

注意到我这里说的是这两个方法返回的对象是不一样的，因为我看到Activity中这两个方法返回了两个对象，就单纯的以为他们真的是不一样的，看来真是浅尝辄止了，做了个错误示范，代码还是要刨根问底啊。
## 找不同
今天来做一个纠正和补充，我们来继续往下看代码，看看他们是不是真的不一样，还是有相似之处：


```
public abstract Context getApplicationContext();
```
getApplicationContext我们知道是一个抽象方法，他的真正实现是在**ContextImpl**中：

```
@Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
```
再来看看getApplication方法（只存在于Activity和Service中）：

```
public final Application getApplication() {
        return mApplication;
    }
```
那mApplication的赋值在哪？搜索一下，只有一个地方有赋值：

```
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);
        .......
        mApplication = application;
        }

```
在上篇文章中，我看到了这里就觉得这两个方法返回的对象不一样，可是我们忽略了**getApplicationContext**这个方法，当mPackageInfo不为空和为空是分别调用了**mPackageInfo.getApplication()和mMainThread.getApplication()**，那**getApplicationContext**到底返回的东西跟**mApplication**有什么不同，来看看这两个方法，在**LoadedApk.java**中看到**mPackageInfo.getApplication()**：

```
Application getApplication() {
        return mApplication;
    }
```
在**LoadedApk**也有一个mApplication，这个mApplication的赋值在LoadedApk的makeApplication：

```
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
...
if (mApplication != null) {
        return mApplication;
}
Application app = null;
...
app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
...
mActivityThread.mAllApplications.add(app);
        mApplication = app;
...
}

```
看到首先是一个空判断（单例），为空的话新建了一个Application然后赋值给mApplication，我们再看看mMainThread.getApplication()返回了什么，在**ActivityThread.java**中：

```
public Application getApplication() {
        return mInitialApplication;
    }
```
再来看看**mInitialApplication**的赋值在哪里：


```
private void handleBindApplication(AppBindData data) {
...
Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;
...

}
```
我们又看到了**makeApplication**，至于data.info也是**LoadedApk**这个类，看到这里我们就一目了然了，绕来绕去结果都是同一个东西，只是可能创建的时机不同，一个是在**LoadedApk**，一个是在**ActivityThread**，不过最后我们发现这个**getApplicationContext()**返回的都是**mApplication**。
## 真相大白
这个命名就很有意思了，在LoadedApk我们看到了一个叫mApplication的东西，在Activity也有一个叫mApplication，那他们是不是有什么联系呢？来看看在**Activity**中**mApplication**的赋值，在**attach**方法中找到了它（方法中的其他参数我去掉了）：

```
final void attach(Application application) {
mApplication = application;
}
```
也就是说等于调用attach方法时传入的application,那Activity的attach是在哪里调用呢，我们要来到反复提到的一个应用程序入口类ActivityThread，它有一个**performLaunchActivity**的方法，用来加载一个Activity，这里就有attach()的调用(我去掉了其他参数)：

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
...
Application app = r.packageInfo.makeApplication(false, mInstrumentation);
...
activity.attach(app);
...
}
```
我们发现又来了。。。熟悉的makeApplication()，r.packageInfo果然是**LoadedApk**类，最后殊途同归，又来到了这个单例，返回程序唯一的**mApplication**，还是一样的配方。。。
## 结果
结果很明显了，标题的问题已解，**getApplicationContext和getApplication**返回的是不是同一个对象？答：**是的！** 

> 当然话不能说的那么死，他们相同的前提是mApplication不为空，话又说回来，这个是全局的上下文，程序都启动了他怎么会为空呢，至于它到底什么情况会为空造成返回的对象不一样呢，待武功精进了继续分解。。。

