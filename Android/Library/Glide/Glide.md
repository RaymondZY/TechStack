# Glide

对Glide源码4.11.0版本进行分析。

```gradle
implementation 'com.github.bumptech.glide:glide:4.11.0'
```

## 类的作用

* `Glide`：SDK主入口。
* `RequestManagerRetriever`：用于创建`RequestManager`。
* `RequestManager`：管理请求的新建、暂停、取消操作，并向`Context`注册生命周期监听。
* `LifecycleListener`：生命周期监听接口，由`RequestManager`实现。
* `RequestBuilder`：由`RequestManager`构建生成。用于构造`Request` 和`Target`。
* `Request`：请求接口，封装了请求的参数。持有`Engine`对象进行请求的执行。
  * `SingleRequest`：普通的加载。
  * `ThumbnailRequestCoodinator`：普通加载与缩略图加载的合集，进行统一管理。

* `Target`：加载目标接口，继承自`LifecycleListener`接口，管理最终的图片使用。
* `Engine`：加载引擎，加载的主逻辑。

## 加载流程

### Glide.with()

`Glide.with()`返回`RequestManager`对象。

可以传入的参数有`FragmentActivity`，`Activity`，`Context`，`Fragment`，`View`。根据传入的参数不同，构造出不同的`RequestManager`。

内部通过`RequestManagerRetriever`类进行`RequestManager`的构造。

```java
class Glide {
    public static RequestManager with(@NonNull FragmentActivity activity) {
        return getRetriever(activity).get(activity);
    }
}

class RequestManagerRetriever {
    public RequestManager get(@NonNull FragmentActivity activity) {
        if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            FragmentManager fm = activity.getSupportFragmentManager();
            return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
        }
    }

    private RequestManager supportFragmentGet(...) {
        SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            /* ----- 1. 生成Glide对象 ----- */
            Glide glide = Glide.get(context);
            /* ----- 2. 构造RequestManager ----- */
            requestManager =
                factory.build(
                glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }
}
```

### RequestManager.load()

`RequestManager.load()`返回的是`RequestBuilder`对象。内部完成了对`RequestBuilder`的构造，以及加载类型的封装。

```java
class RequestManager {
    public RequestBuilder<Drawable> load(@Nullable String string) {
        return asDrawable().load(string);
    }

    public RequestBuilder<Drawable> asDrawable() {
        return as(Drawable.class);
    }

    public <ResourceType> RequestBuilder<ResourceType> as(@NonNull Class<ResourceType> resourceClass) {
        return new RequestBuilder<>(glide, this, resourceClass, context);
    }
}

class RequestBuilder {
    public RequestBuilder<TranscodeType> load(@Nullable String string) {
        return loadGeneric(string);
    }

    private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
        this.model = model;
        isModelSet = true;
        return this;
    }
}
```

### RequestBuilder.into()

`RequestBuilder.into()`方法中会进行`Request`和`Target`的构造，并最终启动`Request`。

```java
class Reqeust Builder {
    public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {

        ... 

            return into(
            /* ----- 构造Target ----- */
            glideContext.buildImageViewTarget(view, transcodeClass),
            /*targetListener=*/ null,
            requestOptions,
            Executors.mainThreadExecutor());
    }

    private <Y extends Target<TranscodeType>> Y into(...) {
        /* ----- 构造Request ----- */
        Request request = buildRequest(target, targetListener, options, callbackExecutor);

        Request previous = target.getRequest();
        if (request.isEquivalentTo(previous)
            && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
            // If the request is completed, beginning again will ensure the result is re-delivered,
            // triggering RequestListeners and Targets. If the request is failed, beginning again will
            // restart the request, giving it another chance to complete. If the request is already
            // running, we can let it continue running without interruption.
            if (!Preconditions.checkNotNull(previous).isRunning()) {
                // Use the previous request rather than the new one to allow for optimizations like skipping
                // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
                // that are done in the individual Request.
                previous.begin();
            }
            return target;
        }

        requestManager.clear(target);
        target.setRequest(request);

        /* ----- 触发request开始 ----- */
        requestManager.track(target, request);

        return target;
    }
}

class RequestManager {
    synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
        targetTracker.track(target);
        requestTracker.runRequest(request);
    }
}

class RequestTracker {
    public void runRequest(@NonNull Request request) {
        requests.add(request);
        if (!isPaused) {
            /* ----- 回调SingleRequest.bingin() ----- */
            request.begin();
        } else {
            request.clear();
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Paused, delaying request");
            }
            pendingRequests.add(request);
        }
    }
}

class SingleRequest {
    public void begin() {
        synchronized (requestLock) {
            ...

            status = Status.WAITING_FOR_SIZE;
            if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
                onSizeReady(overrideWidth, overrideHeight);
            } else {
                target.getSize(this);
            }

            ...
        }
    }
}
```

