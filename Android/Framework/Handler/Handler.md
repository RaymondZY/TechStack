# Handler
`Handler`是系统提供的异步消息队列。

## 使用

### 创建Handler
`Handler`的构造函数需要传入`Looper`参数，默认构造函数使用当前线程的Looper。

```java
Handler handler = new Handler(handlerThread.getLooper());
```
如果当前线程没有调用过`Looper.prepare()`，那么使用时将会抛出异常。

### 发送消息
使用`Handler`发送消息主要分为两种操作：send操作和post操作。send操作用于发送`Message`消息对象，post操作用于发送一段`Runnable`。

send操作用于操作`Message`对象。

有如下方法：

* `sendEmptyMessage()`：发送一条空的消息。

  > 空的消息也会默认生成一条`Message`对象，但是它没有参数。

* `sendEmptyMessageAtTime()`：在固定时刻发送一条空的消息。

* `sendEmptyMessageDelay()`：延迟一段时间发送一条空的消息。

* `sendMessage()`：发送一条消息。 

* `sendMessageAtTime()`：在固定时刻发送一条消息。

* `sendMessageDelay()`：延迟一段时间发送一条消息。

* `sendMessageAtFrontOfQueue()`：在消息队列头部发送一条消息。

这些方法最后都会转化成`sendMessageAtTime()`方法的调用，然后调用`MessageQueue.enqueueMessage()`将消息发送给`MessageQueue`对象。

```java
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

```

post操作用于操作`Runnable`对象。

有如下方法：

* `post()`：发送一个`Runnable`。
* `postAtTime()`：在固定时刻发送一个`Runnable`。
* `postDelay()`：延迟一段时间发送一个`Runnable`。

`Runnable`最终也是转化为`Message`对象发送给消息队列，在`Message.callback`上记录post的`Runnable`对象。

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

### 处理消息

`Handler`的消息处理分发在`dispatchMessage()`方法中。

处理顺序如下：

* 如果`Message`记录了post的`Runnable`，执行`Runnable`。
* 如果构造`Handler`时定义了`Callback`，可以在`Callback`中优先拦截消息。
* 调用`handleMessage()`处理send相关操作。

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // post相关的操作
        handleCallback(msg);
    } else {
        // 如果构造Handler时传入的Callback优先处理了消息
        // Handler本身就没有机会处理了
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // send相关的操作
        handleMessage(msg);
    }
}
```

因此，`Handler`的子类需要重写`handleMessage()`方法处理消息。

### 删除消息

* `removeMessage()`：使用`Message.what`参数进行区分，删除send操作发送的消息。
* `removeCallback()`：使用`Message.callback`参数进行区分，删除post操作发送的消息。



## Message

包装了一条消息。

### 获取Message

推荐使用`Message.obtain()`方法获取一个`message`对象。它在内部维护了一个对象池，减少对象的创建。

### 参数

* `what`用来区分`Message`的种类。
* `data`用来存储复杂的数据。调用`setData()`设置。
* `target`用来存储最终执行的`Handler`。



## MessageQueue

消息队列类。
* `enqueueMessage()`方法添加一条消息。

* `next()`方法获取下一条消息。  

  内部会进行一个死循环，直到获取到下一条执行的`Message`，虽然是死循环但是它会调用`nativePollOnce()`进行阻塞。

  另外，会在每次空闲的时候进行`IdleHandler`的处理。

  ```java
  Message next() {
      // Return here if the message loop has already quit and been disposed.
      // This can happen if the application tries to restart a looper after quit
      // which is not supported.
      final long ptr = mPtr;
      if (ptr == 0) {
          return null;
      }
  
      int pendingIdleHandlerCount = -1; // -1 only during first iteration
      int nextPollTimeoutMillis = 0;
      for (;;) {
          if (nextPollTimeoutMillis != 0) {
              Binder.flushPendingCommands();
          }
  
          /* ----- 调用native epoll()方法阻塞 ----- */
          nativePollOnce(ptr, nextPollTimeoutMillis);
  
          synchronized (this) {
              final long now = SystemClock.uptimeMillis();
              Message prevMsg = null;
              Message msg = mMessages;
              if (msg != null && msg.target == null) {
                  do {
                      prevMsg = msg;
                      msg = msg.next;
                  } while (msg != null && !msg.isAsynchronous());
              }
              if (msg != null) {
                  if (now < msg.when) {
                      /* ----- 如果Message设置的时间还未到继续循环 ----- */
                      // Next message is not ready.  Set a timeout to wake up when it is ready.
                      nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                  } else {
                      /* ----- 如果已经到了Message设置的时间从队列中弹出 ----- */
                      // Got a message.
                      mBlocked = false;
                      if (prevMsg != null) {
                          prevMsg.next = msg.next;
                      } else {
                          mMessages = msg.next;
                      }
                      msg.next = null;
                      if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                      msg.markInUse();
                      return msg;
                  }
              } else {
                  // No more messages.
                  nextPollTimeoutMillis = -1;
              }
  
              // Process the quit message now that all pending messages have been handled.
              if (mQuitting) {
                  dispose();
                  return null;
              }
  
              // If first time idle, then get the number of idlers to run.
              // Idle handles only run if the queue is empty or if the first message
              // in the queue (possibly a barrier) is due to be handled in the future.
              if (pendingIdleHandlerCount < 0
                      && (mMessages == null || now < mMessages.when)) {
                  pendingIdleHandlerCount = mIdleHandlers.size();
              }
              if (pendingIdleHandlerCount <= 0) {
                  // No idle handlers to run.  Loop and wait some more.
                  mBlocked = true;
                  continue;
              }
  
              if (mPendingIdleHandlers == null) {
                  mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
              }
              mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
          }
  
          /* ----- 处理IdleHandler ----- */
          // Run the idle handlers.
          // We only ever reach this code block during the first iteration.
          for (int i = 0; i < pendingIdleHandlerCount; i++) {
              final IdleHandler idler = mPendingIdleHandlers[i];
              mPendingIdleHandlers[i] = null; // release the reference to the handler
  
              boolean keep = false;
              try {
                  keep = idler.queueIdle();
              } catch (Throwable t) {
                  Log.wtf(TAG, "IdleHandler threw exception", t);
              }
  
              if (!keep) {
                  synchronized (this) {
                      mIdleHandlers.remove(idler);
                  }
              }
          }
  
          // Reset the idle handler count to 0 so we do not run them again.
          pendingIdleHandlerCount = 0;
  
          // While calling an idle handler, a new message could have been delivered
          // so go back and look again for a pending message without waiting.
          nextPollTimeoutMillis = 0;
      }
  }
  ```



## Looper

在当前运行的线程启动一个消息队列`MessageQueue`，并开始循环处理队列消息 。

整个过程分为两个步骤：

* `Looper.prepare()`
  
    ```java
    public static void prepare() {
        prepare(true);
    }
    
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    ```
`prepare()`的核心代码是新建`Looper`并放入`ThreadLocal`中。
    
* `Looper.loop()`
  
    ```java
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
    
    public static void loop() {
            final Looper me = myLooper();
            if (me == null) {
                throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
            }
        final MessageQueue queue = me.mQueue;
    
            // 省略的代码 ...
            for (;;) {
                Message msg = queue.next(); // might block
                // 省略的代码 ...
            }
        }
    ```
    当前的Looper是从`ThreadLocal`中取出的。
    
    `loop()`的核心代码是获取`MessageQueue`，for循环执行`MessageQueue#next()`，获取时可能会阻塞。



