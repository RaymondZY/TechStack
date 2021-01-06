# Screen orientation

## api

`Activity.setRequestedOrientation` 

`Activity.getRequestedOrientation` 



## orientations

定义在`ActivityInfo`中。

可以参考官方文档：https://developer.android.com/reference/android/R.attr#screenOrientation

* `SCREEN_ORIENTATION_UNSET -2`  

  用于标记未设置过屏幕方向，一般会和`SCREEN_ORIENTATION_UNSPECIFIED`的处理方式一致。和`SCREEN_ORIENTATION_UNSPECIFIED`不同的是，`SCREEN_ORIENTATION_UNSPECIFIED`用于表示用户显式设置了不指定屏幕方向，这个值用于表示没有设置过。

* `SCREEN_ORIENTATION_UNSPECIFIED -1` 

  默认的屏幕方向，让系统自己决定。

  * 关闭系统屏幕旋转

    Portrait

  * 开启系统屏幕旋转

    Portrait，Landscape，Reverse Landscape，无法切换成Reverse Portrait

* `SCREEN_ORIENTATION_LANDSCAPE 0` 

  指定为Landscape。

  * 关闭系统屏幕旋转

    Landscape

  * 开启系统屏幕旋转

    Landscape，无法通过旋转成为Reverse Landscape

* `SCREEN_ORIENTATION_PORTRAIT 1` 

  指定为Portrait。无视系统屏幕旋转开关。

* `SCREEN_ORIENTATION_USER 2`

  指定为用户当前期望的屏幕方向。

  行为和默认一致。

* `SCREEN_ORIENTATION_BEHIND 3`

  让Activity保持和whatever is behind this activity一致。

* `SCREEN_ORIENTATION_SENSOR 4`

  即使没有开启系统屏幕旋转，也可以转动屏幕。

  但是无法从Portrait直接180度旋转到Reverse Portrait。

* `SCREEN_ORIENTATION_NOSENSOR` 5

  即使开启屏幕旋转，也无法转动屏幕。

* `SCREEN_ORIENTATION_SENSOR_LANDSCAPE` 6

  指定为Landscape，并且可以使用传感器决定屏幕方向，即使没有开启系统屏幕旋转。

  实际使用中，可以切换到Reverse Landscape。

* `SCREEN_ORIENTATION_SENSOR_PORTRAIT` 7

  指定为Portrait，并且可以使用传感器决定屏幕方向，即使没有开启系统屏幕旋转。

  实际使用中，可以切换到Reverse Portrait。

* `SCREEN_ORIENTATION_REVERSE_LANDSCAPE` 8

  指定为Reverse Landscape。无视系统屏幕旋转开关。

* `SCREEN_ORIENTATION_REVERSE_PORTRAIT` 9

  指定为Reverse Portrait。无视系统屏幕旋转开关。

* `SCREEN_ORIENTATION_FULL_SENSOR` 10

  类似于`SCREEN_ORIENTATION_SENSOR`，即使没有开启系统屏幕旋转，也可以跟随传感器旋转屏幕，不同于`SCREEN_ORIENTATION_SENSOR`的是可以旋转到Reverse Portrait。

* `SCREEN_ORIENTATION_USER_LANDSCAPE` 11

  指定为Landscape，如果用户开启了系统屏幕旋转，可以使用传感器决定屏幕方向。

  实际使用中，可以切换到Reverse Landscape。

* `SCREEN_ORIENTATION_USER_PORTRAIT` 12

  指定为Portrait，如果用户开启了系统屏幕旋转，可以使用传感器决定屏幕方向。

  实际使用中，无法切换到Reverse Landscape。

* `SCREEN_ORIENTATION_FULL_USER` 13

  如果用户开启了系统屏幕旋转，可以使用传感器决定屏幕方向，并且可以旋转到Reverse Portrait。

* `SCREEN_ORIENTATION_LOCKED` 14

  锁定当前的屏幕方向。



## configChanges

Activity标签上需要添加属性：

`android:configChanges="orientation|screenSize|screenLayout"`

参考官方文档：https://developer.android.com/guide/topics/manifest/activity-element#config