### ViewTarget.getSize()

`ViewTarget.getSize()`方法会向`ViewTreeObserver`注册`OnPreDrawListener()`以在合适的时机获取到View的大小。然后再回调`SingleRequest.onSizeReady()`，开始`Engine.load()`加载。

```java
class ViewTraget {
    void getSize(@NonNull SizeReadyCallback cb) {
        int currentWidth = getTargetWidth();
        int currentHeight = getTargetHeight();
        if (isViewStateAndSizeValid(currentWidth, currentHeight)) {
            cb.onSizeReady(currentWidth, currentHeight);
            return;
        }

        // We want to notify callbacks in the order they were added and we only expect one or two
        // callbacks to be added a time, so a List is a reasonable choice.
        if (!cbs.contains(cb)) {
            cbs.add(cb);
        }
        if (layoutListener == null) {
            ViewTreeObserver observer = view.getViewTreeObserver();
            layoutListener = new SizeDeterminerLayoutListener(this);
            observer.addOnPreDrawListener(layoutListener);
        }
    }
}

class SingleRequest {
    public void onSizeReady(int width, int height) {
        ...
        loadStatus = engine.load(...);
        ...
    }
}
```



## 缓存机制

加载引擎`Engine`类中定义了图片加载的逻辑。

调用`Engine.load()`方法进行图片加载。

```java
public class Engine {
    public <R> LoadStatus load() {

        /* ----- 1. 生成图片的Key ----- */
        EngineKey key = keyFactory.buildKey(...);

        EngineResource<?> memoryResource;
        synchronized (this) {
            /* ----- 2. 从内存中加载 ----- */
            memoryResource = loadFromMemory(...);

            if (memoryResource == null) {
                /* ----- 3. 等待正在执行的解码任务或者新建一个解码任务 ----- */
                /* ----- 3. 返回加载状态 ----- */
                return waitForExistingOrStartNewJob(...);
            }
        }

        /* ----- 2. 回调加载成功 ----- */
        cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);

        return null;
    }
}
```

* 生成图片的Key
* 从内存中加载
  * 回调加载成功
* 等待正在执行的解码任务或者新建一个解码任务
  * 返回加载状态

调用`Engine.loadFromMemory()`从内存的加载。

```java
public class Engine {
    private EngineResource<?> loadFromMemory(...) {
        /* ----- 1. 判断是否开启了内存缓存 ----- */
        if (!isMemoryCacheable) {
            return null;
        }

        /* ----- 2. 从ActiveResources中获取 ----- */
        EngineResource<?> active = loadFromActiveResources(key);
        if (active != null) {
            if (VERBOSE_IS_LOGGABLE) {
                logWithTimeAndKey("Loaded resource from active resources", startTime, key);
            }
            return active;
        }

        /* ----- 3. 从内存缓存中获取 ----- */
        EngineResource<?> cached = loadFromCache(key);
        if (cached != null) {
            if (VERBOSE_IS_LOGGABLE) {
                logWithTimeAndKey("Loaded resource from cache", startTime, key);
            }
            return cached;
        }

        return null;
    }
}
```

