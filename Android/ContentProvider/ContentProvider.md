# ContentProvider

## 概述

`ContentProvider`是一层对数据的封装抽象，常用于向其它应用提供数据（当然也可以用于应用内的数据封装）。它提供了一套完整的数据`CRUD`接口，并且有完备的安全性机制，让应用向外部暴露数据时可以不用考虑这些设计问题。

使用`ContentProvider`的优点是，不用向client端暴露内部数据的组织结构，内部实现可以是`SQLite`数据库，文件数据系统或者其它方式。在改变内部数据存储结构或者使用框架时，不会影响到client端。如果需要向其它应用提供数据，或者需要`ContentProvider`提供的数据层封装，可以考虑使用。



## URI

`CotentProvider`内部使用`URI`来定位数据，它可以用来定位一条记录，或者一组记录。

### URI vs. URL

`URI`是`Uniform Resource Identifier`的缩写，用于标记唯一的一个资源。

`URL`是`Uniform Resource Locator`的缩写，用于表示资源的唯一路径。

### Pattern

在与`ContentProvider`配合使用时，`URI`的Pattern定为：`content://authorities/path/id`。

* `content://`：scheme，是一个固定标识。
* `authorities`：与`AndroidManifest.xml`中`<provider>`标签上的`authorities`属性相同，用来标识一个唯一的`ContentProvider`。
* `path`：路径，用来区分不同的表（或者不同的数据内容），常用来表示`SQLite`数据库中的表。
* `id`：表中的唯一id，常用来表示`SQLite`表中的主键id。

`URI`的使用上还可以结合匹配通配符：

* `#`：可以用来匹配任意长度的数字字符串

  `content://authorities/path/#`用于匹配表中的任意一条数据。

* `*`：可以用来匹配任意长度的字符串

  `content://authorities/*`同于匹配`ContentProvider`中的任意一条数据。

### 辅助类

系统提供了两个类辅助使用`URI`：`ContentUris`和`UriMatcher()`。

* `ContentUris`

  使用它我们可以方便地操作`URI`中的id。

  * `ContentUris.withAppendedId(Uri uri, long id)`

    用于向一个表级别的`URI`追加一个id。

  * `ContentUris.parseId(Uri uri)`

    用于解析一个`URI`的id。

* `UrlMatcher`

  使用它我们可以方便地判断请求的`URI`是希望获取什么数据。

  * `UriMatcher.addUri(String authorities, String path, int code)`

    用于向`Matcher`添加一条匹配规则。

  * `UriMatcher.match(Uri uri)`

    用于查询匹配`URI`对应的`code`值。

### Authorities定义

一个`authorities`对应一个`ContentProvider`，它的命名不能有重复。

`authorities`一般可以设计为包名加上`ContentProvider`名称，避免和其它应用冲突。

可以在`Build.Config`中定义全局`authorities`变量，方便使用。

```gradle
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "zhaoyun.techstack.android.contentprovider"
        minSdkVersion 21
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

        def authorities = applicationId + '.provider'
        buildConfigField 'String', 'AUTHORITIES', '"' + authorities + '"'
        manifestPlaceholders = [PROVIDER_AUTHORITIES : authorities]
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
> 注意
>
> `buildConfigField`使用的`String`需要大写，表示`java.lang.String`类。
>
> `resValue`使用的`string`需要小写，表示resource标签`<string></string>`。

在`AndroidManifest.xml`中使用：
```xml
<provider
    android:authorities="${PROVIDER_AUTHORITIES}"
    android:name=".provider.AnimeContentProvider"
    android:exported="false"/>
