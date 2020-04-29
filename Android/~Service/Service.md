# Started Service
## 生命周期
onCreate -> onStartCommand -> onDestroy
```
2018-10-24 18:21:14.491 23463-23463/zhaoyun.techstack.android.service I/MyService: StartService()
2018-10-24 18:21:14.491 23463-23463/zhaoyun.techstack.android.service I/MyService: onCreate()
2018-10-24 18:21:14.491 23463-23463/zhaoyun.techstack.android.service I/MyService: onStartCommand()
2018-10-24 18:21:17.971 23463-23463/zhaoyun.techstack.android.service I/MyService: onDestroy()
```
多次start会再次调用onStartCommand()

## onStartCommand返回值
|         Label            |INT值|
|--------------------------|-----|
|START_STICKY_COMPATIBILITY|  0  |
|START_STICKY              |  1  |
|START_NOT_STICKY          |  2  |
|START_REDELIVER_INTENT    |  3  |

* START_STICKY_COMPATIBILITY :   
START_STICKY兼容版本，并不保证会启动。
* START_STICKY :   
被系统杀掉后，系统会尝试重建Service，并保证会调用onStartCommand，Intent为空。  
但是实际测试过程中，并没有发现Service重启。(SDK 28 Emulator)  
实际使用时，不要依赖这个TAG，因为手机厂商定制的ROM为了待机性能不一定会实现这个功能。
* START_NOT_STICKY :   
被系统杀掉后，系统不会重建Service。
* START_REDELIVER_INTENT :  
被系统杀掉后，系统会尝试重建Service，并保证会调用onStartCommand，并会重新发送Intent。

## stop的方式
* context#stopService()
* service#stopSelf()

## vs. Thread
使用Service的目的是执行
[long-running operations in the background](https://developer.android.com/guide/components/services#top_of_page)。
Thread不宜执行过长的后台任务。  
Service运行在主线程，需要自己实现后台线程，或者使用IntentService。
```
2018-10-24 16:04:07.940 3058-3058/zhaoyun.techstack.android.service I/MyService: StartService()
2018-10-24 16:04:07.941 3058-3058/zhaoyun.techstack.android.service I/MyService: onCreate()
2018-10-24 16:04:07.941 3058-3058/zhaoyun.techstack.android.service I/MyService: onStartCommand()
```

## IntentService
`IntentService#onHandleIntent()`运行在后台线程。
```
2018-10-24 16:06:13.569 8618-8618/zhaoyun.techstack.android.service I/MyIntentService: onCreate()
2018-10-24 16:06:13.570 8618-8618/zhaoyun.techstack.android.service I/MyIntentService: onStartCommand()
2018-10-24 16:06:13.571 8618-8725/zhaoyun.techstack.android.service I/MyIntentService: onHandleIntent()
```
IntentService内部是一个Handler实现的后台工作队列，因此只能串行执行任务。  
IntentService在执行完所有操作后会自动destroy。
```
2018-10-24 16:16:08.116 16505-16505/zhaoyun.techstack.android.service I/MyIntentService: onCreate()
2018-10-24 16:16:08.123 16505-16505/zhaoyun.techstack.android.service I/MyIntentService: onStartCommand()
2018-10-24 16:16:08.144 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: onHandleIntent()
2018-10-24 16:16:08.145 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: Intent = Intent { cmp=zhaoyun.techstack.android.service/.MyIntentService }
2018-10-24 16:16:08.145 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: Simulating long-running operation for 5 seconds..
2018-10-24 16:16:10.331 16505-16505/zhaoyun.techstack.android.service I/MyIntentService: onStartCommand()
2018-10-24 16:16:13.147 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: Job done
2018-10-24 16:16:13.165 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: onHandleIntent()
2018-10-24 16:16:13.165 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: Intent = Intent { cmp=zhaoyun.techstack.android.service/.MyIntentService }
2018-10-24 16:16:13.167 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: Simulating long-running operation for 5 seconds..
2018-10-24 16:16:18.170 16505-16627/zhaoyun.techstack.android.service I/MyIntentService: Job done
2018-10-24 16:16:18.173 16505-16505/zhaoyun.techstack.android.service I/MyIntentService: onDestroy()
```

# Bound Service
## 生命周期
onCreate -> onBind-> onUnbind -> onDestroy
```
2018-10-24 18:19:35.592 23463-23463/zhaoyun.techstack.android.service I/BindService: onCreate()
2018-10-24 18:19:35.593 23463-23463/zhaoyun.techstack.android.service I/BindService: onBind()
2018-10-24 18:19:35.680 23463-23463/zhaoyun.techstack.android.service I/MainActivity: onServiceConnected()
2018-10-24 18:19:39.096 23463-23463/zhaoyun.techstack.android.service I/BindService: onUnbind()
2018-10-24 18:19:39.097 23463-23463/zhaoyun.techstack.android.service I/BindService: onDestroy()
```
多次bind不会回调onBind()

## IPC调用
* [Messenger](https://developer.android.com/guide/components/bound-services#Messenger)  
Messenger使用的是Handler消息队列，因此不需要考虑多线程调用的问题，所有请求将会顺序执行。  
为了实现双向通信需要在Client、Service两端都建立Handler处理Message。  

	```java
		public IBinder onBind(Intent intent) {
			Messenger messenger = new Messenger(new WorkHandler());
			return messenger.getBinder();
		}
	```
	```java
		private static class WorkHandler extends Handler {

			@Override
			public void handleMessage(Message msg) {
				Log.i(TAG, "handleMessage()");

				switch (msg.what) {
					case MESSAGE_TIME:
						Bundle data = new Bundle();
						data.putLong(MainActivity.MESSAGE_DATA_KEY_TIME, System.currentTimeMillis());
						Message message = Message.obtain(null, MainActivity.MESSAGE_TIME);
						message.setData(data);
						try {
							msg.replyTo.send(message);
						} catch (RemoteException e) {
							Log.e(TAG, e.getMessage(), e);
						}
						break;

					default :
						super.handleMessage(msg);
				}
			}
		}
	```
	```java
		Message message = Message.obtain(null, RemoteService.MESSAGE_TIME);
		message.replyTo = new Messenger(new RemoteReplyHandler());
		try {
			mRemoteServiceMessenger.send(message);
		} catch (RemoteException e) {
			Log.e(TAG, e.getMessage(), e);
		}
	```

* [AIDL](https://developer.android.com/guide/components/aidl)