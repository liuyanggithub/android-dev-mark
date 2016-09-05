## 前言
前几天看到一道面试题：**Thread、Handler和HandlerThread有什么区别？**，这个题目有点意思，对于很多人来说，可能对Thread和Handler很熟悉，主要涉及到Android的消息机制(Handler、Message、Looper、MessageQueue)，详见[《 从Handler.post(Runnable r)再一次梳理Android的消息机制（以及handler的内存泄露）》](http://blog.csdn.net/ly502541243/article/details/52062179)

但是这个**HandlerThread**是拿来做什么的呢？它是Handler还是Thread？我们知道Handler是用来异步更新UI的，更详细的说是用来做**线程间的通信**的，更新UI时是**子线程与UI主线程**之间的通信。那么现在我们要是想**子线程与子线程**之间的通信要怎么做呢？当然说到底也是用Handler+Thread来完成（不推荐，需要自己操作Looper），Google官方很贴心的帮我们封装好了一个类，那就是刚才说到的：**HandlerThread**。（类似的封装对于多线程的场景还有[AsyncTask](http://blog.csdn.net/ly502541243/article/details/52329861)）
## 使用方法
还是先来看看HandlerThread的使用方法：
首先新建HandlerThread并且执行start()

```
private HandlerThread mHandlerThread;
......
mHandlerThread = new HandlerThread("HandlerThread");
handlerThread.start();
```
创建Handler，使用mHandlerThread.getLooper()生成Looper：

```
        final Handler handler = new Handler(mHandlerThread.getLooper()){
            @Override
            public void handleMessage(Message msg) {
                System.out.println("收到消息");
            }
        };
```
然后再新建一个子线程来发送消息：
```
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);//模拟耗时操作
                    handler.sendEmptyMessage(0);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
```
最后一定不要忘了在onDestroy释放,避免内存泄漏：
```
    @Override
    protected void onDestroy() {
        super.onDestroy();
        mHandlerThread.quit();
    }
```

执行结果很简单，就是在控制台打印字符串：**收到消息**
## 原理
整个的使用过程我们根本不用去关心Handler相关的东西，只需要发送消息，处理消息，Looper相关的东西交给它自己去处理，还是来看看源码它是怎么实现的，先看构造方法：
```
public class HandlerThread extends Thread {}
```
HandlerThread其实还是一个线程，它跟普通线程有什么不同？
```
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    ......
}
```
答案是多了一个Looper，这个是子线程独有的Looper，用来做消息的取出和处理。继续看看HandlerThread这个线程的run方法：
```
    protected void onLooperPrepared() {
    }
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();//生成Looper
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();//空方法，在Looper创建完成后调用，可以自己重写逻辑
        Looper.loop();//死循环，不断从MessageQueue中取出消息并且交给Handler处理
        mTid = -1;
    }
```
主要就是做了一些Looper的操作，如果我们自己使用Handler+Thread来实现的话也要进行这个操作，再来看看getLooper()方法：

```
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
```
方法很简单，就是加了个同步锁，如果已经创建了(isAlive()返回true)但是mLooper为空的话就继续等待，直到mLooper创建成功，最后看看quit方法，值得一提的是有两个：

```
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
```
**quitSafely**是针对在消息队列中还有消息或者是延迟发送的消息没有处理的情况，调用这个方法后都会被停止掉。
## 总结
**HandlerThread**的使用方法还是比较简单的，但是我们要明白一点的是：**如果一个线程要处理消息，那么它必须拥有自己的Looper**，并不是Handler在哪里创建，就可以在哪里处理消息的。

如果不用HandlerThread的话，需要手动去调用Looper.prepare()和Looper.loop()这些方法。