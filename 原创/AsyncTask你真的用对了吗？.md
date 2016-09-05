## 前言
在之前的文章深入探究了**Handler**，《[从Handler.post(Runnable r)再一次梳理Android的消息机制（以及handler的内存泄露）](http://blog.csdn.net/ly502541243/article/details/52062179)》我们知道了Android的消息机制主要靠**Handler**来实现，但是在**Handler**的使用中，忽略**内存泄露**的问题，不管是代码量还是理解程度上都显得有点不尽人意，所以Google官方帮我们在**Handler**的基础上封装出了**AsyncTask**。但是在使用**AsyncTask**的时候有很多细节需要注意，它的优点到底体现在哪里？还是来看看源码一探究竟。
## 怎么使用
来一段平常简单使用AsyncTask来异步操作UI线程的情况，首先新建一个类继承AsyncTask，构造函数传入我们要操作的组件（**ProgressBar**和**TextView**）

```
class MAsyncTask extends AsyncTask<Void, Integer, String>{
        private ProgressBar mProgressBar;
        private TextView mTextView;

        public MAsyncTask(ProgressBar mProgressBar, TextView mTextView) {
            this.mProgressBar = mProgressBar;
            this.mTextView = mTextView;
        }

        @Override
        protected void onPreExecute() {
            mTextView.setText("开始执行");
            super.onPreExecute();
        }

        @Override
        protected String doInBackground(Void... params) {
            for(int i = 0; i <= 100; i++){
                publishProgress(i);//此行代码对应下面onProgressUpdate方法
                try {
                    Thread.sleep(100);//耗时操作，如网络请求
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            return "执行完毕";
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            mProgressBar.setProgress(values[0]);
            super.onProgressUpdate(values);
        }

        @Override
        protected void onPostExecute(String s) {
            mTextView.setText(s);
            super.onPostExecute(s);
        }
    }
```
在Activity中创建我们新建的**MAsyncTask**实例并且执行(无关代码省略)：

```
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        MAsyncTask asyncTask = new MAsyncTask(mTestPB, mTestTV);
        asyncTask.execute();//开始执行
        ...
    }
}
```

## 看看原理
在上面的代码，我们开了个单一的线程来执行了一个简单的异步更新UI的操作（哈哈，可能会觉得**AsyncTask**有些大材小用了哈），现在来看看AsyncTask具体是怎么实现的，先从构造方法开始：

```
public abstract class AsyncTask<Params, Progress, Result> 
```
AsyncTask为抽象类，并且有三个泛型，我觉得这三个泛型是很多使用者不懂的根源：

- **params**：参数，在**execute()** 传入，可变长参数，跟**doInBackground(Void... params)** 这里的**params**类型一致，我这里没有传参数，所以可以将这个泛型设置为**Void**
- **Progress**：执行的进度，跟**onProgressUpdate(Integer... values)** 的**values**的类型一致，一般情况为**Integer**
- **Result**：返回值，跟**String doInBackground** 返回的参数类型一致，且跟**onPostExecute(String s)** 的**s**参数一致，在耗时操作执行完毕调用。我这里执行完毕返回了个字符串，所以为String

看了这三个泛型，我们就基本上了解了**AsyncTask**的执行过程，主要就是上面代码重写的那几个方法，现在来仔细看，首先在继承AsyncTask时有个抽象方法必须重写:

```
@WorkerThread
protected abstract Result doInBackground(Params... params);
```
顾名思义，这个方法是在后台执行，也就是在子线程中执行，需要子类来实现，在这个方法里面我们可以调用**publishProgress**来发送进度给UI线程，并且在**onProgressUpdate**方法中接收。

根据调用顺序，我们一般会重写这几个方法：

```
//在doInBackground之前调用，在UI线程内执行
@MainThread
protected void onPreExecute() {
}
//在执行中，且在调用publishProgress方法时，在UI线程内执行，用于更新进度
@MainThread
protected void onProgressUpdate(Progress... values) {
}
//在doInBackground之后调用，在UI线程内执行
@MainThread
protected void onPostExecute(Result result) {
}
```
我们来看看这个**publishProgress**方法是怎么来调用**onProgressUpdate**方法的：

```
@WorkerThread
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```
使用**obtainMessage**是避免重复创建消息，调用了getHandler()然后发送消息,这里是一个单例

```
    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }
```
返回了一个**InternalHandler**：

```
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
在判断消息为**MESSAGE_POST_PROGRESS**后我们发现其实内部就是调用了**Handler**来实现这一切，包括执行结束时调用**finish**方法，这个我们后面再说。从头来看一下**AsyncTask**的执行过程，来到execute方法：

```
/**
This method must be invoked on the UI thread.(这行注释为Google官方注释)
*/
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```
**注意！此方法必须在UI线程调用**，这里就不做测试了。在这里又调用**executeOnExecutor**:
```
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    ......

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

我们发现在UI线程先调用了**onPreExecute()**，将传入的参数赋值给**mWorker.mParams**，然后调用了参数exec的**execute**方法，并且将**mFuture**作为参数传入，这里就设计到了三个对象：**sDefaultExecutor**(在**executeOnExecutor**中传入)、**mWorker**、**mFuture**，来看看它们的赋值在哪里：

#### sDefaultExecutor：

```
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
......
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
......
private static class SerialExecutor implements Executor {
   final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
   Runnable mActive;

   public synchronized void execute(final Runnable r) {
       mTasks.offer(new Runnable() {
           public void run() {
               try {
                   r.run();
               } finally {
                   scheduleNext();
               }
           }
       });
       if (mActive == null) {
           scheduleNext();
       }
   }

   protected synchronized void scheduleNext() {
       if ((mActive = mTasks.poll()) != null) {
           THREAD_POOL_EXECUTOR.execute(mActive);
       }
   }
}
```
我们发现sDefaultExecutor的赋值默认就是SERIAL_EXECUTOR，也就是一个顺序执行的线程池，内部实现有一个任务队列。

#### mWorker
```
private final WorkerRunnable<Params, Result> mWorker;

public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
        ......
}

private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
}

```
在AsyncTask的构造方法中，给**mWorker**赋值为一个**Callable**（带返回参数的线程，涉及到java并发的一些基础知识，这里不赘述），并且在call方法中执行了**doInBackground**方法，最后调用**postResult**方法

#### mFuture

```
private final FutureTask<Result> mFuture;
public AsyncTask() {
        ......
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
}
```
mFuture为FutureTask类型，这里将**mWorker**传入，在**mWorker**执行完毕后调用**postResultIfNotInvoked**方法，我们先看看这个方法：
```
private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}
```
其实这个方法也最后调用了**postResult**，在这之前做了个有没调用的判断，确保任务执行完毕后调用此方法。来看看**postResult**方法：

```
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
又看到了熟悉的**obtainMessage**和**sendToTarget**发送消息，这次消息内容变为**MESSAGE_POST_RESULT**，再来看看我们刚才已经提到的**InternalHandler**的**handleMessage**方法：

```
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
```
最后根据消息类型，这里调用了result.mTask.finish，result类型为**AsyncTaskResult**：

```
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```
mTask的类型为**AsyncTask**，找到AsyncTask的finish方法：

```
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
最后如果没有取消的话调用了**onPostExecute**，也就是我们之前重写的那个方法，在执行完毕后调用，并且此方法也在子线程。
## 多线程并发
正如开题所说，AsyncTask本质上就是对Handler的封装，在执行之前，执行中，执行完毕都有相应的方法，使用起来也一目了然，不过这还并不是**AsyncTask**的最大的优点，**AsyncTask**最适合使用的场景是多线程，开始在代码中已经看到了在**AsyncTask**内部有自己维护的线程池，默认的是SERIAL_EXECUTOR
```
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
按照顺序执行，一个任务执行完毕再执行下一个，还提供有一个支持并发的线程池：

```
//获取CPU数目
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
//核心工作线程（同时执行的线程数）
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
//线程池允许的最大线程数目
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
//空闲线程超时时间（单位为S）
private static final int KEEP_ALIVE = 1;
//线程工厂
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};
//阻塞队列，用来保存待执行的任务（最高128个）
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR
 = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

```
声明为**static**，多个实例同用一个线程池，这个是Googl官方自带的一个根据cpu数目来优化的线程池，使用方法如下：

```
for(int i = 0; i < 100; i++) {//模拟100个任务，不超过128
    MAsyncTask asyncTask = new MAsyncTask(mTestPB, mTestTV);
    asyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
}
```
在**executeOnExecutor**中我们还可以传入自己自定义的线程池：

```
//跟默认一样的按顺序执行
asyncTask.executeOnExecutor(Executors.newSingleThreadExecutor());
//无限制的Executor
asyncTask.executeOnExecutor(Executors.newCachedThreadPool());
//同时执行数目为10的Executor
asyncTask.executeOnExecutor(Executors.newFixedThreadPool(10));
```
## 总结
AsyncTask使用起来的确很简单方便，内部也是Android的消息机制，并且很快捷的实现了异步更新UI，特别是多线程时也可以很好的表现，这个是我们单独使用Handler时不具备的，但是在使用过程中注意内部方法的调用顺序以及调用的时机，比如**asyncTask.execute()** 要在UI主线程中调用，在子线程中调用是不可以的，还有就是在使用时根据情况来决定到底应该用哪种线程池。

