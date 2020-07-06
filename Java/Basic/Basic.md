# Java基础

## 三大特性

* 封装

  隐藏对象的内部实现。使用`Public`，`Protected`，`Private`关键字或者默认包权限修饰方法和属性。

* 继承

  子类通过继承父类达到使用父类方法和属性的方式。

* 多态

  多态分为编译时多态和运行时多态：编译时多态指方法的重载；运行时多态指程序中定义的对象引用所指向的具体类型在运行期间才确定。



## 多态

### 三要素

* 继承
* 重写
* 向上转型

### 实现方式

* 接口
* 继承

### 静态绑定

在编译阶段就能够确定调用哪个方法的方式，我们叫做 静态绑定机制 。

除了被static 修饰的静态方法，所有被private 修饰的私有方法、被final 修饰的禁止子类覆盖的方法。JVM会采用静态绑定机制来顺利的调用这些方法。

### 动态绑定

在程序运行过程中，通过动态创建的对象的方法表来定位方法的方式，我们叫做 动态绑定机制 。
类对象方法的调用必须在运行过程中采用动态绑定机制。

首先，根据对象的声明类型（对象引用的类型）找到”合适”的方法。具体步骤如下：

* 如果能在声明类型中匹配到方法签名完全一样（参数类型一致）的方法，那么这个方法是最合适的。
* 在第1条不能满足的情况下，寻找可以“凑合”的方法。标准就是通过将参数类型进行自动转型之后再进行匹配。（根据继承树从叶子向上找）如果匹配到多个自动转型后的方法签名f(A)和f(B)，则用下面的标准来确定合适的方法：传递给f(A)方法的参数都可以传递给f(B)，则f(A)最合适。反之f(B)最合适 。
*  如果仍然在声明类型中找不到“合适”的方法，则编译阶段就无法通过。

```java
class A {
    // 方法1
    String show(A obj) {
        return ("A and A");
    }
    
    // 方法2
    String show(D obj) {
        return ("A and D");
    }
}

class B extends A {
    // 方法3
    String show(A obj) {
        return ("B and A");
    }
    
    // 方法4
    String show(B obj) {
        return ("B and B");
    }
}

class C extends B {}

class D extends B {}

class Solution {

    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new B();
        B b = new B();
        C c = new C();
        D d = new D();
        System.out.println("1--" + a1.show(b)); //A and A
        System.out.println("2--" + a1.show(c)); //A and A
        System.out.println("3--" + a1.show(d)); //A and D
        System.out.println("4--" + a2.show(b)); //B and A
        System.out.println("5--" + a2.show(c)); //B and A
        System.out.println("6--" + a2.show(d)); //A and D
        System.out.println("7--" + b.show(b));  //B and B
        System.out.println("8--" + b.show(c));  //B and B
        System.out.println("9--" + b.show(d));  //A and D
    }
}
```

* `A a1 = new A();`创建了一个`A`的引用，指向`A`对象，它具备调用方法1和方法2的能力。
  * `a1.show(b)`符合`A.show(A)`的签名，绑定给`A.show(A)`。
  * `a1.show(c)`符合`A.show(A)`的签名，绑定给`A.show(A)`。
  * `a1.show(d)`符合`A.show(D)`的签名，绑定给`A.show(D)`。
* `A a2 = new B();`创建了一个`A`的引用指向`B`对象，实现了多态。它具备调用方法1和方法2的能力，不具备调用方法3和方法4的能力。
  * `a2.show(b)`符合`A.show(A)`的签名，然后通过动态绑定，绑定给`B.show(A)`。
  * `a2.show(c)`符合`A.show(A)`的签名，然后通过动态绑定，绑定给`B.show(A)`。
  * `a2.show(d)`符合`A.show(D)`的签名，绑定给`A.show(D)`。

* `B b = new B();`创建了一个`B`的引用指向`B`对象。它具备调用方法3和方法4的能力，也具备调用父类方法2的能力，方法1被重写。
  * `b.show(b)`符合`B.show(B)`的签名，绑定给`B.show(B)`。
  * `b.show(c)`符合`B.show(B)`的签名，绑定给`B.show(B)`。
  * `b.show(d)`符合`A.show(D)`的签名，绑定给`A.show(D)`。



## String

### 不可变

* class声明为final
* 存储结构为private
* 不提供方法修改内部数据

### 不可变的意义

* 保证线程安全
* Hash映射作为key时数据安全

### 内存中的位置

* `String a = "abc";`

  `"abc"`这种声明方式的字符串在编译期已经创建好并存放到了常量池中。JVM会先在常量池中查找是否已经存在`"abc"`常量，如果找到了，就将它的引用返回给`String a`，如果没有找到，会创建一个并返回它的引用。

* `String a = new String("abc");`

  通过`new`创建的对象都储存在堆上，`a`的引用在栈上，`"abc"`字符串常量在常量池中。

