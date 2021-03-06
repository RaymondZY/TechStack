# JVM

## 运行时数据区域

* 方法区：加载的类的结构信息，常量池，静态变量。
* Java虚拟机栈：每个线程都有一个。记录方法调用的状态。栈帧。
* 本地方法栈：Native方法栈。
* Java堆：对象的实例。GC的位置。
* 程序计数器：知道下一条指令是哪一条。

**方法区**和**Java堆**是共享区域，其它的是线程私有的。



## 引用

* 强引用：通过`new`创建的对象。不释放引用，不会被回收。创建对象过多会引起`OutOfMemoryError`异常。
* 软引用：当内存不够时，会回收软引用。如果回收了软引用还是没有足够的内存，会引起`OutOfMemoryError`异常。
* 弱引用：当内存回收时，就会被回收的对象。不管当前内存是否足够。
* 虚引用：如果一个对象持有虚引用，在任何时候都可能被回收，只是在回收的时候会收到一个系统通知。



## 类加载

### 双亲委派机制

`ClassLoader.loadClass()`调用父级`ClassLoder.loadClass()`尝试从父级类加载器进行加载。如果父级加载不了，再调用自己的`ClassLoader.findClass()`方法进行加载。

### 流程

* 加载

  从流中加载类信息到方法区。

* 校验

  校验文件格式、元数据、字节码、符号引用。

* 准备

  为类变量分配内存，设置变量的初始值。

* 解析

  常量池中的符号引用转为直接引用。

* 初始化

  初始化类变量，静态方法块。合并成`<clinit>()`方法。