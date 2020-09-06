# LayoutBuilder

## 基本使用

使用`LayoutBuilder`可以定义Widget在不同Constraints下具体使用的控件。

也可以通过`LayoutBuilder`获取到具体的Constraints信息。

```dart
LayoutBuilder(
  builder: (BuildContext context, BoxConstraints constraints) {
    print("constraints = $constraints");
    return constraints.maxHeight >= 200
      ? Container(
      color: Colors.red,
    )
      : Container(
        color: Colors.yellow,
      );
  },
);
```