* 判断是否开启了内存缓存
* 从`ActiveResources`中获取
* 从`MemoryCache`中获取

调用`Engine.waitForExistingOrStartNewJob()`从任务中获取。

```java
public class Engine {
    private <R> LoadStatus waitForExistingOrStartNewJob(...) {

        /* ----- 1. 从现有任务中获取 ----- */
        EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
        if (current != null) {
            current.addCallback(cb, callbackExecutor);
            if (VERBOSE_IS_LOGGABLE) {
                logWithTimeAndKey("Added to existing load", startTime, key);
            }
            return new LoadStatus(cb, current);
        }

        
        /* ----- 2. 新建任务获取 ----- */
        EngineJob<R> engineJob = engineJobFactory.build(...);

        DecodeJob<R> decodeJob = decodeJobFactory.build(...);

        jobs.put(key, engineJob);

        engineJob.addCallback(cb, callbackExecutor);
        
        /* ----- 3. 启动异步加载任务 ----- */
        engineJob.start(decodeJob);

        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Started new load", startTime, key);
        }
        return new LoadStatus(cb, engineJob);
    }
}
```

* 从现有任务中获取
* 新建任务获取
* 启动异步加载任务

`EngineJob`在线程池中启动`DecodeJob`任务。`DecodeJob`内部是一个状态机，状态包括：

* `RESOURCE_CACHE`：处理后的图片缓存。
* `DATA_CACHE`：原图缓存。
* `SOURCE`：图片数据源。

`DecodeJob`会依次在这几个状态间切换，尝试获取。

### Key

生成图片Key的类是`EngineKey`。

`EngineKey`内部保存了图片的一系列参数信息。

```java
class EngineKey implements Key {
    private final Object model;
    private final int width;
    private final int height;
    private final Class<?> resourceClass;
    private final Class<?> transcodeClass;
    private final Key signature;
    private final Map<Class<?>, Transformation<?>> transformations;
    private final Options options;
    private int hashCode;
}
```

重写了`Object.equals()`和`Object.hashCode()`。从中我们可以看到，许多参数都会影响Key的生成。其中最明显的就是图片的宽高，如果图片的宽高发生了变化，将重新加载一遍。

```java
@Override
public boolean equals(Object o) {
    if (o instanceof EngineKey) {
        EngineKey other = (EngineKey) o;
        return model.equals(other.model)
            && signature.equals(other.signature)
            && height == other.height
            && width == other.width
            && transformations.equals(other.transformations)
            && resourceClass.equals(other.resourceClass)
            && transcodeClass.equals(other.transcodeClass)
            && options.equals(other.options);
    }
    return false;
}

@Override
public int hashCode() {
    if (hashCode == 0) {
        hashCode = model.hashCode();
        hashCode = 31 * hashCode + signature.hashCode();
        hashCode = 31 * hashCode + width;
        hashCode = 31 * hashCode + height;
        hashCode = 31 * hashCode + transformations.hashCode();
        hashCode = 31 * hashCode + resourceClass.hashCode();
        hashCode = 31 * hashCode + transcodeClass.hashCode();
        hashCode = 31 * hashCode + options.hashCode();
    }
    return hashCode;
}
```

### ActiveResources

内部使用的是一个弱引用`HashMap`来记录正在被使用的图片。

```java
class ActiveResources {
    private final boolean isActiveResourceRetentionAllowed;
    private final Executor monitorClearedResourcesExecutor;
    @VisibleForTesting final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
    private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = new ReferenceQueue<>();
}
```

实现的原理是使用`WeakReference`和`ReferenceQueue`两个类来监听资源被GC回收。一个`WeakReference`被GC回收后，会添加到`ReferenceQueue`。通过调用`ReferenceQueue.remove()`方法可以阻塞等待获取到下一个被GC的弱引用对象。

