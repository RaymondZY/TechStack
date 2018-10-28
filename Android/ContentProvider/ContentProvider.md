# Reference
* [Content providers](https://developer.android.com/guide/topics/providers/content-providers)
* [Android：关于ContentProvider的知识都在这里了！](https://www.jianshu.com/p/ea8bc4aaf057)
* [图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)
* [ContentObserver原理](ContentObserver原理)

# URI
URI : Uniform Resource Identifier 用于标记唯一的一个资源。  
URL : Uniform Resource Locator 用于表示资源的路径。

## 使用细节
Pattern：  
`content://authorities/path/id`  
`content://authorities/path/#`  
`content://authorities/*`

Utils类：  
`ContentUris`类
* `ContentUris#withAppendedId(Uri uri, long id)` 追加一个Id
* `ContentUris#parseId(Uri uri)` parse一个Id

`UriMatcher`类
* `UriMatcher#addUri(String authorities, String path, int code)` 添加一条Match规则

可以在`Build.Config`中定义全局authorities变量，方便使用。
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
注意:  
`buildConfigField`使用的`String`需要大写，表示`java.lang.String`类,  
`resValue`使用的`string`需要小写，表示resource标签`<string></string>`。

在AndroidManifest.xml中使用：
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

# ContentProvider
## 作用
* 应用间共享数据
* 提供原始数据存储之上的一层抽象
* 供SDK在应用启动时初始化
    > 参考：[使用ContentProvider初始化你的Library](https://www.jianshu.com/p/5c0570263dfd)

## 原理
底层使用Binder进行进程间通信。

## 方法
初始化方法：
* `onCreate() ` 
    注意`onCreate()`方法运行在主线程，不要在这里做耗时的操作。  
    耗时的操作可以等到第一次查询，构建数据库`SQLiteOpenHelper#onOpen()`回调时进行。

CRUD的核心方法：
* `ContentProvider#insert()`
* `ContentProvider#delete()`
* `ContentProvider#update()`
* `ContentProvider#query()`

注意这些方法并不线程安全。  
如果只是使用一个SqliteDatabase，并没有关系，因为SqliteDatabase内部做了线程同步。  

如果调用方和ContentProvider在同一进程中，上述方法运行在调用方所处的线程中。  
如果调用方和ContentProvider不在同一进程中，上述方法运行在ContentProvider进程的**Binder线程池**中（不是主线程）。参考[IBinder](https://developer.android.com/reference/android/os/IBinder)的Reference文档：

> The system maintains a pool of transaction threads in each process that it runs in. These threads are used to dispatch all IPCs coming in from other processes. For example, when an IPC is made from process A to process B, the calling thread in A blocks in transact() as it sends the transaction to process B. The next available pool thread in B receives the incoming transaction, calls Binder.onTransact() on the target object, and replies with the result Parcel. Upon receiving its result, the thread in process A returns to allow its execution to continue. In effect, other processes appear to use as additional threads that you did not create executing in your own process.

自定义的操作：
* `ContentProvider.call()`  
    这个方法可以实现定制的需求。

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
* 首先，需要在ContentProvider所在的AndroidManifest设置允许临时权限。  
* 然后，需要在向其它App发送的Intent中设置资源Uri以及特定的Flag参数。

在`<provider>`标签里可以设置允许使用临时权限：  
* 设置`<provider>`标签里的属性`android:grantUriPermission`为`true`表示允许使用整个provider的临时权限。
    ```xml
    <provider android:grantUriPermission="true"/>
    ```
* 添加`<provider>`标签里的子标签`<grant-uri-permission>`表示某个的指定uri允许用临时权限。
    ```xml
    <provider>
        <grant-uri-permission 
            android:path="string"
            android:pathPattern="string"
            android:pathPrefix="string" 
        />
    </provider>
    ```

在Intent里添加需要使用的Uri和授予权限的Flag：
* `FLAG_GRANT_READ_URI_PERMISSION`  
* `FLAG_GRANT_WRITE_URI_PERMISSION`

# ContentResolver
* 为App提供操作ContentProvider的统一接口。
* 为ContentObserver提供了统一的连接ContentService的接口。  
参考：[ContentObserver原理](ContentObserver原理)

# ContentObserver
* 为App提供监听ContentProvider变化的回调接口。
* 底层实现是在ServiceManager注册的ContentService完成了进程间的通信。  
参考：[ContentObserver原理](ContentObserver原理)