# Okhttp

## 重要的类

* `OkhttpClient`：SDK入口，一般创建为单例，可以用于创建网络请求的`Call`对象。
* `Call`：网络请求整体的封装，包含了`Request`和`Response`，提供了同步请求接口`execute()`，异步请求接口`enqueue()`，请求取消接口`cancel()`等。
  * `RealCall`：`Call`的具体实现类。
* `Request`：请求部分的封装。
* `Response`：结果部分的封装。
* `Dispatcher`：请求执行调度器，用于调度请求的等待、执行、取消等，也实现了请求的同步和异步操作。
* `Callback`：异步请求回调。
* `RealInterceptorChain`：请求拦截器责任链。



## 请求流程

重点关注异步请求流程，同步过程相对简单。

### 创建请求

创建`OkHttpClient`对象。

```java
OkHttpClient client = new OkHttpClient.Builder().build();
```

创建`Request`对象。

```java
Request request = new Request.Builder().url("https://www.baidu.com").build();
```

创建`Call`对象。

```java
Call call = client.newCall(request);
```

调用`Call.enqueue()`进行异步请求。

```java
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        Log.d("DEBUG", "MainActivity.onFailure()");
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        Log.d("DEBUG", "MainActivity.onResponse()");
        Log.d("DEBUG", "MainActivity.onResponse() response = " + response.toString());
    }
});
```

### 请求入队

调用`RealCall.enqueue()`。

```java
class RealCall {
    @Override public void enqueue(Callback responseCallback) {
        synchronized (this) {
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
        captureCallStackTrace();
        eventListener.callStart(this);
        client.dispatcher().enqueue(new AsyncCall(responseCallback));
    }
}
```

* 检查是否重复请求。

调用`Dispatcher.enqueue()`。

```java
class Dispatcher {
    void enqueue(AsyncCall call) {
        synchronized (this) {
            readyAsyncCalls.add(call);

            // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
            // the same host.
            if (!call.get().forWebSocket) {
                AsyncCall existingCall = findExistingCallWithHost(call.host());
                if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
            }
        }
        promoteAndExecute();
    }
}
```

* 将请求放到等待队列，并计算请求同一域名的并发数。
* 从等待队列中晋升一批请求进行执行。

```java
class Dispatcher {
    private boolean promoteAndExecute() {
        assert (!Thread.holdsLock(this));

        List<AsyncCall> executableCalls = new ArrayList<>();
        boolean isRunning;
        synchronized (this) {
            for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
                AsyncCall asyncCall = i.next();

                /* ----- 如果当前执行的请求数量达到最大请求数（64），放弃这一次的晋升 ----- */
                if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.

                /* ----- 如果这个请求的域名并发数量达到了最大并发数（5），跳过这个域名的请求 ----- */
                if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

                /* ----- 从等待队列中移除 ----- */
                i.remove();

                /* ----- 同域名并发数增加 ----- */
                asyncCall.callsPerHost().incrementAndGet();

                /* ----- 放入执行队列 ----- */
                executableCalls.add(asyncCall);
                runningAsyncCalls.add(asyncCall);
            }
            isRunning = runningCallsCount() > 0;
        }

        for (int i = 0, size = executableCalls.size(); i < size; i++) {
            AsyncCall asyncCall = executableCalls.get(i);
            /* ----- 放入线程池执行 ----- */
            asyncCall.executeOn(executorService());
        }

        return isRunning;
    }
}
```

在`Dispatcher`内部构造了这个线程池。这个线程池是一个缓存线程池。

```java
class Dispatcher {
    public synchronized ExecutorService executorService() {
        if (executorService == null) {
            executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
                                                     new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
        }
        return executorService;
    }   
}
```

### 异步执行

调用`RealCall.executeOn()`。

