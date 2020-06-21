# Context

## 定义

Context提供了关于应用环境全局信息的接口。它是一个抽象类，它的执行被Android系统所提供。它允许获取以应用为特征的资源和类型，是一个统领一些资源（应用程序环境变量等）的上下文。就是说，它描述一个应用程序环境的信息（即上下文）；是一个抽象类，Android提供了该抽象类的具体实现类；通过它我们可以获取应用程序的资源和类（包括应用级别操作，如启动Activity，发广播，接受Intent等）。



## 继承关系

`Context`的子类：`ContextWrapper`。

`ContextWrapper`的子类：`ContextThemeWrapper`，`Application`，`Service`。

`ContextThemeWrapper`的子类：`Activity`。



## Application Context与Activity Context的区别

* 生命周期不同。Application Context生命周期与应用进程相同，Activity Context与Activity生命周期一致。
* 父类不同。Application Context继承自`ContextWrapper`，Activity Context继承自`ContextThemeWrapper`，封装了对Theme主题的支持。

