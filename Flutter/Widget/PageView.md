# PageView

## 基本使用

通过`PageView.Builder()`构造，传入`itemBuilder`和`itemCount`参数。

```dart
PageView.builder(
  itemCount: 100,
  itemBuilder: (context, index) {
    return Text("$index");
  },
),
```



## Controller

在`PageView.Builder()`中传入`Controller`参数控制`PageView`的一些页面特性。

```dart
controller: PageController(
  initialPage: 50,
  viewportFraction: 0.8,
),
```

* `initialPage`：初始显示的页面
* `viewportFraction`：页面宽度占比



## onPageChanged

在`PageView.Builder()`中传入`onPageChanged`参数监听页面切换回调。

```dart
PageView.builder(
  itemCount: 100,
  itemBuilder: (context, index) {
    return Text("$index");
  },
  onPageChanged: (index) {
    print("zhaoyun ===> onPageChanged : $index");
  },
),
```

