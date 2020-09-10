# GridView

## 创建

### GridView()

使用默认构造函数创造，在`children`属性中添加所有希望展示的Widget。

```dart
GridView(
  children: children,
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,
  ),
);
```

### GridView.builder()

可以使用`GridView.builder()`传入构造函数进行构造。

```dart
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,
  ),
  itemBuilder: (context, index) {
    return Container(
      color: Colors.green[100 * (index % 4)],
      alignment: Alignment.center,
      child: ListTile(
        title: Text("GridView $index"),
      ),
    );
  },
);
```

