# AspectRatio

## 基本使用

可以指定子Widget的宽高比。

```dart
AspectRatio(
  aspectRatio: 4 / 3,
  child: child,
),
```

子Widget自己设置的宽高可能会失效。

```dart
Container(
  color: Colors.grey,
  width: 400,
  height: 400,
  child: Center(
    child: AspectRatio(
      aspectRatio: 4 / 3,
      child: Container(
        color: Colors.red,
        // 不会产生效果
        width: 100,
        // 不会产生效果
        height: 100,
      ),
    ),
  ),
);
```

`AspectRatio`会根据从父Widget收到的宽高Constraints，先让宽尽量大，然后再根据宽高比计算高的大小。

如果此时高度没有超出父Widget高度的限制，那么宽高就确定了。

如果此时高度超出了父Widget限制的最大高度，那么让高度为最大，重新通过宽高比计算宽度的大小。