```java
class RealCall {
    void executeOn(ExecutorService executorService) {
        assert (!Thread.holdsLock(client.dispatcher()));
        boolean success = false;
        try {
            /* ----- 在线程池中执行 ----- */
            executorService.execute(this);
            success = true;
        } catch (RejectedExecutionException e) {
            InterruptedIOException ioException = new InterruptedIOException("executor rejected");
            ioException.initCause(e);
            transmitter.noMoreExchanges(ioException);
            responseCallback.onFailure(RealCall.this, ioException);
        } finally {
            if (!success) {
                client.dispatcher().finished(this); // This call is no longer running!
            }
        }
    }
}
```

线程池回调给`RealCall.execute()`。

```java
class RealCall {
    @Override protected void execute() {
        boolean signalledCallback = false;
        transmitter.timeoutEnter();
        try {
            /* ----- 通过责任链拦截器获取请求结果 ----- */
            /* ----- 在下一小节重点分析 ----- */
            Response response = getResponseWithInterceptorChain();
            signalledCallback = true;
            
            /* ----- 成功回调 ----- */
            responseCallback.onResponse(RealCall.this, response);
        } catch (IOException e) {
            if (signalledCallback) {
                // Do not signal the callback twice!
                Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
            } else {
                /* ----- 失败回调 ----- */
                responseCallback.onFailure(RealCall.this, e);
            }
        } catch (Throwable t) {
            cancel();
            if (!signalledCallback) {
                IOException canceledException = new IOException("canceled due to " + t);
                canceledException.addSuppressed(t);
                /* ----- 失败回调 ----- */
                responseCallback.onFailure(RealCall.this, canceledException);
            }
            throw t;
        } finally {
            /* ----- 回调Dispatcher结束此次请求，并组织下一次请求 ----- */
            client.dispatcher().finished(this);
        }
    }
}
```

回调`Dispatcher.finished()`。

```java
class Dispatcher {
    void finished(AsyncCall call) {
        call.callsPerHost().decrementAndGet();
        finished(runningAsyncCalls, call);
    }

    private <T> void finished(Deque<T> calls, T call) {
        Runnable idleCallback;
        synchronized (this) {
            /* ----- 移除此次的请求 ----- */
            if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
            idleCallback = this.idleCallback;
        }

        /* ----- 组织下一次的请求 ----- */
        boolean isRunning = promoteAndExecute();

        if (!isRunning && idleCallback != null) {
            idleCallback.run();
        }
    }
}
```



## 拦截器责任链

上一节中获取网络请求结果的最终方法是`RealCall.getResponseWithInterceptorChain();`

```java
Response getResponseWithInterceptorChain() throws IOException {
    /* ----- 构造拦截器责任链 ----- */
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
                                                       originalRequest, this, client.connectTimeoutMillis(),
                                                       client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
        /* ----- 通过责任链调用Child.proceed()获取结果 ----- */
        Response response = chain.proceed(originalRequest);
        if (transmitter.isCanceled()) {
            closeQuietly(response);
            throw new IOException("Canceled");
        }
        return response;
    } catch (IOException e) {
        calledNoMoreExchanges = true;
        throw transmitter.noMoreExchanges(e);
    } finally {
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null);
        }
    }
}
```

* 拦截器责任链依次包括

  * 用户自定义`Interceptor`。

  * `RetryAndFollowUpInterceptor`，重试及重定向拦截器。

  * `BridgeInterceptor`，桥接拦截器。

    > 桥接应用代码和网络代码，通俗地讲就是把用户友好的请求对象转换成底层网路请求对象。

  * `CacheInterceptor`，缓存拦截器。

  * `ConnectInterceptor`，网络连接拦截器。

  * `CallServerInterceptor`，请求服务拦截器。

每一个拦截器的责任各不相同，处理流程层层递进。一个拦截器可以自己决定采取哪种行动：一种是调用下一级拦截器，并把输入和输出进行一定的包装；另一种是进行拦截，直接返回结果。

> 例如，桥接拦截器就属于第一种，它不具备返回结果的能力，仅对输入和输出进行一定的转换。
>
> 又例如，缓存拦截器命中时就属于第二种，它拦截了真实的网络请求过程，直接返回了结果。

用户可以自定义拦截器来实现一些定制功能，例如：添加Log，Url签名，Body加密等。



## 缓存策略



## 连接池



## 重连



## Gzip