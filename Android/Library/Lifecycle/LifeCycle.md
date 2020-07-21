# Lifecycle

## LifecycleOnwer

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



## LifecycleRegistry

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



## LifecycleObserver

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



## 兼容性

对于老版本的`Activity`，使用一个没有界面的`Fragment`插入，来实现生命周期的回调。