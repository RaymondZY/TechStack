# Dart

## 基础

### 类

可以通过`class`定义类，用`abstract class`定义抽象类，用`extends`关键字定义继承关系。

```dart
abstract class Shape {
  num get area;
}

class Circle extends Shape {

  num radius;

  Circle(this.radius);

  @override
  num get area => pi * pow(radius, 2);
}
```

例如上面的例子中，抽象类`Shape`中定义了`num area`属性和该属性需要有一个`get`方法。`Circle`实现了`Shape`，并且实现了`area`属性的`get`方法。

可以使用`..`进行瀑布式调用，它会在调用完后返回对象。

```dart
class CascadesPlayground {
  
  int num;

  void printA() => print("A");

  void printB() => print("B");
}

void main() {
  var cascadesPlayground = CascadesPlayground();
  cascadesPlayground
    ..num = 1
    ..printA()
    ..printB();
}
```

### 接口

`dart`中没有`interface`关键字，所有类都定义了一个接口。

使用`implements`关键字实现接口。

`dart`中的接口需要实现类中的所有能力。

```dart
abstract class NamePrintable {
  
  void printName();
}

class Person implements NamePrintable {
  
  String name;
  
  Person(this.name);
  
  @override
  void printName() {
    print(name);
  }
}

class PersonMock implements Person {

  @override
  String name;

  @override
  void printName() {
    
  }
}
```

例如在这个例子中，`Person`需要实现`NamePrintable`中的抽象方法`printName()`，`PersonMock`需要实现`Person`提供的`name`属性和`printName()方法。`

因此，如果要实现类似`java`的接口功能，最好是使用`abstract class`定义。

### 函数

在`dart`中可以把函数定义成一个变量。并使用这个变量进行函数的执行。

```dart
void main() {
  var func;

  // 最繁琐的写法，明确了index的类型，书写了=>和{}代码block
  func = (int item) => {print("this is $item")};
  for (var i in [1, 2, 3]) {
    func(i);
  }

  // 省略掉index类型
  func = (item) => {print("this is $item")};

  // item类型为int可以进行调用
  for (var i in [1, 2, 3]) {
    func(i);
  }
  // item类型为String也可以进行调用
  for (var i in ["a", "b", "c"]) {
    func(i);
  }

  // 可以省略=>
  func = (item) {
    print("this is $item");
  };

  // 如果{}代码block只有一行，可以省略{}括号
  func = (item) => print("this is $item");

  // 使用匿名函数的方式进行调用
  for (var i in ["a", "b", "c"]) {
    ((item) => print("this is $item"))(i);
  }
}
```

### 可空性

使用`??` ，`??=`和`?.`对可空的对象进行处理：

```dart
String foo = 'a string';
String bar; // Unassigned objects are null by default.

// Substitute an operator that makes 'a string' be assigned to baz.
String baz = foo ?? bar;

void updateSomeVars() {
  // Substitute an operator that makes 'a string' be assigned to bar.
  bar ??= 'a string';
}

String upperCaseIt(String str) {
  return str?.toUpperCase();
}
```

### getter setter

可以向类中添加`get`和`set`自定义控制变量的读取和赋值。

```dart
class ShoppingCart {
  List<double> _prices = [];

  double get total {
    double sum = 0;
    for (var price in _prices) {
      sum += price;
    }
    return sum;
  }

  set prices(List<double> prices) {
    if (prices.any((price) => price < 0)) {
      throw Exception("price can't be negative");
    }
    _prices = prices;
  }
}

void main() {
  var shoppingCart = ShoppingCart();
  /*
  Exception
  shoppingCart.prices = [-1.0, 9.0];
  print("total = ${shoppingCart.total}");
   */
  shoppingCart.prices = [1.0, 9.0];
  print("total = ${shoppingCart.total}");
}
```

### 可选参数

可以使用`{}`把方法的可选参数包括起来。

```dart
class MyDataObject {
  final int anInt;
  final String aString;
  final double aDouble;

  MyDataObject({
     this.anInt = 1,
     this.aString = 'Old!',
     this.aDouble = 2.0,
  });

  MyDataObject copyWith({int newInt, String newString, double newDouble}) {
    return MyDataObject(
      anInt: newInt ?? this.anInt,
      aString: newString ?? this.aString,
      aDouble: newDouble ?? this.aDouble,
    );
  }
}
```

### 带命名的构造方法

可以使用带命名的构造方法，进行自定义的对象构造。

```dart
class Point {
  double x, y;

  Point(this.x, this.y);

  Point.origin() {
    x = 0;
    y = 0;
  }
}
```

### 构造方法初始化语句

可以在构造方法代码block执行前，先执行一些赋值语句或者断言语句。

```dart
class Person {
  String name;

  Person.createFromIndex(int index)
      : name = "Person$index",
        assert(index >= 0) {
    print("name is $name");
  }
}

void main() {
  /*
  assert failed
  var personInvalid = Person.createFromIndex(-1);
   */
  var person0 = Person.createFromIndex(0);
  var person1 = Person.createFromIndex(1);
}
```

