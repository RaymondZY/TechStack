# SharedPreferences

## 类结构

`SharedPreferencesImpl`实现了`SharedPreferences`接口。

`SharedPreferencesImpl.EditorImpl`实现了`SharedPreferences.Editor`接口。



## API

* `Context.getSharedPreferences()`：获取`SharedPrefernces`对象。
* `SharedPreferences.edit()`：获取`Editor`对象
* `SharedPreferences.getXXX()`：获取value值。
* `Editor.putXXX()`：写入value值。



## 初始化

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    
    /* ----- 从文件读取 ----- */
    startLoadFromDisk();
}

private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```

在第一次获取`SharedPreferences`对象时，异步从文件读取数据。



## 线程安全

```java
@Override
public Editor putString(String key, @Nullable String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}
```

`put`操作会加锁保证多线程edit操作的安全。

```java
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        /* ----- 等待读取完毕 ----- */
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```

`get`操作会加锁`mLoaded`保证数据加载完后再执行获取操作。



## 优化

* 不要放过多的内容，避免文件加载过慢。
* 不要多次`edit()`，多次`apply()`。
* 多进程访问不安全，使用`ContentProvider`。