# Crash

## ANR

**InputDispatching Timeout**：5秒内无法响应屏幕触摸事件或键盘输入事件

**BroadcastQueue Timeout** ：在执行前台广播（BroadcastReceiver）的`onReceive()`函数时10秒没有处理完成，后台为60秒。

**Service Timeout** ：前台服务20秒内，后台服务在200秒内没有执行完毕。

**ContentProvider Timeout** ：ContentProvider的publish在10s内没进行完。



## Log

从/data/anr/里获取anr详细log进行分析。

手机root以后使用adb pull或者使用adb bugreport打包以后再pull。