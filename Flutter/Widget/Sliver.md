# Sliver

## CustomScrollView

`CustomScrollView`可以解决多个滚动控件配合使用的问题。

需要传入`slivers`参数，表示子sliver widget。

一般一起使用的有`SliverAppBar`，`SliverList`，`SliverGrid`，`SliverToBoxAdapter`等。

和`ListView`一样，也可以配置`ScrollController`，`ScrollPhysics`等参数。



## SliverAppBar



## SliverList

类似于`ListView`，用于展示一组子widget列表。

和`ListView`有所不同，并没有`physics`，`controller`等控制参数，只能设置子widget。

普通的`ListView`不能和`CustomScrollView`一起使用。

需要传入`delegate`参数构造`SliverList`中的widget。

使用`SliverChildListDelegate`，传入数组形式的子widget。

```dart
SliverList(
  delegate: SliverChildListDelegate([
    Container(
      color: Colors.purple[100],
      child: ListTile(
        title: Text("SliverChildListDelegate 0"),
      ),
    ),
    Container(
      color: Colors.purple[200],
      child: ListTile(
        title: Text("SliverChildListDelegate 1"),
      ),
    ),
    Container(
      color: Colors.purple[300],
      child: ListTile(
        title: Text("SliverChildListDelegate 2"),
      ),
    ),
  ]),
),
```

或者使用`SliverChildBuilderDelegate`传入builder形式的构造方法。

```dart
SliverList(
  delegate: SliverChildBuilderDelegate(
    (context, index) {
      return Container(
        color: Colors.green[100 * (index % 4 + 1)],
        child: ListTile(
          title: Text("SliverChildBuilderDelegate $index"),
        ),
      );
    },
    childCount: 50,
  ),
),
```



## SliverGrid

类似于`GridView`用于展示一组表格。

普通的`GridView`无法和`CustomScrollView`一起使用。

需要传入`delegate`参数构造`SliverGrid`中的widget。

需要传入`gridDelegate`传入表格列的定义方式。

```dart
dSliverGrid(
  delegate: SliverChildBuilderDelegate((context, index) {
    return Container(
      color: Colors.green[100 * (index % 3 + 1)],
      alignment: Alignment.center,
      child: ListTile(
        title: Text("SliverGrid $index"),
      ),
    );
  }),
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,
  ),
),
```



## SliverPadding

类似于`Padding`，`SliverPadding`用于包裹sliver widget。

sliver widget不能使用`Padding`包裹。

```dart
SliverPadding(
  padding: EdgeInsets.all(16.0),
  sliver: SliverList(
    delegate: SliverChildBuilderDelegate(
      (context, index) {
        return Container(
          color: Colors.green[100 * (index % 4 + 1)],
          child: ListTile(
            title: Text("SliverList $index"),
          ),
        );
      },
      childCount: 50,
    ),
  ),
),
```



## SliverToBoxAdapter

`SliverToBoxAdapter`可以把一个普通的box widget转成一个sliver widget。

普通的box widget是不能直接在`CustomScrollView`中使用的。

```dart
SliverToBoxAdapter(
  child: Container(
    padding: EdgeInsets.all(32),
    alignment: Alignment.center,
    color: Colors.purple[200],
    child: Text("SliverToBoxAdapter"),
  ),
),
```



## SliverFillRemaining

`SliverFillRemaining`可以用于填充`CustomScrollView`剩余的部分。

```dart
CustomScrollView(
  slivers: [
    SliverList(
      delegate: SliverChildListDelegate([
        Container(
          color: Colors.green[100],
          child: ListTile(
            title: Text("SliverList 0"),
          ),
        ),
        Container(
          color: Colors.green[200],
          child: ListTile(
            title: Text("SliverList 1"),
          ),
        ),
        Container(
          color: Colors.green[300],
          child: ListTile(
            title: Text("SliverList 2"),
          ),
        )
      ]),
    ),
    SliverFillRemaining(
      child: Container(
        color: Colors.purple,
      ),
      hasScrollBody: false,
      fillOverscroll: true,
    ),
  ],
);
```

