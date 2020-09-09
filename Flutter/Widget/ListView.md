# ListView

## 创建

### ListView()

使用默认构造函数创造，在`children`属性中添加所有希望展示的Widget。

```dart
class ListViewConstructorPlayground extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    var colors = <Color>[
      Colors.purple,
      Colors.orange,
      Colors.green,
      Colors.red,
    ];
    var children = <Widget>[];
    for (int i = 0; i < 100; i++) {
      children.add(Container(
        height: 50,
        color: colors[i % colors.length],
      ));
    }
    return ListView(children: children);
  }
}
```

`ListView`并不会渲染所有的Widget，而是会根据`cacheExtent`属性渲染超出屏幕的部分。

`cacheExtent`的值默认是250，定义在`RenderAbstractViewport`中。

```dart
abstract class RenderAbstractViewport extends RenderObject {
	static const double defaultCacheExtent = 250.0;
}
```



### ListView.builder()

如果需要展示的widget很多，可以使用`ListView.Builder`方法进行构造。

```dart
ListView.builder(
  itemBuilder: (context, index) {
    return Container(
      color: Colors.green[100 * (index % 4 + 1)],
      padding: EdgeInsets.all(16),
      child: Text("ListView $index"),
    );
  },
  itemCount: 100,
);
```

* `itemCount`参数并不是一个必要的参数，可以不指定，那么列表会是无限长度



### ListView.separator()

`ListView.separator()`方法可以很方便地构造一个有分割线的列表。

```dart
class ListViewSeparatorPlayground extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView.separated(
      itemBuilder: (context, index) {
        return ListTile(
          title: Text("ListView $index"),
        );
      },
      separatorBuilder: (context, index) => Divider(),
      itemCount: 100,
    );
  }
}
```

* `separatorBuilder`参数为分割线的构造方法



## ScrollController

使用`ScrollController`可以监听和控制`ListView`的滚动。

一般需要在`initState()`方法中对`ScrollController`进行初始化。

并在`dispose()`方法中调用`ScrollController.dispose()`进行资源的释放。

初始化：

```dart
@override
void initState() {
  _controller.addListener(() {
    // 监听到列表滚动
  });
  super.initState();
}
```

释放：

```dart
@override
void dispose() {
  _controller.dispose();
  super.dispose();
}
```

`ScrollController`提供了`ScrollPosition`属性可以获取当前滚动的坐标，和最大的滚动坐标等关键数据。

依此可以进行加载更多功能的实现。

```dart
_controller.addListener(() {
  var currentPosition = _controller.position.pixels;
  var maxPosition = _controller.position.maxScrollExtent;
  if (currentPosition > maxPosition - 500) {
    _loadMore();
  }
});
```

* `_controller.position`获取的为`ScrollPosition`对象
* `_controller.position.pixels`获得的为当前滚动偏移量
* `_controller.position.maxScrollExtent`获取的为最大的滚动偏移量

`ScrollController`还提供了`jump()`，`animateTo()`等方法，可以用代码控制列表的滚动。



## ScrollPhysics

`ListView`的构造函数中可以传入`physics`参数控制列表滑动的特性。

* `NeverScrollableScrollPhysics`：可以使列表不能滚动
* `ClampingScrollPhysics`：可以使列表滚动到边界时立即停止，在Android设备上可以触发Glow效果。
* `BouncingScrollPhysics`：可以使列表滚动到边界时触发回弹。