```java
class ActiveResources {

    ActiveResources(
        boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {
        this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;
        this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;

        /* ----- 异步线程不断监听弱引用的移除 ----- */
        monitorClearedResourcesExecutor.execute(
            new Runnable() {
                @Override
                public void run() {
                    cleanReferenceQueue();
                }
            });
    }

    void cleanReferenceQueue() {
        while (!isShutdown) {
            try {
                /* ----- 同步阻塞等待下一个被GC的弱引用 ----- */
                ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
                cleanupActiveReference(ref);

                // This section for testing only.
                DequeuedResourceCallback current = cb;
                if (current != null) {
                    current.onResourceDequeued();
                }
                // End for testing only.
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
        synchronized (this) {
            activeEngineResources.remove(ref.key);

            if (!ref.isCacheable || ref.resource == null) {
                return;
            }
        }

        EngineResource<?> newResource =
            new EngineResource<>(
            ref.resource, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ false, ref.key, listener);
        /* ----- 回调接口 ----- */
        listener.onResourceReleased(ref.key, newResource);
    }
}
```

`ActiveResources`提供了`activate()`和`deactivate()`方法来激活和失效缓存，内部实现就是弱引用`HashMap`的维护。

```java
class ActiveResources {
    synchronized void activate(Key key, EngineResource<?> resource) {
        ResourceWeakReference toPut =
            new ResourceWeakReference(
            key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);

        ResourceWeakReference removed = activeEngineResources.put(key, toPut);
        if (removed != null) {
            removed.reset();
        }
    }

    synchronized void deactivate(Key key) {
        ResourceWeakReference removed = activeEngineResources.remove(key);
        if (removed != null) {
            removed.reset();
        }
    }
}
```

`listener.onResourceReleased()`接口会回调到`Engine.onResourceReleased()`，并调用`ActiveResources.deactivate()`移除。

```java
class Engine {
    @Override
    public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
        /* ----- 从ActiveResources中移除 ----- */
        activeResources.deactivate(cacheKey);
        if (resource.isMemoryCacheable()) {
            /* ----- 放入MemoryCache ----- */
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource, /*forceNextFrame=*/ false);
        }
    }
}
```

`Engine.onEngineJobComplete()`加载完毕时，会调用`ActiveResources.activate()`添加。

```java
class Engine {
    @Override
    public synchronized void onEngineJobComplete(
        EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
        // A null resource indicates that the load failed, usually due to an exception.
        if (resource != null && resource.isMemoryCacheable()) {
            /* ----- 添加到ActiveResources ----- */
            activeResources.activate(key, resource);
        }

        jobs.removeIfCurrent(key, engineJob);
    }
}
```

### MemoryCache

用来缓存最近使用过的图片。

内部实现使用的是`LRUCache`，`LRUCache`内部实现使用的是`LinkedHashMap`。

### ResourcesCache

处理后的图片缓存。

```java
class ResourcesCacheGenerator {
    public boolean startNext() {
        ...
        while (modelLoaders == null || !hasNextModelLoader()) {
            ...
            currentKey = new ResourceCacheKey(...);
            /* ----- 1. 从DiskCache中尝试获取 ----- */
            cacheFile = helper.getDiskCache().get(currentKey);
            if (cacheFile != null) {
                sourceKey = sourceId;
                /* ----- 2. 如果找到了缓存获取文件对应的ModelLoader ----- */
                modelLoaders = helper.getModelLoaders(cacheFile);
                modelLoaderIndex = 0;
            }
        }

        loadData = null;
        boolean started = false;
        while (!started && hasNextModelLoader()) {
            ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
            /* ----- 3. 使用ModelLoader创建LoadData ----- */
            loadData =
                modelLoader.buildLoadData(
                cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
            if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
                started = true;
                /* ----- 4. 使用LoadData中的DataFetcher获取数据 ----- */
                loadData.fetcher.loadData(helper.getPriority(), this);
            }
        }

        return started;
    }
}
```

* 从`DiskCache`中尝试获取。
* 如果找到了缓存获取文件对应的`ModelLoader`。
* 使用`ModelLoader`创建`LoadData`。
* 使用`LoadData`中的`DataFetcher`获取数据。