### 构造方法factory

可以使用`factory`方法控制创建的对象。与命名构造方法不同，命名构造方法一定会返回一个对应类的对象，并且不用显式调用`return`语句，`factory`方法可以返回对应类、子类、甚至是`null`。

```dart
class Square extends Shape {}

class Circle extends Shape {}

class Shape {
  Shape();

  factory Shape.fromTypeName(String typeName) {
    if (typeName == 'square') return Square();
    if (typeName == 'circle') return Circle();

    print('I don\'t recognize $typeName');
    return null;
  }
}
```

### 重定向的构造方法

可以使用重定向的构造方法把构造逻辑转移到默认的构造方法。

```dart
class Person {
  String name;
  int age;
  bool married;

  Person({this.name, this.age, this.married});

  Person.married(String name, int age)
      : this(name: name, age: age, married: true);
}
```

### const构造方法

使用`const`构造方法可以在编译期保证对象内部的所有变量都是`final`的。

```dart
class ImmutablePoint {
  const ImmutablePoint(this.x, this.y);

  final int x;
  final int y;

  static const ImmutablePoint origin = ImmutablePoint(0, 0);
}
```

### Exception

使用`try`，`on`，`catch`，`finally`进行异常捕获控制。

`dart`可以`throw`任意对象。

自定义异常`implements Exception`。

```dart
try {
  playWithException();
} on CustomException catch (e) {
  print("caught exception $e");
} finally {
  print("finally");
}

class CustomException implements Exception {
  String message;

  CustomException(this.message);
}
```



## Iterable Collections

* where
* first
* last
* firstWhere
* any
* every
* takeWhere
* skipWhere
* map



## 异步编程

### async await future

使用`Future<T>`，`async`，`await`将方法改为异步 。

```dart
Future<void> printWithDelay() async {
  print("wait for it");
  await Future.delayed(Duration(milliseconds: 1000));
  print("here it comes");
  await Future.delayed(Duration(milliseconds: 1000));
  print("almost there");
  await Future.delayed(Duration(milliseconds: 1000));
  print("boom");
}
```

例如上述的例子中，`print("wait for it")`仍然会立刻执行，`async`方法在碰到`await`之前的语句都会同步执行。

### streams

使用`async*`，`yield`生产`stream`。

使用`await for`消费`stream`。

```dart
void main() {
  playWithStream();
}

Future<void> playWithStream() async {
  // 1
  print("playWithStream()");
  var countDownStream = generateCountDownStream(10);
  var sum = await calcSum(countDownStream);
  print(sum);
}

Stream<int> generateCountDownStream(int max) async* {
  // 3
  print("generateCountDownStream()");
  for (var i = max; i >= 0; i--) {
    await Future.delayed(Duration(seconds: 1));
    print("yield $i"); // 4 6 8 ..
    yield i;
  }
}

Future<int> calcSum(Stream<int> countDownStream) async {
  // 2
  print("calcSum()");
  int sum = 0;
  await for (var num in countDownStream) {
    print("sum $num"); // 5 7 9 ..
    sum += num;
  }
  return sum;
}
```

上述例子中，代码执行顺序如注释标记，`await for`消费总会等到`yield`生产之后。

如果不执行`calcSum`订阅`stream`，那么`stream`中的生产代码将不会被执行。

#### 操作

`stream`上也定义了类似于`iterable`的用于操作的方法。

```dart
Future<T> get first;
Future<bool> get isEmpty;
Future<T> get last;
Future<int> get length;
Future<T> get single;
Future<bool> any(bool Function(T element) test);
Future<bool> contains(Object needle);
Future<E> drain<E>([E futureValue]);
Future<T> elementAt(int index);
Future<bool> every(bool Function(T element) test);
Future<T> firstWhere(bool Function(T element) test, {T Function() orElse});
Future<S> fold<S>(S initialValue, S Function(S previous, T element) combine);
Future forEach(void Function(T element) action);
Future<String> join([String separator = ""]);
Future<T> lastWhere(bool Function(T element) test, {T Function() orElse});
Future pipe(StreamConsumer<T> streamConsumer);
Future<T> reduce(T Function(T previous, T element) combine);
Future<T> singleWhere(bool Function(T element) test, {T Function() orElse});
Future<List<T>> toList();
Future<Set<T>> toSet();
```

`stream`上也定义了一些进行变换的方法：

```dart
Stream<R> cast<R>();
Stream<S> expand<S>(Iterable<S> Function(T element) convert);
Stream<S> map<S>(S Function(T event) convert);
Stream<T> skip(int count);
Stream<T> skipWhile(bool Function(T element) test);
Stream<T> take(int count);
Stream<T> takeWhile(bool Function(T element) test);
Stream<T> where(bool Function(T event) test);
```

