# Reference
* [Understand the Activity Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)

# Life cycles
	onCreate
	onRestart
	onStart
	onRestoreInstanceState
	onResume
	onPause
	onSaveInstanceState
	onStop
	onDestroy

# Q&A
Q :  
横竖屏切换的时候，Activity 各种情况下的生命周期

A :  
默认情况下：
```
10-14 10:08:50.677 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onPause()
10-14 10:08:50.727 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onSaveInstanceState()
10-14 10:08:50.736 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStop()
10-14 10:08:50.743 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onDestroy()
10-14 10:08:50.827 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onCreate()
10-14 10:08:50.889 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStart()
10-14 10:08:50.894 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onRestoreInstanceState()
10-14 10:08:50.901 5707-5707/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onResume()
```
AndroidManifest.xml设置configChanges后：
```xml
<activity 
	android:name=".SecondActivity"
	android:configChanges="orientation">
</activity>
```
```
10-14 10:12:40.423 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onConfigurationChanged()
```
		
Q :  
前台切换到后台，然后再回到前台，Activity生命周期回调方法。弹出Dialog，生命值周期回调方法。

A :  
```
10-14 12:49:09.739 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onCreate()
10-14 12:49:09.883 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStart()
10-14 12:49:09.889 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onResume()
```
弹出Dialog
```
10-14 12:49:34.533 681-681/zhaoyun.techstack.android.activity.lifecycles I/SimpleDialogFragment: onCreateDialog()
```
点击Home切到后台
```
10-14 12:49:45.938 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onPause()
10-14 12:49:46.051 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onSaveInstanceState()
10-14 12:49:46.107 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStop()
```
返回前台
```
10-14 12:50:03.347 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onRestart()
10-14 12:50:03.361 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStart()
10-14 12:50:03.366 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onResume()
```
		
Q :  
Activity上有Dialog的时候按Home键时的生命周期

A :  
和普通情况一样
```
10-14 12:49:09.739 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onCreate()
10-14 12:49:09.883 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStart()
10-14 12:49:09.889 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onResume()
```
弹出Dialog
```
10-14 12:49:34.533 681-681/zhaoyun.techstack.android.activity.lifecycles I/SimpleDialogFragment: onCreateDialog()
```
点击Home切到后台
```
10-14 12:49:45.938 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onPause()
10-14 12:49:46.051 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onSaveInstanceState()
10-14 12:49:46.107 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStop()
```
返回前台
```
10-14 12:50:03.347 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onRestart()
10-14 12:50:03.361 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStart()
10-14 12:50:03.366 681-681/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onResume()
```
	
Q :  
两个Activity 之间跳转时必然会执行的是哪几个方法？

A :  
FirstActivity -> SecondActivity
```
10-14 10:17:16.250 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onPause()
10-14 10:17:16.479 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onCreate()
10-14 10:17:16.523 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onStart()
10-14 10:17:16.531 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onResume()
10-14 10:17:17.490 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onSaveInstanceState()
10-14 10:17:17.498 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStop()
```

FirstActivity <- SecondActivity
```
10-14 10:19:23.404 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onPause()
10-14 10:19:23.542 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onActivityResult()
10-14 10:19:23.552 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onRestart()
10-14 10:19:23.565 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onStart()
10-14 10:19:23.577 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/FirstActivity: onResume()
10-14 10:19:24.372 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onStop()
10-14 10:19:24.379 6150-6150/zhaoyun.techstack.android.activity.lifecycles I/SecondActivity: onDestroy()
```		
	