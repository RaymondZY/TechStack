# Flutter get started

## 结构

* 入口

    ```dart
    void main() => runApp(MyApp());
    ```
    
    * `=>`：一行函数调用的简写方式。
    
* MyApp

    ```dart
    class MyApp extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        return MaterialApp(
          title: 'Welcome to Flutter',
          home: Scaffold(
            appBar: AppBar(
              title: Text('Welcome to Flutter'),
            ),
            body: Center(
              child: Text('Really?? nigga?'),
            ),
          ),
        );
      }
    }
    ```

    * `MyApp`：继承自`StatelessWidget`，flutter中所有东西都是`Widget`。
    * `MaterialApp`：提供material design样式的app
      * `title`：任务栏中的标题
      * `home`：页面视图树
    * `Scaffold`：脚手架
      * `appBar`：标题栏
      * `body`：页面内容
    * `AppBar`：标题栏
      * `title`：标题栏文字



## Stateful widget

* Stateless widget：属性不能变更

* Stateful widget：属性可以变更

  Stateful widget内部保存一个State对象，Stateful widget本身是不可变的，会被销毁和重新创建，但是内部保存的State对象是可以重用的。