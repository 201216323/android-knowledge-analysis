# 从源码角度一步步分析AsyncTask的用法与原理

AsyncTask 是Android特有的一个轻量级异步抽象类，在类中通过```doInBackground()```在子线程执行耗时操作，执行完毕在主线程调用```onPostExecute()```。

## 前言
众所周知，Android视图的绘制、监听、事件等都UI线程（主线程，Main Thread）执行，如果执行访问网络请求、数据库等耗时操作，可能会阻塞主线程，若阻塞时间超过5秒，可能会引起系统的ANR异常（Application Not Responding）。所以耗时操作需要放在子线程（也称为Worker Thread）执行，这样就避免了主线程的阻塞，然而在线程是不能有更新UI的操作的，比如在子线程调用```TextView.setText()```就会引发以下错误：
```
Only the original thread that created a view hierarchy can touch its views.
```
故而可以用 “Handle + Thread”的方式，子线程执行耗时操作，通过Handler通知主线程更新UI。但是这个方式略微麻烦，于是便引入了AsyncTask。

## AsyncTask特点
AsyncTask<Params, Progress, Result> 定义了三种泛型，作用如下：

- Params：启动AsyncTask任务所需要的参数，在```doInBackground```方法接收
- Progress：后台任务执行进度
- Result：后台任务执行完毕之后回馈给```onPostExecute```的结果

AsyncTask 内部提供了多个方法，子类可重写，会自动在相应的时刻自动调用，并且用户最好不好手动调用以下的方法，避免引起不可预知的错误。以下方法只有```doInBackground```方法是必须重写的，其他方法可选。

- **onPreExecute()**：任务执行之前调用
- **doInBackground(Params... params)**：子线程执行任务，在这个方法中是不可以进行UI更新操作的，如果需要更新UI元素，例如展示该任务的执行进度，可以调用publishProgress(Progress... progress)方法来完成。
- **onProgressUpdate(Progress... values)**：任务执行进度。可用于进度条显示
- **onPostExecute(Result result)**：任务执行完毕
- **onCancelled(Result reslut)**、**onCancelled()**：用户取消任务

最后，调用```execute()```开启异步任务。

## AsyncTask简单使用
```java
public void request() {

    AsyncTask<String, Integer, Integer> task = new AsyncTask<String, Integer, Integer>() {

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            Log.i("AsyncTask", "准备执行后台任务");
        }

        @Override
        protected Integer doInBackground(String... params) {
            String url_1 = params[0];

            doRequest(url_1);

            publishProgress(50);

            String url_2 = params[1];

            doRequest(url_2);

            publishProgress(100);

            return 1;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            Log.i("AsyncTask", "当前进度" + values[0] + "%");
        }

        @Override
        protected void onPostExecute(Integer ret) {
            super.onPostExecute(ret);
            Log.i("AsyncTask", "执行完毕，执行结果" + ret);
        }
    };

    String url_1 = "https://api.github.com/users/smuyyh";
    String url_2 = "https://api.github.com/users/smuyyh/followers";

    task.execute(url_1, url_2); // 开启后台任务
}
```
代码较为简单，就不做过多的解释了。后面着重介绍AsyncTask的内部实现机制。

## 原理分析
这一节将从源码的角度来分析一下[AsyncTask][1]。以下源码基于Android-23.

- 开启后台任务之前，首先需要创建AsyncTask的实例，所以还得从构造函数说起。

```java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();
            return postResult(result); // 任务的具体实现
        }
    };

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
在构造函数中，初始化了两个变量，分别是mWorker与mFuture，mFuture创建的时候传入了mWorker参数，而mWorker本身是一个[Callable][2]对象。那么，mFutrue是个什么东西呢？

mFuture是一个[FutureTask][3]对象，FutureTask实际上是一个任务的操作类，它并不启动新线程，并且只负责任务调度。任务的具体实现是构造FutureTask时提供的，实现自Callable<V>接口，也就是刚才的mWorker。

 - AsyncTask对象穿件完毕之后调用```execute(Params...)```执行，跟进看看
```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```
只有一句话，可知是调用executeOnExecutor进行执行。这里就有个疑问了，sDefaultExecutor是个什么东西？在说这个之前，需要明确一下一下三个事：

**1**、Android3.0之前部分代码
```java
private static final int CORE_POOL_SIZE = 5;  
private static final int MAXIMUM_POOL_SIZE = 128;  
private static final int KEEP_ALIVE = 10;  

private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,  MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                TimeUnit.SECONDS, sWorkQueue, sThreadFactory); 
```
**2**、在Android3.0之前，AsyncTask执行中最终触发的是把任务交给线池THREAD_POOL_EXECUTOR来执行，提交的任务并行的在线程池中运行，但这些规则在3.0之后发生了变化，3.0之后提交的任务是默认串行运行的，执行完一个任务才执行下一个！

**3**、在Android3.0以前线程池里核心线程有5个，同时存在的线程数最大不能超过128个，线程池里的线程都是并行运行的，在3.0以后，直接调用execute(params)触发的是sDefaultExecutor的execute(runnable)方法，而不是原来的THREAD_POOL_EXECUTOR。在Android4.4以后，线程池大小等于 cpu核心数 + 1，最大值为cpu核心数 * 2 + 1。这些变化大家可以自行对比一下。

跟进源码不难发现，sDefaultExecutor实际上是指向SerialExecutor的一个实例，从名字上看是一个顺序执行的executor，并且它在AsyncTask中是以常量的形式存在的，因此在整个应用程序中的所有AsyncTask实例都会共用同一个SerialExecutor。
```java
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
SerialExecutor是使用ArrayDeque这个队列来管理Runnable对象的。当Executor提交一个任务，执行一次```execute()```，在这里向mTasks队列添加一个Runnable对象。初次添加任务时mActive为null，故接下来会执行```scheduleNext()```，将mActive指向刚刚添加的runbale，并提交到```THREAD_POOL_EXECUTOR```中执行。

