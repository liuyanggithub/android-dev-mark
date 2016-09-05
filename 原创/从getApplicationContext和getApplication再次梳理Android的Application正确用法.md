## Context
在Android开发的时候，很多地方我们都会用上Context这个东西，比如我们最常用的startActivity,以前也没怎么在意这个到底有什么用，方法要参数就直接传过去，今天看到**getApplicationContext**和**getApplication**有点懵逼，我觉得有必要去一探究竟了，首先看看什么是Context：

Context，翻译为上下文，环境。多么直白又艹的翻译，想问啥又是上下文，啥又是环境，程序还有上下文。。。为了不误人子弟，来Google的官方说法：
> Interface to global information about an application environment. This is an abstract class whose implementation
  is provided by the Android system. It allows access to application-specific resources and classes, as well as up-calls 
  for application-level operations such as launching activities, broadcasting and receiving intents, etc
  
翻译下：它是一个应用程序的全局环境，是Android系统的一个抽象类，可以通过它获取程序的资源，比如：加载Activity，广播，接收Intent信息等等。

总的来说它就像是一个程序运行的时候的环境，如果Activity，Service这些是水里的鱼，那它就是水？（原谅我的理解能力，不知道怎么形容），好吧，理解不透就看代码（以下代码来自**API-23**）：

```
public abstract class Context {}
```
首先它是个抽象类，那它提供了哪些方法，哎，太多了，随便看几个吧：

```
//[这个可以看看我的博客另外一篇专门讲消息机制的](http://blog.csdn.net/ly502541243/article/details/52062179/)
public abstract Looper getMainLooper();
//获取当前应用上下文
public abstract Context getApplicationContext();
//开启activity
public abstract void startActivity(Intent intent);
//获取valus/strings.xml声明的字符串
public final String getString(@StringRes int resId) {
        return getResources().getString(resId);
    }
    
//获取valus/colors.xml声明的颜色    
public final int getColor(int id) {
    return getResources().getColor(id, getTheme());
}
//发送广播
public abstract void sendBroadcast(Intent intent);
//开启服务
public abstract ComponentName startService(Intent service);
//获取系统服务（ALARM_SERVICE，WINDOW_SERVICE，AUDIO_SERVICE、、、）
public abstract Object getSystemService(@ServiceName @NonNull String name);
```
我们发现Context这个抽象类里面声明了很多我们开发中常用一些方法，那有哪些类实现了这个抽象类呢（普及一个快捷键，eclipse下点击类后按F4可以看这个类的继承结构，AndroidStudio我设置的是eclipse的快捷键），结构如下

> - Context 
>   - **ContextWrapper**
>     - TintContextWrapper
>     - ContextThemeWrapper
>     - IsolatedContext
>     - MutableContextWrapper
>     - ContextThemeWrapper
>       - **Activity** 
>     - **Service**
>     - RenamingDelegatingContext
>     - **Application**
>     - BackupAgent

我们主要关注一下：ContextWrapper，Activity，Service，Application,先来看看Context的主要实现类**ContextWrapper**(*剧透：其实这并不是真正的实现类*)：看下官方注释，意思就是这是个简单的实现：
> Proxying implementation of Context that simply delegates all of its calls to another Context.

构造方法：

```
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    //设置BaseContext，同构造方法，多了个不为空的判断
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    
    .....
    .....
    @Override
    public ContentResolver getContentResolver() {
        return mBase.getContentResolver();
    }
    @Override
    public Looper getMainLooper() {
        return mBase.getMainLooper();
    }
    @Override
    public Context getApplicationContext() {
        return mBase.getApplicationContext();
    }
}
```
注意方法是**public**的，所以继承类可以直接访问，看了方法的实现，我们发现真是simply，就是都交给**mBase**来做相应的处理，关键就是构造方法或者**attachBaseContext**方法设置mBase并且进行操作。

来看看我们最常用的Activity，主要看看**getApplication**：

```
public class Activity extends ContextThemeWrapper implements ... {
       private Application mApplication; 
       final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
            attachBaseContext(context); 
            ...
            mApplication = application;
        }
}
public final Application getApplication() {
        return mApplication;
}
```
我们看到了在**attach**调用了我们刚才说的**attachBaseContext**，还有给**mApplication**赋值。这里出现了另外一个我们关注的**Application**，到源码看看：

