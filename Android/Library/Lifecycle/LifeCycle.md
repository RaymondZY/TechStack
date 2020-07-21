## Lifecycle

### LifecycleOnwer

`Lifecycle`对象的持有者，被`Activity`，`Fragment`实现，内部创建了`LifecycleRegistry`对象来实现这个接口。

`AppCompatActivity`的父类`ComponentActivity`：

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements LifecycleOwner {

    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```

`Fragment`：

```java
public class Fragment implements LifecycleOwner {
    LifecycleRegistry mLifecycleRegistry;

    public Fragment() {
        initLifecycle();
    }

    private void initLifecycle() {
        mLifecycleRegistry = new LifecycleRegistry(this);
    }

    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```

并且会在生命周期回调的时候通过`LifecycleRegistry`进行分发。

### LifecycleRegistry

`LifecycleRegistry`继承自抽象类`Lifecycle`。

完成了`LifecycleObserver`的注册管理和事件的分发。

```java
public class LifecycleRegistry extends Lifecycle {
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
        new FastSafeIterableMap<>();

    private State mState;

    public void addObserver(@NonNull LifecycleObserver observer) {
    }

    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }
}
```

### LifecycleObserver

生命周期事件监听接口。

默认是空的，需要使用它的子类`FullLifecycleObserver`（`DefaultLifecycleObserver`）和`LifecycleEventObserver`。

```java
public interface LifecycleObserver {
}
```

```java
interface FullLifecycleObserver extends LifecycleObserver {

    void onCreate(LifecycleOwner owner);

    void onStart(LifecycleOwner owner);

    void onResume(LifecycleOwner owner);

    void onPause(LifecycleOwner owner);

    void onStop(LifecycleOwner owner);

    void onDestroy(LifecycleOwner owner);
}

public interface DefaultLifecycleObserver extends FullLifecycleObserver {

    @Override
    default void onCreate(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onStart(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onResume(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onPause(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onStop(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onDestroy(@NonNull LifecycleOwner owner) {
    }
}
```

```java
public interface LifecycleEventObserver extends LifecycleObserver {
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

### 兼容性

对于老版本的`Activity`，使用一个没有界面的`Fragment`插入，来实现生命周期的回调。



## LiveData

包装数据源，配合`Lifecycle`使用，提供生命周期监听。当生命周期处于不活跃的状态时，不进行回调。

### MutableLiveData

继承自`LiveData`，提供了同步`setValue()`接口，异步`postValue()`接口，改变数据的值。

```java
public class MutableLiveData<T> extends LiveData<T> {

    public MutableLiveData(T value) {
        super(value);
    }

    public MutableLiveData() {
        super();
    }

    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

### observe()

`LiveData.observe()`方法传入`LifecycleOwner`和`Observer`接口，用于监听值的变化。

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```

方法向`LifecycleOwner`持有的`Lifecycle`注册了`LifecycleObserver`，在生命周期发生变化时进行记录，让`LiveData`的`Observer`失效。



## ViewModel

`ViewModel`主要解决`Activity`旋转后，数据保存和恢复的问题，它可以在屏幕旋转的情况下保持存活。

### ViewModelProvider

`ViewModel`需要通过`ViewModelProvider`创建。

```java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
    this(owner.getViewModelStore(), factory);
}
```

`ViewModelProvider`构造函数中需要传入`ViewModelStoreOwner`，`Activity`实现了这个接口，提供`ViewModelStore`进行`ViewModel`的存储和管理。

### ViewModelProvider.Factory

`ViewModelProvider.Factory`需要根据传入的`Key`和`Class`创建对应的`ViewModel`对象。

```java
public interface Factory {

    @NonNull
    <T extends ViewModel> T create(@NonNull Class<T> modelClass);
}
```

通常，`ViewModel`可以配合`LiveData`一起使用，在内部存储`LiveData`向View层提供数据。它隔离了View层和Model层的逻辑，让View可以更专注于对数据的处理而不用担心生命周期和保存恢复等问题。