# Broadcast

## 概述

广播可以被用于App内，或者App间通信，是一种最基础的通讯机制。它使用的是订阅者模式，接收方向系统注册需要监听的广播，当发送方发送广播时，进行回调。



## 种类

广播有不同的类型，按照粒度可以分为：系统广播，应用间广播，应用内广播。按照监听方式可以分为：`AndroidManifest`静态注册广播，`Context`动态注册广播。



## 发送

最简单的可以使用`Context.sendBroadcast()`发送Broadcast。
```java
Intent intent = new Intent(BuildConfig.BROADCAST_ACTION);
sendBroadcast(intent);
```

还可以在`Intent`中设置权限限制，包名限制等。



## 接收

接收广播需要继承`BroadcastReceiver`类，并实现`BroadcastReceiver.onReceive()`方法，然后并向系统注册。注册可以分为静态注册或者动态注册。
### 静态注册
静态注册需要在`AndroidManifest`中添加`<receiver>`标签，并设置需要接收的`<action>`。

```xml
<receiver
    android:name=".StaticNoPermissionBroadcastReceiver">
    <intent-filter>
        <action android:name="${BROADCAST_ACTION}"/>
    </intent-filter>
</receiver>
```

> 需要注意，如果应用`targetSDK >= 26(Android 8.0)`，静态注册将不能收到大部分系统广播。

### 动态注册

动态注册需要在运行时调用`Context.registerReceiver()`注册。
```java
IntentFilter intentFilter = new IntentFilter(BuildConfig.BROADCAST_ACTION);
registerReceiver(
        mNoPermissionBroadcastReceiver,
        intentFilter
);
```
并且可以调用`Context.unregisterReceiver()`取消注册。

```java
unregisterReceiver(mNoPermissionBroadcastReceiver);
```
最好在`onResume()`和`onPause()`里完成注册和取消注册，因为考虑到`Activity`在`onPause()`时是可能被系统回收内存的。
> 这个观点参考[Android四大组件：BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)。



## 回调

广播发出时，会回调`BroadcastReceiver.onReceive()`方法。回调发生在应用主线程，不要在这里做长时间的操作。如果需要在后台执行一些操作，需要使用`BroadcastReceiver#goAsync()`获取一个`PendingResult`对象，并且在操作执行完毕后调用`PendingResult#finish()`。否则系统会误认为操作已经执行完毕，后台的任务会有被系统杀死的可能。

```java
@Override
public void onReceive(Context context, Intent intent) {
    PendingResult pendingResult = goAsync();
    new BackgroundTask(pendingResult).execute();
}
```
```java
private static class BackgroundTask extends AsyncTask<Object, Integer, Object> {

    private PendingResult mPendingResult;

    private BackgroundTask(PendingResult pendingResult) {
        mPendingResult = pendingResult;
    }

    @Override
    protected Object doInBackground(Object[] objects) {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return null;
    }

    @Override
    protected void onPostExecute(Object o) {
        super.onPostExecute(o);
        mPendingResult.finish();
    }
}
```



## 安全性

为了避免发送和接受Broadcast被三方app利用，需要限定Broadcast的发送方和接受方。可以自定义`<permission>`来实现限定效果。

### 定义Permission

可以在`AndroidManifest.xml`的`<manifest>`标签下新建`<permission>`标签新建一个自定义的权限。

```xml
<permission
    android:name="some.permission"
    android:protectionLevel="signature" />
```

`protectionLevel`属性可以指定使用`Permission`的应用必须和声明`Permission`的应用签名一致。

### 限定接收方
广播的发送方可以使用`permission`限制接收方，设置分为2步：  
1. 在`AndroidManifest.xml`中增加自定义的`<permission>`标签。
    ```xml
    <permission android:name="${BROADCAST_SENDER_PERMISSION}"/>
    ```
2. 在调用`Context.sendBroadcast()`时加入`Permission`参数。
    ```java
    Intent intent = new Intent(BuildConfig.BROADCAST_ACTION);
    sendBroadcast(intent, BuildConfig.BROADCAST_SENDER_PERMISSION);
    ```

