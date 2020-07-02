# Handler
`Handler`是系统提供的异步消息队列。

## 使用

### 创建Handler
`Handler`的构造函数需要传入`Looper`参数，默认构造函数使用当前线程的Looper。

```java
Handler handler = new Handler(handlerThread.getLooper());

class Handler {
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
}
```
如果当前线程没有调用过`Looper.prepare()`，那么使用时将会抛出异常。

然后`Handler`将持有`Looper`中的`MessageQueue`对象，用于向消息队列中发送消息。

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

推荐使用`Handler.obtainMessage()`或者`Message.obtain()`方法获取一个`message`对象。它在内部维护了一个对象池，减少对象的创建。

`Handler.obtainMessage()`将自动将`Handler`绑定到`Message.target`属性上进行回调，如果使用`Message.obtain()`需要自己设置`target`。

### 参数

* `what`用来区分`Message`的种类。
* `data`用来存储复杂的数据。调用`setData()`设置。
* `target`用来存储最终执行的`Handler`。
* `callback`用来存储`Handler.post()`请求的`Runnable`。
* `next`下一条`Message`的指针用来维护消息队列列表。
* `replyTo`同来存储`Messenger`机制回调的`Messenger`对象。

### 异步消息

通过`Messenger.setAsynchronous()`方法可以将消息指定为异步消息，具体情况将在异步消息章节详细说明。



## MessageQueue

消息队列类。

### 消息入队

```java
boolean enqueueMessage(Message msg, long when) {
    // msg.target不能为空，否则没有Handler进行回调处理
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    // 不能重复入队
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    // 同步
    synchronized (this) {
        // 如果已经调用了quit，无法再入队
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
		// 标记正在使用
        msg.markInUse();
        // 记录希望执行的时间，这个值是从Handler.sendMessageAtTime()中的参数中传递过来的
        // 如果这个值是0表示放到队列头部
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // p == null 的情况表示队列是空的
            // when == 0 的情况表示希望放到队列头部
            // when < p.when 的情况表示当前的Message比队列里的Message希望的时间都更早
            // 那么本来在next()过程中计算的阻塞时间就失效了
            // 需要以这条消息的时间为准重新计算
            // 所以要打破之前的阻塞时间
            msg.next = p;
            mMessages = msg;
            // 如果当前Block的状态就唤醒整个队列
            needWake = mBlocked;
        } else {
            // 找到message.when时间合适的位置插入到队列中
            // 只有一种情况我们需要唤醒整个队列：
            // mBlocked 当前队列正在阻塞
            // p.target == null 队列头部是一个同步栅栏，拦截了所有的同步消息
            // msg.isAsynchronous() 并且当前消息是最早的一条异步消息（意味着接下来的遍历中不会碰到一条异步消息）
            // 执行到这里说明当前的消息不是when时间最早的
            // 那么只有一种情况下会导致之前next()过程计算的阻塞时间失效
            // 就是队列中全部都是同步消息，被头部的一个同步栅栏阻塞
            // 并且当前这条消息就是那唯一的一条异步消息
            // 这种情况就和空队列是类似的
            // 所以要打破之前的阻塞时间
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            // 循环遍历找到一个when时间合适的位置
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    // 找到一个合适的位置停止循环
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    // 如果碰到了一条异步消息，那么说明这条异步消息的when比当前的消息要更早
                    needWake = false;
                }
            }
            // 操作链表插入到这个位置
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        if (needWake) {
            // 如果需要唤醒消息队列进行native调用
            nativeWake(mPtr);
        }
    }
    return true;
}
```



### 获取下一条

