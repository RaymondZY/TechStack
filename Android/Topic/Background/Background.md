# Background

## AsyncTask

### 构造

构造会创建`WorkerRunnable`，它是一个`Callable`对象，还会构造一个`Future`

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}

public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);

    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                
                // doInBackground
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {

                // PostResult
                postResult(result);
            }
            return result;
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
                // 取消后回调
                postResultIfNotInvoked(null);
            }
        }
    };
}

private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                                                 new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

### 线程池

线程池默认，核心线程数1，最大线程池大小是20，线程保留时间3秒。如果发生了`RejectedExecutionException`，会切换到备用的线程池备用线程池线程数为5。这线程池是静态变量，意味着所有的`AsyncTask`将公用这一个线程池。

```java
private static final int CORE_POOL_SIZE = 1;
private static final int MAXIMUM_POOL_SIZE = 20;
private static final int BACKUP_POOL_SIZE = 5;
private static final int KEEP_ALIVE_SECONDS = 3;
public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
        CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), sThreadFactory);
    threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}

private static final RejectedExecutionHandler sRunOnSerialPolicy =
    new RejectedExecutionHandler() {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        android.util.Log.w(LOG_TAG, "Exceeded ThreadPoolExecutor pool size");
        // As a last ditch fallback, run it on an executor with an unbounded queue.
        // Create this executor lazily, hopefully almost never.
        synchronized (this) {
            if (sBackupExecutor == null) {
                sBackupExecutorQueue = new LinkedBlockingQueue<Runnable>();
                sBackupExecutor = new ThreadPoolExecutor(
                    BACKUP_POOL_SIZE, BACKUP_POOL_SIZE, KEEP_ALIVE_SECONDS,
                    TimeUnit.SECONDS, sBackupExecutorQueue, sThreadFactory);
                sBackupExecutor.allowCoreThreadTimeOut(true);
            }
        }
        sBackupExecutor.execute(r);
    }
};
```

### Handler

内部使用静态的主线程`Handler`回调`finish`和`onProgressUpdate`。

```java
private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}

private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
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

private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

### executeOnExecutor()

默认会调用到`executeOnExecutor()`，传入的`Executor`是默认的`SerialExecutor`。它用一个`ArrayDeque`维护了需要执行的任务，不断调用线程池进行执行。

```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        // 重新进行Runnable包装，让每次执行完后进行下一次schedule
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        // 这里为了首次启动
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

public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

```java
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

    // 回调onPreExecute()
    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

### onPreExecute()

重写方法，在开始执行前在主线程回调。

```java
protected void onPreExecute() {
}
```

### doInBackground()

重写方法，在异步线程回调。

```java
protected abstract Result doInBackground(Params... params);
```

### publishProgress()

在`doInBackground()`中调用，发布进度状态。最后会回调到主线程。

```java
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```

### onProgressUpdate()

重写方法，在主线程回调。

```java
protected void onProgressUpdate(Progress... values) {
}
```

### onPostExecute()

```java
protected void onPostExecute(Result result) {
}
```

### cancel()

`mayInterruptIfRunning`：是否intercept正在执行的future。

```java
public final boolean cancel(boolean mayInterruptIfRunning) {
    mCancelled.set(true);
    return mFuture.cancel(mayInterruptIfRunning);
}
```

### get()

同步等待结果

```java
public final Result get() throws InterruptedException, ExecutionException {
    return mFuture.get();
}
```

### 缺点

* 生命周期不会随Activity结束而结束，并且会引起内存泄漏。
* 旋转屏幕Activity重建后，`onPostExecute()`结果更新界面不再生效。

* 所有的`AsyncTask`公用一个线程池都是串行执行的。