### DataCache

原图图片缓存。流程与`ResourcesCache`类似。

```java
class DataCacheGenerator {
    public boolean startNext() {
        while (modelLoaders == null || !hasNextModelLoader()) {
            ...
            Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
            /* ----- 1. 从DiskCache中尝试获取 ----- */
            cacheFile = helper.getDiskCache().get(originalKey);
            if (cacheFile != null) {
                this.sourceKey = sourceId;
                /* ----- 2. 如果找到了缓存获取文件对应的ModelLoader ----- */
                modelLoaders = helper.getModelLoaders(cacheFile);
                modelLoaderIndex = 0;
            }
        }

        loadData = null;
        boolean started = false;
        while (!started && hasNextModelLoader()) {
            ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
            /* ----- 3. 使用ModelLoader创建LoadData ----- */
            loadData =
                modelLoader.buildLoadData(
                cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
            if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
                started = true;
                /* ----- 4. 使用LoadData中的DataFetcher获取数据 ----- */
                loadData.fetcher.loadData(helper.getPriority(), this);
            }
        }
        return started;
    }
}
```

* 从`DiskCache`中尝试获取。
* 如果找到了缓存获取文件对应的`ModelLoader`。
* 使用`ModelLoader`创建`LoadData`。
* 使用`LoadData`中的`DataFetcher`获取数据。

### Source

图片原始数据源。

```java
class SourceGenerator {
    public boolean startNext() {
        if (dataToCache != null) {
            Object data = dataToCache;
            dataToCache = null;
            cacheData(data);
        }

        if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
            return true;
        }
        sourceCacheGenerator = null;

        loadData = null;
        boolean started = false;
        while (!started && hasNextModelLoader()) {
            loadData = helper.getLoadData().get(loadDataListIndex++);
            if (loadData != null
                && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
                    || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
                started = true;
                startNextLoad(loadData);
            }
        }
        return started;
    }
}
```



## 生命周期管理

在调用`Glide.with()`方法，`RequestManagerRetriever`创建`RequestManager`时，会对当前`with`的对象进行判断。如果是`FragmentActivity`，`Activity`，`Fragment`，会对`RequestManager`增加生命周期的管理。

具体实现的方式是在`FragmentManager`中添加一个没有界面的`Fragment`，向它的`Lifecycle`注册监听接口`LifecycleListener`。`RequestManager`实现了`LifecycleListener`，并在回调时管理任务的暂停、恢复、终止。

### RequestManagerRetriever.get()

```java
class RequestManagerRetriever {
    public RequestManager get(@NonNull FragmentActivity activity) {
        if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            FragmentManager fm = activity.getSupportFragmentManager();
            return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
        }
    }

    private RequestManager supportFragmentGet(...) {
        /* ----- 1. 构造没有界面的Fragment ----- */
        SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            Glide glide = Glide.get(context);
            /* ----- 2. 构造RequestManager并绑定回调接口 ----- */
            requestManager =
                factory.build(
                glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }
}

class RequestManager implements LifecycleListener {
    RequestManager(...) {
        if (Util.isOnBackgroundThread()) {
            mainHandler.post(addSelfToLifecycle);
        } else {
            lifecycle.addListener(this);
        }
    }

    public synchronized void onStop() {
        pauseRequests();
        targetTracker.onStop();
    }


    public synchronized void onStart() {
        resumeRequests();
        targetTracker.onStart();
    }

    public synchronized void onDestroy() {
        targetTracker.onDestroy();
        for (Target<?> target : targetTracker.getAll()) {
            clear(target);
        }
        targetTracker.clear();
        requestTracker.clearRequests();
        lifecycle.removeListener(this);
        lifecycle.removeListener(connectivityMonitor);
        mainHandler.removeCallbacks(addSelfToLifecycle);
        glide.unregisterRequestManager(this);
    }
}
```

