# AsyncLayoutInflater

## 原理

在异步线程进行View的inflate操作，然后通过Handler回调主线程，优化页面初始化速度。



## inflate()

```java
@UiThread
public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
                    @NonNull OnInflateFinishedListener callback) {
    if (callback == null) {
        throw new NullPointerException("callback argument may not be null!");
    }
    InflateRequest request = mInflateThread.obtainRequest();
    request.inflater = this;
    request.resid = resid;
    request.parent = parent;
    request.callback = callback;
    mInflateThread.enqueue(request);
}

private static class InflateRequest {
    AsyncLayoutInflater inflater;
    ViewGroup parent;
    int resid;
    View view;
    OnInflateFinishedListener callback;

    InflateRequest() {
    }
}

public interface OnInflateFinishedListener {
    void onInflateFinished(@NonNull View view, @LayoutRes int resid,
            @Nullable ViewGroup parent);
}
```

inflate时需要传入`OnInflateFinishedListener`，在inflate结束时回调。

inflate操作被封装成`InflateRequest`请求放入`InflateThread`的等待队列中。



## InflateThread

```java
private static class InflateThread extends Thread {
    private ArrayBlockingQueue<InflateRequest> mQueue = new ArrayBlockingQueue<>(10);
    private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);
}
```

* `mQueue`：`InflateRequest`阻塞队列。
* `mRequestPool`：`InflateRequest`对象池。

```java
private static class InflateThread extends Thread {

    @Override
    public void run() {
        while (true) {
            runInner();
        }
    }

    public void runInner() {
        InflateRequest request;
        try {
            request = mQueue.take();
        } catch (InterruptedException ex) {
            // Odd, just continue
            Log.w(TAG, ex);
            return;
        }

        try {
            request.view = request.inflater.mInflater.inflate(
                request.resid, request.parent, false);
        } catch (RuntimeException ex) {
            // Probably a Looper failure, retry on the UI thread
            Log.w(TAG, "Failed to inflate resource in the background! Retrying on the UI"
                  + " thread", ex);
        }
        Message.obtain(request.inflater.mHandler, 0, request)
            .sendToTarget();
    }
}
```

循环取出阻塞队列中任务，在异步线程进行View的inflate操作，然后用过`Handler`回调给主线程。



## Handler

```java
public AsyncLayoutInflater(@NonNull Context context) {
    mInflater = new BasicInflater(context);
    mHandler = new Handler(mHandlerCallback);
    mInflateThread = InflateThread.getInstance();
}
```

需要在主线程初始化`AsyncLayoutInflator`，创建了主线程Handler。

```java
private Callback mHandlerCallback = new Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        InflateRequest request = (InflateRequest) msg.obj;
        if (request.view == null) {
            request.view = mInflater.inflate(
                    request.resid, request.parent, false);
        }
        request.callback.onInflateFinished(
                request.view, request.resid, request.parent);
        mInflateThread.releaseRequest(request);
        return true;
    }
};
```

通过`Handler.Callback`进行消息的处理，回调`OnInflateFinishedListener`。