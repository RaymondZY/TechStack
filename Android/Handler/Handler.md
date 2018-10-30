# Handler
在一个线程建立一个消息队列，以队列的方式处理操作。  

## 创建Handler
Handler的构造函数需要传入`Looper`参数，默认构造函数使用当前线程的Looper。
```java
Handler handler = new Handler(handlerThread.getLooper());
```
如果当前线程没有调用过`Looper.prepare()`，那么使用时将会抛出异常。

## 使用Handler
使用Handler主要分为两种操作：
* send操作  
    把一个`Message`添加到队列尾部，消息队里处理到消息时回调`Handler#handleMessage()`方法。
* post操作  
    用`Runnable`组装一个的`Message`添加到队列尾部，消息队列处理到消息时，直接执行`Runnable`不回调`Handler#handleMessage()`方法

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // post相关的操作
        handleCallback(msg);
    } else {
        // 这种逻辑我们涉及不到
        // 为mCallback赋值的构造函数被标为了@hide
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

# Message
包装了一条消息。
* 推荐使用`Message#obtain()`方法获取一个message对象。
* `what`用来区分Message的种类。
* `data`用来存储复杂的数据。调用`setData()`设置。

# MessageQueue
消息队列类。
* `enqueueMessage()`方法添加一条消息。
* `next()`方法获取下一条消息。  

# Looper
在当前运行的线程启动一个消息队列`MessageQueue`，并开始循环处理队列消息。  
整个过程分为两个步骤：
1. `Looper#prepare()`
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

2. `Looper#loop()`
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
    `Loop()`的核心代码是获取`MessageQueue`，for循环执行`MessageQueue#next()`，获取时可能会阻塞。

# HandlerThread
`Thread`子类。重写的`run`方法内默认实现了`Looper`的初始化，是一个供Handler使用的辅助类。   
调用`HandlerThread#start()`启动`Looper`。
```java
HandlerThread handlerThread = new HandlerThread("background");
handlerThread.start();
```
调用`HandlerThread#quit()`和`HandlerThread#quitSafely()`停止`Looper`。
```java
// Pending Message会丢失
handlerThread.quit();
// Pending Message不会丢失
handlerThread.quitSafely();
```



