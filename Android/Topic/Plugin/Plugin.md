# Plugin

要实现插件化，有两个基本的问题：一是解决插件类加载的问题，二是解决插件资源加载的问题。

随着技术的发展，根据实现方式，可以分为占位式，Hook式和LoadedApk式。



## 类加载

通过`DexClassLoader`可以实现对APK文件的加载。

```java
mDexClassLoader = new DexClassLoader(
    pluginApk.getAbsolutePath(),
    context.getCacheDir().getAbsolutePath(),
    // Library路径，一般为null
    null,
    context.getClassLoader()
);

public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
    super((String)null, (File)null, (String)null, (ClassLoader)null);
    throw new RuntimeException("Stub!");
}
```

* `dexPath`：APK文件的路径。
* `optimizedDirectory`：进行优化操作的文件夹。
* `librarySearchPath`：`.so`库的搜索路径，一般为`null`。
* `parent`：上一级`ClassLoader`，一般传递当前的`ClassLoader`。



## 资源加载

通过`AssetManager`和`Resources`类进行加载。

```java
AssetManager assetManager = AssetManager.class.newInstance();
Method addAssetPathMethod = AssetManager.class.getMethod("addAssetPath", String.class);
addAssetPathMethod.invoke(assetManager, pluginApk.getAbsolutePath());

mResources = new Resources(assetManager, context.getResources().getDisplayMetrics(), context.getResources().getConfiguration());

public Resources(AssetManager assets, DisplayMetrics metrics, Configuration config) {
    this(null);
    mResourcesImpl = new ResourcesImpl(assets, metrics, config, new DisplayAdjustments());
}
```

* `assets`：包含插件APK的`AssetManager`，需要通过反射构造。
* `metrics`：`DisplatMetrics`，一般使用宿主`Resources`中的值。
* `config`：`Configuration`，一般使用宿主`Configuration`中的值。



## 插件Activity使用类和资源

通过重写`Context.getClassLoader()`与`ContextThemeWrapper.getResources()`方法可以让插件`Activity`使用加载的内容。

```java
public class PluginHostActivity extends AppCompatActivity {
    @Override
    public ClassLoader getClassLoader() {
        return PluginLoader.getInstance().getDexClassLoader();
    }

    @Override
    public Resources getResources() {
        return PluginLoader.getInstance().getResources();
    }
}
```



## 占位式

### 简述

利用一个占位的`Activity`来代理插件`Activity`的所有方法。

### 优点

它的侵入性小，不需要Hook或者反射破坏Framework层。

### 缺点

插件`Activity`没有`Context`环境，许多方法必须使用宿主`Context`进行开发。



## Hook式

### 简述

利用反射或者动态代理的方式Hook系统源码，让插件`Activity`拥有运行`Context`，像宿主中的`Activity`一样正常启动。

### 优点

插件`Activity`能像宿主`Activity`一样运行，解决了`Context`的问题。

### 缺点

系统侵入性强。Hook系统源码会随着系统版本升级源码结构变化而失效，必须要重新进行适配。并且随着版本升高，越来越多的系统API被放到了隐藏API黑名单被禁止反射调用，失效的可能性越来越高。

另一个缺点就是，插件Dex需要添加到宿主的`PathList`中，随着插件增多，宿主的会合并越来越多的插件Dex，带来性能问题。

### Hook点

* `Instrumentation.execStartActivity()`：利用动态代理，代理`IActivityTaskService`，将`Intent`替换成占位的`Activity`，绕过`AMS`的`AndroidManifest.xml`注册检测。
* `ActivityThread.mH`：利用反射，添加自定义的`Handler.Callback`到`ActivityThread.H`，在处理`handleLaunchActivity`的`message`时，将`Intent`还原。



## LoadedApk

### 简述

在Hook方式上的升级版本，主要为了解决宿主`PathList`合并插件Dex的问题。通过创建插件自己的`LoadedApk`，并在启动`Activity`时进行宿主和插件的切换，达到不需要合并`PathList`的目的。

### 优点

相较于Hook方式，在插件数量较大的情况下，性能更好。

### 缺点

同Hook方式一样，对系统侵入性强。

