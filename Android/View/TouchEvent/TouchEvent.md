# Reference
* [Manage touch events in a ViewGroup - Android Developers](https://developer.android.com/training/gestures/viewgroup)
* [Android的Touch事件分发机制简单探析](https://www.cnblogs.com/net168/p/4165970.html)
* Source code : `ViewGroup.java`, `View.java` SDK verion 28

# 流程
`TouchEvent`的分发过程涉及三个重要的方法。
* `ViewGroup#dispatchTouchEvent()`
* `ViewGroup#onInterceptTouchEvent()`
* `View#onTouchEvent()`

# Situation 1
|Return|dispatchTouchEvent|onInterceptTouchEvent|onTouchEvent|
|:--|:--:|:--:|:--:|
|<font color=#00FF00>MainActivity</font>|false|x|false|
|ContainerConstraintLayout|false|false|false|
|SampleTextView|false|false|false|

Logcat输出
```
I/MainActivity:                 dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onInterceptTouchEvent() ACTION_DOWN
I/SampleTextView:               dispatchTouchEvent() ACTION_DOWN
I/SampleTextView:               onTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onTouchEvent() ACTION_DOWN
I/MainActivity:                 onTouchEvent() ACTION_DOWN

I/MainActivity:                 dispatchTouchEvent() ACTION_UP
I/MainActivity:                 onTouchEvent() ACTION_UP
```

# Situation 2
|Return|dispatchTouchEvent|onInterceptTouchEvent|onTouchEvent|
|:--|:--:|:--:|:--:|
|MainActivity|false|x|false|
|<font color=#00FF00>ContainerConstraintLayout</font>|false|false|<font color=#00FF00>true</font>|
|SampleTextView|false|false|false|

Logcat输出
```
I/MainActivity:                 dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onInterceptTouchEvent() ACTION_DOWN
I/SampleTextView:               dispatchTouchEvent() ACTION_DOWN
I/SampleTextView:               onTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onTouchEvent() ACTION_DOWN

I/MainActivity:                 dispatchTouchEvent() ACTION_UP
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_UP
I/ContainerConstraintLayout:    onTouchEvent() ACTION_UP
```

# Situation 3
|Return|dispatchTouchEvent|onInterceptTouchEvent|onTouchEvent|
|:--|:--:|:--:|:--:|
|MainActivity|false|x|false|
|ContainerConstraintLayout|false|false|true|
|<font color=#00FF00>SampleTextView</font>|false|false|<font color=#00FF00>true</font>|

Logcat输出
```
I/MainActivity:                 dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onInterceptTouchEvent() ACTION_DOWN
I/SampleTextView:               dispatchTouchEvent() ACTION_DOWN
I/SampleTextView:               onTouchEvent() ACTION_DOWN

I/MainActivity:                 dispatchTouchEvent() ACTION_UP
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_UP
I/ContainerConstraintLayout:    onInterceptTouchEvent() ACTION_UP
I/SampleTextView:               dispatchTouchEvent() ACTION_UP
I/SampleTextView:               onTouchEvent() ACTION_UP
```

# Situation 4
|Return|dispatchTouchEvent|onInterceptTouchEvent|onTouchEvent|
|:--|:--:|:--:|:--:|
|MainActivity|false|x|false|
|<font color=#00FF00>ContainerConstraintLayout</font>|false|<font color=#00FF00>true</font>|<font color=#00FF00>true</font>|
|SampleTextView|false|false|true|

Logcat输出
```
I/MainActivity:                 dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onInterceptTouchEvent() ACTION_DOWN
I/ContainerConstraintLayout:    onTouchEvent() ACTION_DOWN

I/MainActivity:                 dispatchTouchEvent() ACTION_UP
I/ContainerConstraintLayout:    dispatchTouchEvent() ACTION_UP
I/ContainerConstraintLayout:    onTouchEvent() ACTION_UP
```