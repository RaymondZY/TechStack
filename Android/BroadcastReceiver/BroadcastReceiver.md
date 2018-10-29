# Reference
* [
Broadcasts overview - Android Developers](https://developer.android.com/guide/components/broadcasts)
* [Android四大组件：BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)

# Send Broadcast
使用`Context#sendBroadcast()`发送Broadcast。
```java
Intent intent = new Intent(BuildConfig.BROADCAST_ACTION);
sendBroadcast(intent);
```

# Receive Broadcast
BroadcastReceiver可以静态注册或者动态注册。
## Static BroadcastReceiver
静态注册需要在AndroidManifest中添加`<receiver>`标签，并设置需要接收的Broadcast的Action。   
需要注意，如果应用TargetSDK>=26（Android 8.0），静态注册的BroadcastReceiver将不能收到大部分系统Broadcast，也无法收到除了明确指定了包名的三方Broadcast。
```xml
<receiver
    android:name=".StaticNoPermissionBroadcastReceiver">
    <intent-filter>
        <action android:name="${BROADCAST_ACTION}"/>
    </intent-filter>
</receiver>
```

## Dynamic BroadcastReceiver
动态注册需要在运行时调用`Context#registerReceiver()`注册，  
并且可以调用`Context#unregisterReceiver()`取消注册。
```java
IntentFilter intentFilter = new IntentFilter(BuildConfig.BROADCAST_ACTION);
registerReceiver(
        mNoPermissionBroadcastReceiver,
        intentFilter
);
```
```java
unregisterReceiver(mNoPermissionBroadcastReceiver);
```
最好在`onResume()`和`onPause()`里完成注册和取消注册，因为考虑到App在onPause()时是可能被系统回收内存的。
> 这个观点参考[Android四大组件：BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)。

## onReceive()
注意，回调`onReceive()`在UI线程，不要在这里做长时间的操作。如果需要在后台执行一些操作，需要使用`BroadcastReceiver#goAsync()`获取一个`PendingResult`对象，并且在操作执行完毕后调用`PendingResult#finish()`。否则系统会误认为BroadcastReceiver已经使用完毕，后台的任务会有被系统杀死的可能。
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

# Security
为了避免发送和接受Broadcast被三方app利用，需要限定Broadcast的发送方和接受方。可以自定义`<permission>`来实现限定效果。

## Protection Level
可以使用`<permission>`标签的`protectionLevel`属性指定使用Permission的应用必须和声明Permission的应用签名一致。
```xml
<permission
    android:name="some.permission"
    android:protectionLevel="signature" />
```

## 限定接收方
发送方的设置分为2步：  
1. 在AndroidManifest中增加自定义的`<permission>`标签。
    ```xml
    <permission android:name="${BROADCAST_SENDER_PERMISSION}"/>
    ```
2. 在调用`Context#sendBroadcast()`时加入Permission参数。
    ```java
    Intent intent = new Intent(BuildConfig.BROADCAST_ACTION);
    sendBroadcast(intent, BuildConfig.BROADCAST_SENDER_PERMISSION);
    ```
接受方的设置只有1步：
1. 在`AndroidManifest`中使用`<use-permission>`标签。
    ```xml
    <uses-permission android:name="${BROADCAST_SENDER_PERMISSION}"/>
    ```

## 限定发送方
接受方的设置分为2步：
1. 在AndroidManifest中增加自定义的`<permission>`标签。
    ```xml
    <permission android:name="${BROADCAST_RECEIVER_PERMISSION}"/>
    ```
2. 在调用`Context#registerBroadcastReceiver()`时加入permission参数。
    ```java
    IntentFilter intentFilter = new IntentFilter(BuildConfig.BROADCAST_ACTION);
    registerReceiver(
                mReceiverPermissionBroadcastReceiver,
                intentFilter,
                BuildConfig.BROADCAST_RECEIVER_PERMISSION,
                null
        );
    ```
    如果使用的是Static Receiver，可以在`<receiver>`标签添加`permission`属性。
发送方的设置只有1步：
1. 在`AndroidManifest`中使用`<use-permission>`标签。
    ```xml
    <uses-permission android:name="${BROADCAST_RECEIVER_PERMISSION}"/>
    ```

## 声明Permission的小技巧
可以使用Gradle脚本动态声明Permission参数，保证AndroidManifest和代码中使用的Permission保持一致。
```gradle
// Project Level Gradle

ext {
    broadcastParams = [
            senderPermission : 'zhaoyun.techstack.android.broadcastreceiver.sender.PERMISSION',
            receiverPermission : 'zhaoyun.techstack.android.broadcastreceiver.receiver.PERMISSION',
            action : 'zhaoyun.techstack.android.broadcastreceiver.ACTION'
    ]
}
```
```gradle
// App Level Gradle

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
使用`Intent#setPackage`可以限制接收方的包名。在TargetSDK>=26时，也只有设定了Package的Broadcast才能被静态BroadcastReceiver捕获。

# LocalBroadcastManager
使用`LocalBroadcastManager`可以实现只在进程内发送广播。其内部实现是一个Handler。  
注意：回调仍然是在UI线程。

## 监听
```java
IntentFilter intentFilter = new IntentFilter(BuildConfig.BROADCAST_ACTION);
mBroadcastReceiver = new LocalBroadcastReceiver();
LocalBroadcastManager.getInstance(this).registerReceiver(mBroadcastReceiver, intentFilter);
```

## 取消监听
``` java
LocalBroadcastManager.getInstance(this).unregisterReceiver(mBroadcastReceiver);
```

## 发送
```java
Intent intent = new Intent(BuildConfig.BROADCAST_ACTION);
LocalBroadcastManager.getInstance(this).sendBroadcast(intent);
```