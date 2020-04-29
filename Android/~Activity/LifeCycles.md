# Activity Lifecycles

## 生命周期

Activity的生命周期如下图：

![](./images/activity_lifecycle.png)



## 常见问题

* 横竖屏切换的时候，Activity 各种情况下的生命周期

  * 默认情况下：

    ```
    onPause()
    onStop()
    onDestroy()
    onCreate()
    onStart()
    onResume()
    ```

  * AndroidManifest.xml设置configChanges后：

    ```xml
    <activity 
    	android:name=".SecondActivity"
    	android:configChanges="orientation">
    </activity>
    ```

    ```
    onConfigurationChanged()
    ```

* 前台切换到后台，然后再回到前台，Activity生命周期回调方法

  * 切换到后台

    ```
    onPause()
    onStop()
    ```

  * 回到前台

    ```
    onRestart()
    onStart()
    onResume()
    ```

* 弹出Dialog，生命值周期回调方法

  ```
  onCreateDialog()
  ```

  `Activity`并不会发生生命周期变化

* Activity跳转时生命周期

  * 普通的Activity

    ```
    FirstActivity: onPause()
    								SecondActivity: onCreate()
    								SecondActivity: onStart()
    								SecondActivity: onResume()
    FirstActivity: onStop()
    ```

    在第一个Activity调用`onPause()`后，界面仍处于可见状态，需要等待第二个Activity界面完全显示。在第二个Activity界面完全创建并调用`onResume()`后，才会执行第一个Activity的`onStop()`。因此，和UI相关的回收操作应该放在`onStop()`中，而不是`onPause()`中。

    ```
    								SecondActivity: onPause()
    FirstActivity: onActivityResult()
    FirstActivity: onRestart()
    FirstActivity: onStart()
    FirstActivity: onResume()
    								SecondActivity: onStop()
    								SecondActivity: onDestroy()
    ```

    同样的道理，在第二个Activity调用`onPause()`后，需要等待第一个activity的界面完全显示并调用`onResume()`后，才会执行第二个activity的`onStop()`和`onDestroy()`。

  * 透明Activity

    ```
    FirstActivity: onPause()
    								TransparentActivity: onCreate()
    								TransparentActivity: onStart()
    								TransparentActivity: onResume()
    ```

    ```
    								TransparentActivity: onPause()
    FirstActivity: onActivityResult()
    FirstActivity: onResume()
    								TransparentActivity: onStop()
    								TransparentActivity: onDestroy()
    ```

  * DialogActivity

    ```
    FirstActivity: onPause()
    								DialogActivity: onCreate()
    								DialogActivity: onStart()
    								DialogActivity: onResume()
    ```

    ```
    								DialogActivity: onPause()
    FirstActivity: onActivityResult()
    FirstActivity: onResume()
    								DialogActivity: onStop()
    								DialogActivity: onDestroy()
    ```