```
在代码中使用：
```java
mUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
mUriMatcher.addURI(BuildConfig.AUTHORITIES, AnimeTable.TABLE_NAME, MATCH_CODE_ANIME);
```



## 创建ContentProvider

### 继承

`ContentProvider`是一个抽象类，自定义一个`ContentProvider`需要继承`ContentProvider`类，然后需要实现它定义的抽象方法。

* `onCreate() ` 

    初始化方法。

    `onCreate()`方法运行在主线程。

* `insert()`，`delete()`，`update()`，`query()`

  `CRUD`方法。

  这些方法并不线程安全，它们不一定运行在主线程。如果底层实现是`SqliteDatabase`，并没有关系，因为`SqliteDatabase`内部做了线程同步。  

  >如果调用方和`ContentProvider`在同一进程中，上述方法运行在调用方所处的线程中。如果调用方和`ContentProvider`不在同一进程中，上述方法运行在`ContentProvider`进程的**Binder线程池**中（不是主线程）。

  处理`CRUD`方法请求的`URI`时，可以使用`UriMatcher`类，分别请求的条目。

  如下例，可以用来区分请求一条记录还是所有记录。如果是一条记录，可以使用`ContentUris`类来快速获取`id`。

  ```java
  switch (mUriMatcher.match(uri)) {
      case MATCH_CODE_ANIME:
          break;
      case MATCH_CODE_ANIME_ITEM:
          long id = ContentUris.parseId(uri);
          break;
      default:
          break;
  }
  ```

* `ContentProvider.call()`
  
    自定义的操作，这个方法可以实现定制的需求。

### 回调

可以在`CRUD`方法数据发生变化时，调用`ContentResolver.notifyChange(Uri uri)`方法通知`ContentObserver`发生的内容变化。

```java
private void notifyChange(Uri itemUri) {
    Context context = getContext();
    if (context != null) {
        context.getContentResolver().notifyChange(itemUri, null);
    }
}
```

为了给`ContentProvider.query()`返回的`Cursor`也提供数据变化的回调，需要调用`Cursor.setNotificationUri()`方法设置它关联的`URI`，以便在调用`ContentResolver.notifyChange()`方法时正确通知给`Cursor`的`Observer`。

```java
if (cursor != null && getContext() != null) {
    cursor.setNotificationUri(getContext().getContentResolver(), uri);
}
```

### 注册

`ContentProvider`需要在`AndroidManifest.xml`中进行注册。

```xml
<provider
          android:authorities="${PROVIDER_AUTHORITIES}"
          android:name=".provider.AnimeContentProvider"
          android:grantUriPermissions="true"
          android:exported="true"
          tools:ignore="ExportedContentProvider">
</provider>
```



## 请求ContentProvider

### ContentResolver

系统提供了`ContentResolver`类作为向`ContentProvider`的请求的统一入口。它也提供了`insert()`，`delete()`，`update()`，`query()`的`CRUD`操作方法。

并且`CRUD`参数上还定义了进行数据库操作的参数，以`query()`为例：

```java
public final Cursor query(
    Uri uri, 
    String[] projection, 
    String selection,
    String[] selectionArgs,
    String sortOrder) {
	...
}
```

`projection`，`selection`，`selectionArgs`，`sortOrder`提供了向数据库`ContentProvider`请求时的参数控制：

* `projection`定义了数据需要返回的字段。
* `selection`定义了`where`语句条件。
* `selection`定义了`where`语句中`?`占位的参数值。
* `sortOrder`定义了排序方式。

经常有问题会提到为什么`ContentProvider`和`ContentResolver`两个类都提供了`CRUD`方法，而我们请求的时候使用的是`ContentResolver`，为什么不是直接请求`ContentProvider`呢？

我们得从多进程的角度进行考虑。

在跨进程的请求时，Client端是无法获取到Server提供的`ContentProvider`对象的，两者分处于不同的进程。因此，直接调用`ContentProvider`对象的`CRUD`方法无从谈起。此时，必须通过`Binder`机制进行进程间的通信。而`ContentResolver`内部帮助我们封装了这一系列操作。

其具体的调用路径如下。

以`ContentResolver.query()`为例，其内部调用了`ContentResolver.acquireUnstableProvider()`获取`IContentProvider`接口进行请求的处理。

```java
public final Cursor query(...) {
    ...
	IContentProvider unstableProvider = acquireUnstableProvider(uri);
    ...
    qCursor = unstableProvider.query(mPackageName, uri, projection,
                        queryArgs, remoteCancellationSignal);
    ...
}
```

`ContentResolver`的实例是`ContentImpl`下的内部类`ApplicationContentResolver`。

```java
private final ApplicationContentResolver mContentResolver;