#### 常见问题

`String a = new String("abc");`创建了几个对象？

一个`a`的引用在栈上，一个`new String()`对象在堆上，一个`"abc"`常量字符串在常量池中。

#### 实例

```java
public class StringPool {

    public static void main(String[] args) {
        String a = "abc";
        String b = "abc";
        String c = "a" + "b" + "c"; // 在编译期就被处理成"abc"
        System.out.println(a == b); // true
        System.out.println(a == c); // true

        String d = new String("abc");
        String e = new String("abc");
        System.out.println(d == e); // false

        d = d.intern();
        e = e.intern();
        System.out.println(c == d); // true
    }
}
```

### 拼接

#### StringBuilder, StringBuffer

使用`StringBuilder`和`StringBuffer`，后者线程安全，使用`synchronize`关键字。它们都继承自`AbstractStringBuilder` ，由它提供具体拼接操作。

内部实现使用的是一个char[]数组`value`，进行数组的拷贝。

```java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}
```

#### String.concat()

使用`String.concat()`方法，内部实现也是数组拷贝。并会重新创建一个String对象。

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}
```

#### 实例

```java
public class StringConcat {

    public static void main(String[] args) {
        String a = "a";
        String b = "b";
        String c = "c";

        String string1 = "a" + "b" + "c";
        String string2 = a + b + c; // 等价于 String string2 = new StringBuilder().append(a).append(b).append(c).toString();

        System.out.println(string1 == string2); // false

        String string3 = "abc";
        String string4 = string3.concat("def");

        System.out.println(string3 == string4); // false
    }
}
```



## Primitives，包装类型，装箱，拆箱

### Primitives

| 类型    | 大小  |
| ------- | ----- |
| boolean | 1bit  |
| byte    | 1byte |
| char    | 2byte |
| short   | 2byte |
| int     | 4byte |
| long    | 8byte |
| float   | 4byte |
| double  | 8byte |

### 与包装类型区别

* 传递方式
  * Primitives：按值传递
  * 包装类型：按引用传递（但是没有方法改变它的值）
* 存储位置
  * Primitives：栈
  * 包装类型：引用在栈上，对象在堆上

### 装箱

自动将基本数据类型转换为包装器类型。

```java
Integer i = 0; // Integer.valueOf()
```

```java
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];
    
    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }
    
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
}
```

### 拆箱

自动将包装器类型转换为基本数据类型。

```java
Integer i = 0;
int j = i; // Integer.intValue()
```

### 实例

```
public class Primitives {

    public static void main(String[] args) {
        Integer integer1 = 1;                     // Integer.valueOf(1) -> Integer.IntegerCache.cache[]
        Integer integer2 = 1;                     // Integer.valueOf(1) -> Integer.IntegerCache.cache[]
        System.out.println(integer1 == integer2); // true

        Integer integer3 = 128;                   // Integer.valueOf(128) -> new Integer(128)
        Integer integer4 = 128;                   // Integer.valueOf(128) -> new Integer(128)
        System.out.println(integer3 == integer4); // false

        Integer integer5 = new Integer(1);
        Integer integer6 = new Integer(1);
        System.out.println(integer5 == integer6); // false

        Integer integer7 = new Integer(1);
        int integer8 = 1;
        System.out.println(integer7 == integer8); // true unboxing
    }
}
```



## ==，equals()与hashcode()

### ==

* 基本数据类型

  基本数据类型用`==`比较的是数据的值。

* 引用类型

  引用类型用`==`进行比较的是他们在内存中的存放地址。
  所以，除非是同一个new出来的对象，他们的比较后的结果为true，否则比较后结果为false。
  对象是放在堆中的，栈中存放的是对象的引用（地址）。由此可见`==`是对栈中的值进行比较的。如果要比较堆中对象的内容是否相同，那么就要重写equals方法了。

### equals()

默认实现比较的是引用地址。

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

### hashcode()

默认实现保证每个对象的hash码不同（在对象的内存地址基础上经过特定算法返回一个hash码）。

```java
public native int hashCode();
```

### 原则

* a.equals(b)，那么两者hashcode相同。
* !a.equals(b)，两者hashcode可能相同，也可能不同。尽量不同将提高哈希表的效率。
* a.hashcode() == b.hashcode()，两者可能相同，也可能不同。
* a.hashcode() != b.hashcode()，两者不同。



## 内部类

###  静态内部类

```java
public class OutterClass {

    private static String staticString;
    private String string;

    private static void staticMethod() {
    }

    private void method() {
    }

    private static class StaticInnerClass {

        private StaticInnerClass() {
            // 无法获取Outter对象
            // OutterClass outerClass = OutterClass.this;

            // 无法调用非静态方法
            // method();

            // 可以调用静态方法
            staticMethod();

            // 无法获取非静态属性
            // System.out.println(string);

            // 可以获取静态属性
            System.out.println(staticString);
        }
    }


