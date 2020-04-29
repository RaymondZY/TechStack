# Back Stack

Activity启动后会存在于Task中，Activity实例是否复用，是否新建Task受到两个因素的影响。

一是Activity在`AndroidManifest.xml`中定义的`launchMode`标签。

二是启动Activity的Intent中设置的Flag。

第二点的优先级比第一点更高，会覆盖`launchMode`的逻辑。



## LaunchMode

在`AndroidManifest.xml`中可以给`<Activity>`标签设置`android:launchMode`属性。这个属性可以控制Activity的启动方式，影响BackStack的处理逻辑。

`launchMode`有以下几种：

* Standard

  标准模式。

  不进行设置时的默认值。

  每次新建一个Activity添加到当前Task中。同一个Activity，可以存在于多个Task，每个Task可以拥有多个不同的实例。

  * 场景1

    ```
    A -> B -> C -> D
    launch D
    A -> B -> C -> D -> D
    ```

  

* SingleTop

  栈顶复用模式。

  每次新建一个Activity，检查当前Task的顶部是否为此Activity。如果不是，新建一个实例放到顶部，如果是，会调用`Activity.onNewIntent()`方法。

  注意在`onNewIntent()`方法中，调用`setIntent()`方法，保存新的Intent，否则再调用`getIntent()`方法时，还将会返回旧的Intent，出现错误。

  使用场景：搜索框。

  * 场景1

    ```
    A -> B -> C -> D
    launch D
    A -> B -> C -> D (D onNewIntent())
    ```

  * 场景2

    ```
    A -> B -> C -> D	
    launch B
    A -> B -> C -> D -> B
    ```



* SingleTask

  栈内复用模式。

  每次新建一个Activity，检查当前Task中是否存在此Activity。如果不存在，新建一个实例放到顶部，如果是，会将它上方的Activity全部退栈，调用`Activity.onNewIntent()`方法。

  注意在`onNewIntent()`方法中，调用`setIntent()`方法，保存新的Intent，否则再调用`getIntent()`方法时，还将会返回旧的Intent，出现错误。

  使用场景：App首页。

  * 场景1

    ```
    A -> B -> C -> D
    launch D
    A -> B -> C -> D (D onNewIntent())
    ```

  * 场景2

    ```
    A -> B -> C -> D 
    launch B
    A -> B  (destroy C, D)
    ```

  默认情况下，不额外设置taskAffinity标签时，都将在当前的Task中处理。如果设置了taskAffinity标签，将在一个新的Task中新建Activity实例。

  

* SingleInstance

  单例模式。

  这个Activity只会存在于一个Task中。如果不存在，新建一个实例放到一个新的Task中，如果存在，会调用`Activity.onNewIntent()`方法。并且任何从这个Activity启动的Activity也不会存在于它所在的Task中。

  使用场景：电话接听。



## Intent Flags

使用Intent启动Activity的时候，可以给Intent设置Flag来控制Activity的启动模式，改变BackStack的逻辑。这个Flag的优先级比`AndroidManifest.xml`中设置的`launchMode`更高，会优先满足Flag的处理逻辑。

```java
Intent intent = new Intent(Intent.ACTION_MAIN);
intent.setClassName(
    "zhaoyun.techstack.android.activity.backstack",
    "zhaoyun.techstack.android.activity.backstack.EntranceActivity"
);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

常用的有以下几种Flag：

* FLAG_ACTIVITY_NEW_TASK

  和SingleTask的逻辑相同，新建的Activity会新启用一个Task。同样地，如果有

  

* FLAG_ACTIVITY_SINGLE_TOP

  和SingleTop的逻辑相同。如果当前Task的顶部为此Activity，会回调`Activity.onNewIntent()`，否则在Task顶部加入一个Activity实例。

  

* FLAG_ACTIVITY_CLEAR_TOP

  如果当前Task中有此Activity，会移除它上方的其它Activiy，并回调`Activity.onNewIntent()`。但是，如果这个Activity的`launchMode`设置了`standard`，那么会新建一个新的实例来替换旧的。



## Task Affinity

在`AndroidManifest.xml`中，可以给`<Activity>` 标签设置`android:taskAffinity`属性，控制Task的依附性。通俗的来说，就是控制Activity要不要在一个新的Task中启动。

默认情况下，不设置`taskAffinity`属性，它的值就是应用包名字符串。这也是为什么默认Activity都是启动在同一个Task中。如果指定了`taskAffinity`，那么这个Activity就会启动在一个新的Task中。如果从用户的角度，包含不同的"App"，那么你就应该考虑把不同的”App”设置到不同的`taskAffinity`中。

`android:allowTaskReparenting`标签设置为true时，可以使Activity具备重新选择Task的能力。

举例来说，如果一个Activity从另一个App中被启动，此时它处于调用方App的Task中。但是如果再打开它自身所在的App并启动了它的话，将会复用之前在调用方Task中的实例，将它从调用方Task中移除，添加到自身App的Task中。



## 清除Back Stack

 默认情况下，如果一个用户离开一个Task比较久，那么系统会清除Task里除了Root Activity的其它Acitivity。

可以通过在`<Activity>`标签上设置以下几个属性来改变这个默认逻辑：

* alwaysRetainTaskState

  如果在Root Activity上设置这个属性为true，即使过了很长时间，Task中Activity也会回收。

* clearTaskOnLaunch

  如果在Root Activity上设置这个属性为true，即使只过了很短的时间，Task中的Activity也会被回收，仅仅保留Root Activity。

* finishOnTaskLaunch

  设置这个属性为true的Activity只会在此次的session中保留，并且它可以设置给Root Activity。



## ADB命令

可以用下面的ADB命令查看所有的Task和Task中的Activity。

```
adb shell dumpsys activity activities
```



## Reference

* [Understand Tasks and Back Stack](https://developer.android.com/guide/components/activities/tasks-and-back-stack)
* [Android Activity Launch Mode](https://android.jlelse.eu/android-activity-launch-mode-e0df1aa72242)