@Override
public ContentResolver getContentResolver() {
    return mContentResolver;
}
```

最终，交给`ActivityThread`获取目标`IContentProvider`接口。

```java
@Override
protected IContentProvider acquireUnstableProvider(Context c, String auth) {
    return mMainThread.acquireProvider(
        c,
        ContentProvider.getAuthorityWithoutUserId(auth),
        resolveUserIdFromAuthority(auth), 
        false
    );
}
```

交给`ActivityThread`的目的是为了在跨进程的场景下统一调用`ActivityManagserService`。

```java
public final IContentProvider acquireProvider(...) {
    final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
    if (provider != null) {
        return provider;
    }

    ContentProviderHolder holder = null;
    try {
        synchronized (getGetProviderLock(auth, userId)) {
            holder = ActivityManager.getService().getContentProvider(
                getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
        }
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    if (holder == null) {
        Slog.e(TAG, "Failed to find provider info for " + auth);
        return null;
    }

    holder = installProvider(c, holder, holder.info,
                             true /*noisy*/, holder.noReleaseNeeded, stable);
    return holder.provider;
}
```

并且这样也方便系统维护一些必要的参数，封装一些必要的预处理操作。



## 监听ContentProvider

### ContentObserver

系统提供了`ContentObserver`类监听`ContentProvider`的数据变化。

在Client端，通过调用`ContentResolver().registerObserver()`方法注册监听。

```java
getContentResolver().registerContentObserver(
    uri, 
    true, 
    new ContentObserver(null) {
        @Override
        public void onChange(boolean selfChange, Uri uri) {
        }
    }
);
```



## 展示Query结果

调用`ContentResolver.query()`后，得到的是一个`Cursor`对象。有多种实现方式与`RecyclerView`配合进行结果的展示。

### Cursor to bean

最简单的一种方式就是遍历`Cursor`中的所有数据，转换成Bean对象的列表后，交给`RecyclerView`展示。这种方式实现简单，但是也有其弊端。

当数据量庞大的时候，Cursor中的内容需要全部加载到内存中，并且创建Bean对象的时间可能会较耗时。

如果操作数据的步骤在异步线程中，Bean对象列表容易出现线程安全问题。

### Cursor

另一种方式可以参考`CursorAdapter`的实现，让`Adapter`直接持有`Cursor`对象，在`Adapter.onBindView()`方法执行时再进行解析`Cursor`的值。

```java
@Override
public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
    if (mCursor.moveToPosition(position)) {
        ...
    }
}
```

并提供数据源发生变化时的更新方法。

```java
public void swapCursor(Cursor cursor) {
    if (cursor != null && mCursor != cursor) {
        mCursor = cursor;
        notifyDataSetChanged();
    }
}
```

在`Activity`中，使用`Loader`进行`ContentProvider`请求。让`Activity`实现`LoaderManager.LoaderCallbacks<Cursor>`接口。

在`onCreateLoader()`方法中用目标`URI`创建`CursorLoader`。

```java
@NonNull
@Override
public Loader<Cursor> onCreateLoader(int id, @Nullable Bundle args) {
    Uri uri = new Uri.Builder()
        .scheme(ContentResolver.SCHEME_CONTENT)
        .authority(BuildConfig.AUTHORITIES)
        .path(AnimeTable.TABLE_NAME)
        .build();
    return new CursorLoader(this, uri, null, null, null, null);
}
```

在`onLoadFinishied()`方法中，更新`Adapter`的数据。

```java
@Override
public void onLoadFinished(@NonNull Loader<Cursor> loader, Cursor data) {
    mAnimeAdapter.swapCursor(data);
}
```

在`onLoaderReset()`方法中清空`Adapter`的数据。

```java
@Override
public void onLoaderReset(@NonNull Loader<Cursor> loader) {
    mAnimeAdapter.swapCursor(null);
}
```

这种实现方式的底层会使用`ContentObserver`监听创建`CursorLoader`时使用的`URI`，在`ContentProvider`数据发生变化时，回调`onLoadFinished()`方法自动更新数据。
内部是通过`Cursor.registerContentObserver()`方法实现的，与`CursorAdapter`的实现原理一致。

```java
@Override
public Cursor loadInBackground() {
    synchronized (this) {
        if (isLoadInBackgroundCanceled()) {
            throw new OperationCanceledException();
        }
        mCancellationSignal = new CancellationSignal();
    }
    try {
        Cursor cursor = ContentResolverCompat.query(getContext().getContentResolver(),
                                                    mUri, mProjection, mSelection, mSelectionArgs, mSortOrder,
                                                    mCancellationSignal);
        if (cursor != null) {
            try {
                // Ensure the cursor window is filled.
                cursor.getCount();
                cursor.registerContentObserver(mObserver);
            } catch (RuntimeException ex) {
                cursor.close();
                throw ex;
            }
        }
        return cursor;
    } finally {
        synchronized (this) {
            mCancellationSignal = null;
        }
    }
}
```



## Permission

详细使用方法需要仔细阅读官方文档[Implementing permissions](https://developer.android.com/guide/topics/providers/content-provider-creating#implementing-permissions)。
### 全局Permission
在`<provider>`标签里设置以下三种属性可以指定全局读写、全局读、全局写三种权限：
```xml
<provider 
    android:permission="zhaoyun.techstack.android.contentprovider.PROVIDER"
    android:readPermission="zhaoyun.techstack.android.contentprovider.PROVIDER_READ"
    android:writePermission="zhaoyun.techstack.android.contentprovider.PROVIDER_WRITE"
/>
```

### Uri指定Permission
在`<provider>`标签里添加子标签`<path-permission>`可以为指定的Uri设置权限：
```xml
<provider>
    <path-permission 
        android:path="string"
        android:pathPrefix="string"
        android:pathPattern="string"
        android:permission="string"
        android:readPermission="string"
        android:writePermission="string"
    />
</provider>
```

### 临时Permission
有时需要向其它App提供临时的资源访问权限。

比如需要调用三方App显示图片，就需要临时提供图片Uri的读取权限。

实现分为两部分：  

* 首先，需要在`ContentProvider`所在的`AndroidManifest.xml`中设置允许临时权限。  
* 然后，需要在向其它App发送的Intent中设置资源Uri以及特定的Flag参数。

在`<provider>`标签里可以设置允许使用临时权限：  
* 设置`<provider>`标签里的属性`android:grantUriPermission`为`true`表示允许使用整个provider的临时权限。
    ```xml
    <provider android:grantUriPermission="true"/>
    ```
* 添加`<provider>`标签里的子标签`<grant-uri-permission>`表示某个的指定`URI`允许用临时权限。
    ```xml
    <provider>
        <grant-uri-permission 
            android:path="string"
            android:pathPattern="string"
            android:pathPrefix="string" 
        />
    </provider>
    ```

在`Intent`里添加需要使用的`URI`和授予权限的Flag：
* `FLAG_GRANT_READ_URI_PERMISSION`  
* `FLAG_GRANT_WRITE_URI_PERMISSION`



## 其它用途

供SDK在应用启动时初始化。

> 参考：[使用ContentProvider初始化你的Library](https://www.jianshu.com/p/5c0570263dfd)





# Reference

* [Content providers](https://developer.android.com/guide/topics/providers/content-providers)
* [Android：关于ContentProvider的知识都在这里了！](https://www.jianshu.com/p/ea8bc4aaf057)
* [图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)