* `next()`方法获取下一条消息。  

  内部会进行一个死循环，直到获取到下一条执行的`Message`，虽然是死循环但是它会调用`nativePollOnce()`进行阻塞。

  另外，会在每次空闲的时候进行`IdleHandler`的处理。

  ```java
  Message next() {
      // 如果已经quit过了，不能再进行消息队列循环了
      final long ptr = mPtr;
      if (ptr == 0) {
          return null;
      }
  
      // 这个变量记录的是IdleHandler的数量
      // 这个值一开始初始化为-1用于区分是不是第一次循环
      int pendingIdleHandlerCount = -1;
      
      // timeout一开始设置为0
      // 调用nativePollOnce()时timeout设置为0表示没有等待立刻执行
      int nextPollTimeoutMillis = 0;
      
      // 从这里开始进行循环 直到获取到Message为止
      for (;;) {
          if (nextPollTimeoutMillis != 0) {
              Binder.flushPendingCommands();
          }
  
          // 调用native方法
          // timeout表示等待时长
          nativePollOnce(ptr, nextPollTimeoutMillis);
  
          // 同步
          synchronized (this) {
              final long now = SystemClock.uptimeMillis();
              Message prevMsg = null;
              // 获取第一条Message
              Message msg = mMessages;
              // 如果第一条是一个同步栅栏，那么我们不能让接下来的同步消息通过，只能往后遍历寻找一个异步的消息
              if (msg != null && msg.target == null) {
                  do {
                      prevMsg = msg;
                      msg = msg.next;
                  } while (msg != null && !msg.isAsynchronous());
              }
              // 找到消息了吗
              if (msg != null) {
                  // 找到了消息了
                  // 这条消息设置的时间到了吗
                  if (now < msg.when) {
                      // 如果时间还没有到设置timeout让下次nativePollOnce()时进行这么长时间的等待
                      nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                  } else {
                      // 设置的时间到了这条消息可以用
                      // 那么把消息队列的阻塞取消，让它继续循环下去
                      mBlocked = false;
                      // 操作队列指针从队列中移除
                      if (prevMsg != null) {
                          prevMsg.next = msg.next;
                      } else {
                          mMessages = msg.next;
                      }
                      // 移除掉next指针
                      msg.next = null;
                      // 标记正在使用
                      msg.markInUse();
                      // 返回
                      return msg;
                  }
              } else {
                  // 没有找到消息
                  // 那么只能把timeout设置成-1，让下次nativePollOnce()时无限等待
                  nextPollTimeoutMillis = -1;
              }
  
              // 如果调用过quit，处于收尾阶段，没找到马上执行的消息的话就直接停止等待了
              if (mQuitting) {
                  dispose();
                  return null;
              }
  
              // 如果没有找到消息
              // 或者找到的消息还没有到预期的时间
              // 并且pendingIdleHandlerCount == -1
              // 表示是for循环中第一次处于这种空闲的状态
              // 那我们就准备回调IdleHandler的回调了
              if (pendingIdleHandlerCount < 0
                      && (mMessages == null || now < mMessages.when)) {
                  pendingIdleHandlerCount = mIdleHandlers.size();
              }
              if (pendingIdleHandlerCount <= 0) {
                  // No idle handlers to run.  Loop and wait some more.
                  // 如果没有在等待的IdleHandler
                  // 将队列重置为阻塞状态
                  mBlocked = true;
                  continue;
              }
  
              if (mPendingIdleHandlers == null) {
                  mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
              }
              // 把PendingIdleHandler的ArrayList放到数组中
              // 接下来要退出同步代码块了
              // 使用这个数组避免多线程修改ArrayList
              mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
          }
  
          // 开始回调IdleHandler
          // 这段循环在for循环中只有第一次会运行到
          // 因为接下来我们会把pendingIdleHandlerCount设置为0
          // 根据上面代码它会一直保持为0
          // 这段循环不会再被执行到了
          for (int i = 0; i < pendingIdleHandlerCount; i++) {
              final IdleHandler idler = mPendingIdleHandlers[i];
              mPendingIdleHandlers[i] = null; // release the reference to the handler
  
              boolean keep = false;
              try {
                  // 回调IdleHandler.queueIdle()接口
                  // 判断是否需要继续保留它
                  keep = idler.queueIdle();
              } catch (Throwable t) {
                  Log.wtf(TAG, "IdleHandler threw exception", t);
              }
  
              if (!keep) {
                  // 不用保留就移除了
                  // 进行同步保护ArrayList安全
                  synchronized (this) {
                      mIdleHandlers.remove(idler);
                  }
              }
          }
  
          // 把pendingIdleHandlerCount设置为0
          // 保证IdleHandler回调只会被执行一次
          pendingIdleHandlerCount = 0;
  
          // 执行到这里了说明我们回调过IdleHandler
          // 回调IdleHandler是需要花费时间的
          // 那么之前设置的等待时间可能已经没用了
          // nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
          // 在执行回调的这段时间可能有新的消息已经插入到队列中了
          // 可能需要马上执行
          // 所以把timeout设置为0让下一次pollOnce()马上执行
          nextPollTimeoutMillis = 0;
      }
  }
  ```

### 异步Message

可以通过`Message.setAsynchronous()`将消息设置为异步。如果消息队列中有同步栅栏，那么只有它后面的异步消息能被`next()`方法取到。

### 同步栅栏

添加同步栅栏可以用`Looper.myQueue().postSyncBarrier()`方法。这个方法会返回一个token，用于调用`Looper.myQueue().removeSyncBarrier()`时做区分。

```java
private int postSyncBarrier(long when) {
    // 同步
    synchronized (this) {
        // token更新
        final int token = mNextBarrierToken++;
        // 创建一个新的消息
        // 这个消息没有Target 用于和其它消息区分
        // 并把token保存在arg1上用于之后的删除进行比较
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            // 在消息队列中找到一个when值合适的位置
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 插入同步栅栏消息
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        // 返回token用于之后的removeSyncBarrier()
        return token;
    }
}

public void removeSyncBarrier(int token) {
    // 同步
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        // 找到token标记的同步栅栏
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                                            + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        // 检查是否需要唤醒消息队列
        if (prev != null) {
            // 如果同步栅栏有prev节点，
            // 它如果也是一个同步栅栏，那么移除掉当前的栅栏也不会改变队列被阻塞的性质，不需要wake
            // 如果它是一个正常的消息，那么移除掉当前的栅栏也不会改变队列的性质
            prev.next = p.next;
            needWake = false;
        } else {
            // 如果同步栅栏没有prev节点，那么它在队列头部
            // 如果下一个节点是一个正常的消息（有target），那么要进行wake
            // 如果下一个节点是空的，也进行wake（没有特别理解，可能是为了再触发一次IdleHandler？）
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();

        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```



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



## Looper

在当前运行的线程启动一个消息队列`MessageQueue`，并开始循环处理队列消息 。

整个过程分为两个步骤：

### prepare()

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

### Loop

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
        
        // 回调Handler执行
        msg.target.dispatchMessage(msg);
    }
}
```
当前的Looper是从`ThreadLocal`中取出的。

`loop()`的核心代码是获取`MessageQueue`，for循环执行`MessageQueue.next()`，获取时可能会阻塞。



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