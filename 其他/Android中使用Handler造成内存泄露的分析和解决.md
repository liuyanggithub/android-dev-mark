什么是内存泄露？
Java使用有向图机制，通过GC自动检查内存中的对象（什么时候检查由虚拟机决定），如果GC发现一个或一组对象为不可到达状态，则将该对象从内存中回收。也就是说，一个对象不被任何引用所指向，则该对象会在被GC发现的时候被回收；另外，如果一组对象中只包含互相的引用，而没有来自它们外部的引用（例如有两个对象A和B互相持有引用，但没有任何外部对象持有指向A或B的引用），这仍然属于不可到达，同样会被GC回收。
Android中使用Handler造成内存泄露的原因
Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        mImageView.setImageBitmap(mBitmap);
    }
}
上面是一段简单的Handler的使用。当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有一个外部类对象（通常是一个Activity）的引用（不然你怎么可能通过Handler来操作Activity中的View？）。而Handler通常会伴随着一个耗时的后台线程（例如从网络拉取图片）一起出现，这个后台线程在任务执行完毕（例如图片下载完毕）之后，通过消息机制通知Handler，然后Handler把图片更新到界面。然而，如果用户在网络请求过程中关闭了Activity，正常情况下，Activity不再被使用，它就有可能在GC检查时被回收掉，但由于这时线程尚未执行完，而该线程持有Handler的引用（不然它怎么发消息给Handler？），这个Handler又持有Activity的引用，就导致该Activity无法被回收（即内存泄露），直到网络请求结束（例如图片下载完毕）。另外，如果你执行了Handler的postDelayed()方法，该方法会将你的Handler装入一个Message，并把这条Message推到MessageQueue中，那么在你设定的delay到达之前，会有一条MessageQueue -> Message -> Handler -> Activity的链，导致你的Activity被持有引用而无法被回收。
内存泄露的危害
只有一个，那就是虚拟机占用内存过高，导致OOM（内存溢出），程序出错。对于Android应用来说，就是你的用户打开一个Activity，使用完之后关闭它，内存泄露；又打开，又关闭，又泄露；几次之后，程序占用内存超过系统限制，FC。
使用Handler导致内存泄露的解决方法
方法一：通过程序逻辑来进行保护。
1.在关闭Activity的时候停掉你的后台线程。线程停掉了，就相当于切断了Handler和外部连接的线，Activity自然会在合适的时候被回收。
2.如果你的Handler是被delay的Message持有了引用，那么使用相应的Handler的removeCallbacks()方法，把消息对象从消息队列移除就行了。
方法二：将Handler声明为静态类。
静态类不持有外部类的对象，所以你的Activity可以随意被回收。代码如下：
static class MyHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        mImageView.setImageBitmap(mBitmap);
    }
}
但其实没这么简单。使用了以上代码之后，你会发现，由于Handler不再持有外部类对象的引用，导致程序不允许你在Handler中操作Activity中的对象了。所以你需要在Handler中增加一个对Activity的弱引用（WeakReference）：
static class MyHandler extends Handler {
    WeakReference<Activity > mActivityReference;
    MyHandler(Activity activity) {
        mActivityReference= new WeakReference<Activity>(activity);
    }
    @Override
    public void handleMessage(Message msg) {
        final Activity activity = mActivityReference.get();
        if (activity != null) {
            mImageView.setImageBitmap(mBitmap);
        }
    }
}
将代码改为以上形式之后，就算完成了。
延伸：什么是WeakReference？
WeakReference弱引用，与强引用（即我们常说的引用）相对，它的特点是，GC在回收时会忽略掉弱引用，即就算有弱引用指向某对象，但只要该对象没有被强引用指向（实际上多数时候还要求没有软引用，但此处软引用的概念可以忽略），该对象就会在被GC检查到时回收掉。对于上面的代码，用户在关闭Activity之后，就算后台线程还没结束，但由于仅有一条来自Handler的弱引用指向Activity，所以GC仍然会在检查的时候把Activity回收掉。这样，内存泄露的问题就不会出现了。
