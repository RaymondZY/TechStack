# Reference
* [Understand Tasks and Back Stack](https://developer.android.com/guide/components/activities/tasks-and-back-stack)
* [Android Activity Launch Mode](https://android.jlelse.eu/android-activity-launch-mode-e0df1aa72242)


# ADB命令
```
adb shell dumpsys activity activities
```


# 4种LaunchMode
## Standard 
每次新建  
senario 1 :
```
A -> B -> C -> D
launch D
A -> B -> C -> D -> D
```
		
## SingleTop
栈顶复用模式  
使用场景：搜索框  
senario 1 :
```
A -> B -> C -> D
launch D
A -> B -> C -> D (D onNewIntent())
```	
Senario 2 :
```	
A -> B -> C -> D	
launch B
A -> B -> C -> D -> B
```

## SingleTask
栈内复用模式 (使用taskAffinity标签可以重新指定栈)  
使用场景：APP首页  
senario 1 :
```
A -> B -> C -> D
launch D
A -> B -> C -> D (D onNewIntent())
```
senario 2 :
```	
A -> B -> C -> D 
launch B
A -> B  (C,D destroy)
```

## SingleInstance
单例模式  
使用场景：手机拨号