接受方的设置只有1步：

1. 在`AndroidManifest`中使用`<use-permission>`标签。
    ```xml
    <uses-permission android:name="${BROADCAST_SENDER_PERMISSION}"/>
    ```

### 限定发送方
广播的接收方可以使用`permission`限制发送发，设置分为2步：
1. 在`AndroidManifest.xml`中增加自定义的`<permission>`标签。
    ```xml
    <permission android:name="${BROADCAST_RECEIVER_PERMISSION}"/>
    ```
    
2. 在调用`Context.registerBroadcastReceiver()`时加入`permission`参数。
    ```java
    IntentFilter intentFilter = new IntentFilter(BuildConfig.BROADCAST_ACTION);
    registerReceiver(
        mReceiverPermissionBroadcastReceiver,
        intentFilter,
        BuildConfig.BROADCAST_RECEIVER_PERMISSION,
        null
    );
    ```
    如果使用的是静态注册，可以在`<receiver>`标签添加`permission`属性。

发送方的设置只有1步：

1. 在`AndroidManifest`中使用`<use-permission>`标签。
    ```xml
    <uses-permission android:name="${BROADCAST_RECEIVER_PERMISSION}"/>
    ```

### 声明Permission的小技巧

可以使用Gradle的`ext`参数动态声明`Permission`参数，保证`AndroidManifest.xml`和代码中使用的`Permission`字符串保持一致。

在项目级别的`build.gradle`中声明`ext`参数：

```gradle
ext {
    broadcastParams = [
            senderPermission : 'zhaoyun.techstack.android.broadcastreceiver.sender.PERMISSION',
            receiverPermission : 'zhaoyun.techstack.android.broadcastreceiver.receiver.PERMISSION',
            action : 'zhaoyun.techstack.android.broadcastreceiver.ACTION'
    ]
}
```
在应用级别的`build.gradle`中声明`BuildConfig`编译参数和`AndroidManifest.xml`参数。

```gradle
android {
    defaultConfig {
        def broadcastParams = rootProject.ext.broadcastParams
        buildConfigField 'String', 'BROADCAST_ACTION', '"' + broadcastParams.action + '"'
        buildConfigField 'String', 'BROADCAST_SENDER_PERMISSION', '"' + broadcastParams.senderPermission + '"'
        buildConfigField 'String', 'BROADCAST_RECEIVER_PERMISSION', '"' + broadcastParams.receiverPermission + '"'
        manifestPlaceholders = [
                BROADCAST_SENDER_PERMISSION : broadcastParams.senderPermission,
                BROADCAST_RECEIVER_PERMISSION : broadcastParams.receiverPermission
        ]
    }
}
```

## 限定包名

使用`Intent.setPackage()`可以限制接收方的包名。在`targetSDK >= 26 (Android 8.0)`时，也只有设定了包名的广播才能被静态`BroadcastReceiver`捕获。



## LocalBroadcastManager

使用`LocalBroadcastManager`可以实现只在进程内发送广播。其内部实现是一个`Handler`。

>  注意：回调仍然是在UI线程。

### 发送

```java
Intent intent = new Intent(BuildConfig.BROADCAST_ACTION);
LocalBroadcastManager.getInstance(this).sendBroadcast(intent);
```

也可以使用`LocalBroadcastManager.sendBroadcastSync()`同步发送，如果有接收方，方法会在回调方法执行完成后再返回。

### 接收

```java
IntentFilter intentFilter = new IntentFilter(BuildConfig.BROADCAST_ACTION);
mBroadcastReceiver = new LocalBroadcastReceiver();
LocalBroadcastManager.getInstance(this).registerReceiver(mBroadcastReceiver, intentFilter);
```

### 取消接收

``` java
LocalBroadcastManager.getInstance(this).unregisterReceiver(mBroadcastReceiver);
```



# Reference

* [Broadcasts overview - Android Developers](https://developer.android.com/guide/components/broadcasts)
* [Android四大组件：BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)