## IdleHandler

`IdleHandler`是一个`MessageQueue`类定义的接口。

它的作用是在当前线程的消息队列空闲的时候进行回调，执行一些操作。

```java
/**
 * Callback interface for discovering when a thread is going to block
 * waiting for more messages.
 */
public static interface IdleHandler {
    /**
     * Called when the message queue has run out of messages and will now
     * wait for more.  Return true to keep your idle handler active, false
     * to have it removed.  This may be called if there are still messages
     * pending in the queue, but they are all scheduled to be dispatched
     * after the current time.
     */
    boolean queueIdle();
}
```

* `queueIdle()`如果是一次性的工作，返回`false`。执行完毕后接口将被移除。

### 工作原理

在`MessageQueue.next()`方法循环获取下一条`Message`时，如果当前没有需要处理的消息，会回调所有添加的`IdleHandler`。

### 如何使用

```java
Looper.myQeuue().addIdleHandler();
```



## HandlerThread

`Thread`子类。重写的`run`方法内默认实现了`Looper`的初始化，是一个供`Handler`使用的辅助类。

调用`HandlerThread.start()`启动`Looper`。

```java
HandlerThread handlerThread = new HandlerThread("background");
handlerThread.start();
```
调用`HandlerThread.quit()`和`HandlerThread.quitSafely()`停止`Looper`。
```java
// Pending Message会丢失
handlerThread.quit();
// Pending Message不会丢失
handlerThread.quitSafely();
```



## 内存泄露

使用`Handler`和`Runnable`进行异步操作的过程中，如果使用匿名内部类或者非静态内部类实现很容易造成`Activity`的内存泄漏，进而泄露整个视图树。

### Looper
如果没有调用`Looper.quit()`或者`HandlerThread.quit()`，即使`Activity`销毁了，`Looper`仍在内存中。

所以如果一个异步操作持续比较长的时间，`Looper`的`MessageQueue`中会一直持有这个操作对应的`Message`。

### Runnable
`Runnble`对象被`Message.callback`持有。

### Handler
`Handler`对象被`Message.target`持有。

### 泄露
无论是匿名内部类还是非静态内部类都会持有外部`Activity`的引用，导致泄露。