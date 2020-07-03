# Optimization - Memory

## 内存抖动

短时间内多次内存分配和释放，导致频繁GC。

* 频繁调用的方法不要创建对象。例如：`onDraw`，`onLayout()`，`onMeasure()`，`Handler`。

* 使用对象池。



## 内存溢出

使用内存过多，可能是内存泄漏导致的，也可能是代码未优化导致的。

可以做的代码优化有：

* 在`AndroidManifest.xml`中`<Application>`标签使用属性`android:largeHeap=true`，申请更大的堆内存。

* `Bitmap`记载时进行缩放。
* `Bitmap`使用`LRUCache`。
* 使用`onTrimMemory()`方法回收资源。
* 使用`SparseArray`，`ArrayMap`等高效的数据结构。



## 内存泄漏

一个程序不被使用的对象依旧存活在内存中，无法被回收。

本质是对象引用持有者的生命周期超过了对象本身的生命周期。

常见原因：

* `Cursor`，`BroadcastReceiver`等资源未关闭和反注册。

* 非静态的内部类持有外部对象。异步线程或者静态属性持有它的引用，导致生命周期被延长，而发生内存泄漏。

### Android Profiler

在**Memory Profiler**中的使用**Dump Heap**，**Arrange by package**功能进行查看。

### Eclipse MAT

使用`$ANDROID_SDK$/platform-tools/hprof-conv.ext`对**Memory Profiler**输出的`.hprof`文件进行转换，然后再使用**Eclipse MAT**工具进行查看。

功能与**Memory Profiler**类似，可以查看到Heap中的对象和被引用情况。