    public static void main(String[] args) {
        StaticInnerClass staticInnerClass = new OutterClass.StaticInnerClass();
    }
}
```

编译会生成对应的class，可以访问外部类的静态属性。

```java
class OutterClass$StaticInnerClass {
    private OutterClass$StaticInnerClass() {
        OutterClass.access$000();
        System.out.println(OutterClass.access$100());
    }
}
```

### 成员内部类

```java
public class OutterClass {

    private static String staticString;
    private String string;

    private static void staticMethod() {
    }

    private void method() {
    }

    private class InnerClass {

        private InnerClass() {
            // 可以获取Outter对象
            OutterClass outerClass = OutterClass.this;

            // 可以调用非静态方法
            method();

            // 可以调用静态方法
            staticMethod();

            // 可以获取非静态属性
            System.out.println(string);

            // 可以获取静态属性
            System.out.println(staticString);
        }
    }

    public static void main(String[] args) {
        OutterClass outterClass = new OutterClass();
        OutterClass.InnerClass innerClass = outterClass.new InnerClass();
    }
}
```

编译会生成对应的class，将外部类对象通过构造函数传递给匿名内部类。

```java
class OutterClass$InnerClass {
    private OutterClass$InnerClass(OutterClass var1) {
        this.this$0 = var1;
        OutterClass.access$200(var1);
        OutterClass.access$000();
        System.out.println(OutterClass.access$300(var1));
        System.out.println(OutterClass.access$100());
    }
}
```

### 匿名内部类

```java
public class OutterClass {

    private void testAnonymous() {
        String string = "a";
        new Object() {
            @Override
            public String toString() {
                return string;
            }
        };
    }
}
```

编译会生成对应的class，将外部类对象和参数通过构造函数传递给匿名内部类。

```java
class OutterClass$1 {
    OutterClass$1(OutterClass var1, String var2) {
        this.this$0 = var1;
        this.val$string = var2;
    }

    public String toString() {
        return this.val$string;
    }
}
```

### 局部内部类

```java
package zhaoyun.techstack.java;

public class OutterClass {

    private static String staticString;
    private String string;

    private void testLocalInner() {
        class LocalInner {
            void method() {
                // 可以访问非静态变量
                System.out.println(string);

                // 可以访问静态变量
                System.out.println(staticString);

                // 可以调用非静态方法
                OutterClass.this.method();

                // 可以调用静态方法
                OutterClass.staticMethod();
            }
        }
    }

    private static void testStaticLocalInner() {
        class LocalInner {
            void method() {
                // 无法访问非静态变量
                // System.out.println(string);

                // 可以访问静态变量
                System.out.println(staticString);

                // 无法调用非静态方法
                // OutterClass.this.method();

                // 可以调用静态方法
                OutterClass.staticMethod();
            }
        }
    }
}

```

在非静态方法中声明的局部内部类，编译会生成对应的class，将外部类对象和参数通过构造函数传递给局部内部类。

```java
class OutterClass$1LocalInner {
    OutterClass$1LocalInner(OutterClass var1) {
        this.this$0 = var1;
    }

    void method() {
        System.out.println(OutterClass.access$300(this.this$0));
        System.out.println(OutterClass.access$100());
        OutterClass.access$200(this.this$0);
        OutterClass.access$000();
    }
}
```

在静态方法中声明的局部内部类，编译会生成对应的class，可以访问外部类的静态属性和方法。

```java
class OutterClass$2LocalInner {
    OutterClass$2LocalInner() {
    }

    void method() {
        System.out.println(OutterClass.access$100());
        OutterClass.access$000();
    }
}
```



## 反射

在运行时获取类的方法、属性、构造函数等信息，并进行方法的调用、属性的修改和对象的构造，这种动态的机制称为反射。

类的信息保存在`Class`对象中。

### Class对象获取方法

* `类名.class`
* `Object.getClass()`
* `Class.forName()`

### Class

* `getSuperClass()`：获取父类
* `getGenericSuperclass()`：返回继承的父类和泛型类型信息。返回值为`ParameterizedType`，`ParameterizedType.getActualTypeArguments()`可以获取泛型`class`对象。
* `getGenericInterfaces()`：返回实现接口和接口泛型信息。返回值为`ParameterizedType`数组，`ParameterizedType.getActualTypeArguments()`可以获取泛型`class`对象。

### Field

* `Class.getFields()`：所有此类及父类定义的`public`属性。
* `Class.getDeclaredFields()`：所有此类定义的属性。内部类还会包括`this$0`。

### Method

* `Class.getMethods()`：所有此类及父类定义的`public`方法。
* `Class.getDeclaredMethods()`：所有此类定义的方法。

### Constructor

* `Class.getConstructors()`：所有此类定义的`public`构造方法。
* `Class.getDeclaredConstructors()`：所有此类定义的`public`构造方法。



## 泛型



## 注解



## 代理
