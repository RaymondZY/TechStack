# IndexedStack

## 基本使用

`IndexedStack`拥有一组children，通过指定`index`属性确定具体哪个child被显示。

它一次只显示一个child，但是保存了所有children的状态。

使用`alignment`属性来制定对齐方式。

```dart
IndexedStack(
  index: _currentIndex,
  alignment: Alignment.center,
  children: [],
);
```

可以用它来实现首页Tab切换。