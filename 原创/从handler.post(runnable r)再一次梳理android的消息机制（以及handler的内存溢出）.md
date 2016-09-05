## Handler
每个初学Android开发的都绕不开Handler这个“坎”，为什么说是个坎呢，首先这是Android架构的精髓之一，其次大部分人都是知其然却不知其所以然。今天看到Handler.post这个方法之后决定再去翻翻源代码梳理一下Handler的实现机制。
## 异步更新UI
先来一个必背口诀“**主线程不做耗时操作，子线程不更新UI**”，这个规定应该是初学必知的，那要怎么来解决口诀里的问题呢，这时候Handler就出现在我们面前了（AsyncTask也行，不过本质上还是对Handler的封装），来一段经典常用代码(这里忽略内存泄露问题，我们后面再说)：

首先在Activity中新建一个handler:

```
private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case 0:
                    mTestTV.setText("This is handleMessage");//更新UI
                    break;
            }
        }
    };
```

然后在子线程里发送消息：


```
new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);//在子线程有一段耗时操作,比如请求网络
                    mHandler.sendEmptyMessage(0);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
```

至此完成了在子线程的耗时操作完成后在主线程异步更新UI，可是并没有用上标题的post,我们再来看post的版本：

```
new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);//在子线程有一段耗时操作,比如请求网络
                    Handler handler = new Handler();
                    handler.post(new Runnable() {
                        @Override
                        public void run() {
                            mTestTV.setText("This is post");//更新UI
                        }
                    });
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
```

从表面上来看，给post方法传了个Runnable，像是开了个子线程，可是在子线程里并不能更新UI啊，那么问题来了，这是怎么个情况呢？带着这个疑惑，来翻翻Handler的源码：

> 先来看看普通的sendEmptyMessage是什么样子：
```
public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }
```

```
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
```
> 将我们传入的参数封装成了一个消息，然后调用sendMessageDelayed：

```
public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
> 再调用sendMessageAtTime：

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
好了，我们再来看post():

```
public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);//getPostMessage方法是两种发送消息的不同之处
    }
```
方法只有一句，内部实现和普通的sendMessage是一样的，但是只有一点不同,那就是 **getPostMessage(r)** 这个方法：

```
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```
这个方法我们发现也是将我们传入的参数封装成了一个消息，只是这次是**m.callback = r**,刚才是**msg.what=what**,至于Message的这些属性就不看了
## Android消息机制
看到这里，我们只是知道了post和sendMessage原理都是封装成Message，但是还是不清楚Handler的整个机制是什么样子，继续探究下去。

刚才看到那两个方法到最终都调用了**sendMessageAtTime**

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```
这个方法又调用了 **enqueueMessage**，看名字应该是把消息加入队列的意思，点进去看下：

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
mAsynchronous这个异步有关的先不管，继续将参数传给了queue的enqueueMessage方法，至于那个**msg**的**target**的赋值我们后面再看，现在继续进入MessageQueue类的enqueueMessage方法，方法较长，我们看看关键的几行：

```
Message prev;
for (;;) {
    prev = p;
    p = p.next;
    if (p == null || when < p.when) {
        break;
    }
    if (needWake && p.isAsynchronous()) {
        needWake = false;
    }
}
msg.next = p; // invariant: p == prev.next
prev.next = msg;
```
果然像方法名说的一样，一个无限循环将消息加入到消息队列中（链表的形式）,但是有放就有拿，这个消息怎样把它取出来呢？

翻看MessageQueue的方法，我们找到了**next()**，代码太长，不赘述，我们知道它是用来把消息取出来的就行了。不过这个方法是在什么地方调用的呢，不是在Handler中，我们找到了**Looper**这个关键人物，我叫他*环形使者*，专门负责从消息队列中拿消息，关键代码如下：

```
for (;;) {
     Message msg = queue.next(); // might block
     ...
     msg.target.dispatchMessage(msg);
     ...
     msg.recycleUnchecked();
}
```
简单明了，我们看到了我们刚才说的**msg.target**，刚才在Handler中赋值了**msg.target=this**，所以我们来看Handler中的dispatchMessage：

```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
1. msg的callback不为空，调用**handleCallback**方法（**message.callback.run()**）
2. mCallback不为空，调用**mCallback.handleMessage(msg)**
3. 最后如果其他都为空，执行Handler自身的 **handleMessage(msg)** 方法

msg的callback应该已经想到是什么了，就是我们通过Handler.post(Runnable r)传入的Runnable的run方法，这里就要提提java基础了，直接调用线程的run方法相当于是在一个普通的类调用方法，还是在当前线程执行，并不会开启新的线程。

**所以到了这里，我们解决了开始的疑惑，为什么在post中传了个Runnable还是在主线程中可以更新UI**。

继续看如**果msg.callback**为空的情况下的**mCallback**，这个要看看构造方法：

```
1.
public Handler() {
        this(null, false);
    }
2.    
public Handler(Callback callback) {
        this(callback, false);
    }
3.
public Handler(Looper looper) {
        this(looper, null, false);
    }
4.
public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);
    }
5.
public Handler(boolean async) {
        this(null, async);
    }
6.
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
7.
public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

```
具体的实现就只有最后两个，已经知道**mCallback**是怎么来的了，在构造方法中传入就行。

最后如果这两个回调都为空的话就执行Handler自身的handleMessage(msg)方法，也就是我们熟知的新建Handler重写的那个handleMessage方法。
## Looper
看到了这里有一个疑惑，那就是我们在新建Handler的时候并没有传入任何参数，也没有哪里显示调用了Looper有关方法，那Looper的创建以及方法调用在哪里呢？其实这些东西Android本身已经帮我们做了，在程序入口**ActivityThread**的main方法里面我们可以找到：

```
 public static void main(String[] args) {
    ...
    Looper.prepareMainLooper();
    ...
    Looper.loop();
    ...
```



## 总结

已经大概梳理了一下Handler的消息机制，以及post方法和我们常用的sendMessage方法的区别。来总结一下，主要涉及四个类**Handler、Message、MessageQueue、Looper**：

新建**Handler**，通过sendMessage或者post发送消息，**Handler**调用**sendMessageAtTime**将**Message**交给**MessageQueue**

---

**MessageQueue.enqueueMessage**方法将**Message**以链表的形式放入队列中

---

**Looper**的**loop**方法循环调用**MessageQueue.next()**取出消息，并且调用**Handler**的**dispatchMessage**来处理消息

---

在**dispatchMessage**中，分别判断**msg.callback、mCallback**也就是post方法或者构造方法传入的不为空就执行他们的回调，如果都为空就执行我们最常用重写的**handleMessage**。

---


## 最后谈谈handler的内存泄露问题
再来看看我们的新建Handler的代码：
```
private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            ...
        }
    };
```

当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有Activity的引用。

而Handler通常会伴随着一个耗时的后台线程一起出现，这个后台线程在任务执行完毕后发送消息去更新UI。然而，如果用户在网络请求过程中关闭了Activity，正常情况下，Activity不再被使用，它就有可能在GC检查时被回收掉，但由于这时线程尚未执行完，而该线程持有Handler的引用（不然它怎么发消息给Handler？），这个Handler又持有Activity的引用，就导致该Activity无法被回收（即内存泄露），直到网络请求结束。

另外，如果执行了Handler的postDelayed()方法，那么在设定的delay到达之前，会有一条**MessageQueue -> Message -> Handler -> Activity**的链，导致你的Activity被持有引用而无法被回收。

解决方法之一，使用弱引用：

```
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
```





