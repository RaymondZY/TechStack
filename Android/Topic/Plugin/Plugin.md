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



## 占位

### 简述

在`AndroidManifest`中，利用一个占位的`Activity`来实际进行启动和接收声明周期回调，并转发给插件`Activity`。

### 优点

它的侵入性小，不需要Hook或者反射破坏Framework层。

### 缺点

插件`Activity`没有`Context`环境，许多方法必须使用宿主`Context`进行开发。



## Hook

### 简述

利用反射或者动态代理的方式Hook系统源码，让插件`Activity`拥有运行`Context`，像宿主中的`Activity`一样正常启动。

### Hook点

* `Instrumentation.execStartActivity()`：利用动态代理，代理`IActivityTaskService`，将`Intent`替换成占位的`Activity`，绕过`AMS`的`AndroidManifest.xml`注册检测。
* 另一种方式是创建一个自定义的`Instrumentation`，在`execStartActivity`时，进行`Intent`的替换，在`newActvitiy`时，进行`Intent`的还原。再反射修改`ActivityThread`中的`Instrumentation`。
* `ActivityThread.mH`：利用反射，添加自定义的`Handler.Callback`到`ActivityThread.H`，在处理`handleLaunchActivity`的`message`时，将`Intent`还原。



## VirtualAPK

滴滴出品。

### Activity

通过在`AndroidManifest`中添加占坑的Activity，并且在`startActivity()`方法时进行Hook偷梁换柱。

Hook点是`Instrumentation.execStartActivity()`和`Instrumentation.newActivity()`方法，在这两步中进行Intent的改造。因此，使用了一个自定义的`Instrumentation`，并通过反射的方式替换了`ActivityThread`的`Instrumentation`。

### Service

Hook了`ActivityManagerNative`中的单例`gDefault`。替换成了占位的Service。通过占位`Service`进行分发。

### BroadcastReceiver

静态转动态。

### ContentProvider

通过一个代理Provider进行分发。



## RePlugin

前面的步骤还是使用占位的Activity绕过AMS的检查，最后启动的过程，通过自定义一个ClassLoader，加载插件的class实现最终的替换。只进行了ClassLoader的hook，侵入性小。

### Activity

Activity在启动的时候替换成了合适的占坑的activity，然后ClassLoader.loadClass的时候根据占坑Activity到真正Activity的映射关系，输入占坑Activity，返回真正Activity 的类，避免了需要hook。

### Service

Service实现逻辑是这里其实是是直接在UI线程调用了service 的相关生命周期的方法，同时启动一个service来提高service所在进程优先级。

### BroadcastReceiver

BroadcastReceiver是把所有静态注册动态注册在一个代理Receiver，收到广播在代理Receiver进行分发。

### ContentProvider

ContentProvider这里使用的是代理Contentprovider，在对应的生命周期使用反射将对应类生成出来，然后调用对应的声明周期方法。