当AsyncTask不断提交任务时，那么此时mActive不为空了，所以后续添加的任务能得到执行的唯一条件，就是前一个任务执行完毕，也就是```r.run()```。所以这就保证了SerialExecutor的顺序执行。这个地方其实也是一个坑，初学者很容易在这里踩坑，同时提交多个任务，却无法同步执行。

如果想让其并行执行怎么办？AsyncTask提供了一下两种方式：
```java
task.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR); 

task.executeOnExecutor(executor, params); //可以自己指定线程池
```
- 继续跟进```executeOnExecutor(Executor exec, Params... params)```代码
```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
不难看出，最后是调用Executor的```execute(Runnable command)```方法启动mFuture。默认情况下，sDefaultExecutor就是SerialExecutor类，所以为串行执行。当然用户也可以提供自己的Executor来改变AsyncTask的运行方式。最后在```THREAD_POOL_EXECUTOR```真正启动任务执行的Executor。

上面已经提到，Execute执行是调用Runnable的run()方法，也就是mFuture的run方法，继续跟进代码
```java
public void run() {
    if (state != NEW || !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
从第10行代码可发现，最后是调用callable的call()方法。那么这个callable是什么呢？就是初始化mFuture传入的mWorker对象。在前面的构造函数那边可以发现call()方法，我们单独分析一下这个方法
```java
public Result call() throws Exception {
    mTaskInvoked.set(true);

    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    Result result = doInBackground(mParams);
    Binder.flushPendingCommands();
    return postResult(result);
}
```
看了这么久，终于发现了```doInBackground()```，深深松了一口气。执行完之后得到的结果，传给```postResult(result)```。继续跟进
```java
private Result postResult(Result result) {
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```
可以发现，最后是通过Handler的方式，把消息发送出去，消息中携带了```MESSAGE_POST_RESULT```常量和一个表示任务执行结果的AsyncTaskResult对象。而```getHandler()```返回的sHandler是一个```InternalHandler```对象，InternalHandler源码如下所示：
```java
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

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
这里对消息的类型进行了判断，如果是MESSAGE_POST_RESULT，就执行```finish()```，如果是MESSAGE_POST_PROGRESS，就```onProgressUpdate()```方法。那么什么时候触发如果是MESSAGE_POST_PROGRESS消息呢？就是在publishProgress()方法调用的时候，publishProgress()方法用finial标记，说明子类不能重写他，不过可以手动调用，通知进度更新，这就表明了publishProgress可在子线程执行。
```java
protected final void publishProgress(Progress... values) {  
    if (!isCancelled()) {  
        sHandler.obtainMessage(MESSAGE_POST_PROGRESS,  
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();  
    }  
} 
```


然后看一下finish()的代码。
```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
不难发现，如果当前任务被取消掉了，就会调用onCancelled()方法，如果没有被取消，则调用onPostExecute()方法，这样当前任务的执行就全部结束了。并且，当你再次调用execute的时候，这个时候mStatus的状态为Status.FINISHED，表示已经执行过了，那么此时就会抛异常，这也就是为什么一个AsyncTask对象只能执行一次的原因。
```java
switch (mStatus) {
    case RUNNING:
        throw new IllegalStateException("Cannot execute task:"
                + " the task is already running.");
    case FINISHED:
        throw new IllegalStateException("Cannot execute task:"
                + " the task has already been executed "
                + "(a task can be executed only once)");
}
```
到这里，就非常清晰了吧。彻底的了解了AsyncTask内部实现的逻辑。

## 总结
可以看出，在使用AsyncTask的过程中，有许多需要注意的地方。

 1. 由于Handler需要和主线程交互，而Handler又是内置于AsnycTask中的，所以，AsyncTask的创建必须在主线程，execute的执行也应该在主线程。
 2. AsyncTask的doInBackground(Params... Params)方法运行在子线程中，其他方法运行在主线程中，可以操作UI组件。
 3. 尽量不要手动的去调用AsyncTask的onPreExecute, doInBackground, publishProgress, onProgressUpdate, onPostExecute 这些方法，避免发生不可预知的问题。
 4. 一个任务AsyncTask任务只能被执行一次。否则会抛IllegalStateException异常
 5. 在doInBackground()中要检查isCancelled()的返回值，如果你的异步任务是可以取消的话。cancel()仅仅是给AsyncTask对象设置了一个标识位，虽然运行中可以随时调用cancel(boolean mayInterruptIfRunning)方法取消任务，如果成功调用isCancelled()会返回true，并且不会执行 onPostExecute() 方法了，取而代之的是调用 onCancelled() 方法。但是！！！值得注意的一点是，如果这个任务在执行之后调用cancel()方法是不会真正的把任务结束的，而是继续执行，只不过改变的是执行之后的回调方法是 onPostExecute还是onCancelled。可以在doInBackground里面去判断isCancle，如果取消了，那就直接return result; 当然，这种方式也并非非常完美。
 6. Asynctask的生命周期和它所在的activity的生命周期并非一致的，当Activity终止时，它会以它自有的方式继续运行，即使你退出了整个应用程序。另一方面，要注意Android屏幕切换问题，因为这时候Activity会重新创建。

  [1]: http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/os/AsyncTask.java
  [2]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7u40-b43/java/util/concurrent/Callable.java#Callable
  [3]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7u40-b43/java/util/concurrent/FutureTask.java#FutureTask