```
//构造方法传了个空，貌似没什么用
public Application() {
        super(null);
    }
//同样在attach中我们看到了具体的东西
final void attach(Context context) {
        attachBaseContext(context);
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
```
Application和Activity的attach方法感觉都差不多，都调用了**attachBaseContext(context)**，成为了一个Context。

这里还看到了**ContextImpl**，其实它才是**Context**的真正实现类（看名字也看出来了），可是刚才我们看Context的继承结构时没看到这个类啊，原来它跟**ActivityThread**一样，并没有在sdk中，所以是看不到的。这个类的具体实现就不仔细看了，再看要晕了，后面有时间再细细品味...

但是我们知道了**mApplication**和**context**是两个不同的东西，所以严格意义上来说**getApplicationContext**和**getApplication**是不一样的，虽然很多时候他们返回的都是同一个对象，但是getApplication只存在于Activity或者Service中，我们要注意具体的情况，这个我们后面再说
## Application
### 为何物
看到这里我们发现，Application和Activity都继承自Context,他们都是**环境**，只不过Application是随着我们的应用（或者包）启动的时候就存在的环境，Activity是一个界面的环境
### 使用方法
既然Application是在应用一创建就初始化了，而且是在应用运行时一直存在的，那我们可以把它当做是一个全局变量来使用，可以保存一些共享的数据，或者说做一些工具类的初始化工作。要自己来使用Application的话我们需要先新建一个类来继承Application
```
public class MyApplication extends Application {}
```
然后重写它的**onCreate**做一些工具的初始化：

```
@Override
    public void onCreate() {
        super.onCreate();
        ToastUtils.register(this);
        //LeakCanary检测OOM
        LeakCanary.install(this);
    }
```
最后一个关键的工作是要在manifest里面做一下声明（无关代码我忽略了）

```
<application
    android:name=".MyApplication"
    ...
</application>
```
然后说说Application的获取问题，一个方法是我们直接 **(MyApplication)getApplication()**，但是还有一种更常见的做法，要在其他没有Context的地方也能拿到怎么办呢？可以这样，仿照单例的做法（只是仿照！），在MyApplication声明一个静态变量
```
public class MyApplication extends Application {
    private static MyApplication instance;
}
@Override
    public void onCreate() {
        super.onCreate();
        instance = this;
}
 // 获取ApplicationContext
    public static Context getMyApplication() {
        return instance;
}
```

至此我们拿到了MyApplication实例，注意这跟我们常见的单例不一样，不要自作聪明去在**getMyApplication**里面做一下空的判断，**Application**在应用中本来就是一个单例，所以每次返回的都是同一个实体，原文如下：
> There is normally no need to subclass Application.  In most situation, static singletons can provide the same functionality in a more modular way.

## 总结
**Application**，**Activity**，**Service**都是继承自**Context**，是应用运行时的环境，我们可以把**Application**看做是应用，**Activity**看做是一个界面，至于**getApplicationContext**和**getApplication**，在Activity和Service中他们返回的对象是一样的，但是在某些特殊情况下并不保证都是一样，所以如果想要拿到在**manifest**里面声明的那个**Application**，务必用**getApplication**，贴下原文：
> getApplication() is available to Activity and Services only. Although in current Android Activity and Service implementations, getApplication() and getApplicationContext() return the same object, there is no guarantee that this will always be the case (for example, in a specific vendor implementation). So if you want the Application class you registered in the Manifest, you should never call getApplicationContext() and cast it to your application, because it may not be the application instance (which you obviously experienced with the test framework).

还有就是因为他们都继承自Context，比如在打开Dialog的时候好像是都可以，其实不然，比如我们大多数情况：

```
 AlertDialog.Builder builder = new Builder(Activity.this);//可以
 AlertDialog.Builder builder = new Builder(getApplicationContext());//内存泄漏
```
如果把**this**换成**getApplicationContext()**，不会报错，但是就如我们刚才所说，**getApplicationContext()** 返回的上下文会随着应用一直存在，而这里的Dialog应该属于Activity，Activity关闭了我们无法销毁上下文（Dialog持有全局的上下文）。

所以在使用的时候要注意具体的使用场景,避免内存泄漏问题。

[这里顺便附上一个stackoverflow对于这个问题的链接](http://stackoverflow.com/questions/5018545/getapplication-vs-getapplicationcontext)

---
这篇文章还有个续集和补充
[到底getApplicationContext和getApplication是不是返回同一个对象？](http://blog.csdn.net/ly502541243/article/details/52127806)