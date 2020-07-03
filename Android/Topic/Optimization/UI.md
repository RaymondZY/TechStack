# Optimization - UI

## OverDraw

### 检测方式

### 优化方法

* 移除`window`的`background`，移布局中除非必要的`background`。
* 使用`ConstartLayout`扁平化布局。

* 自定义`View`使用`canvas.clipRect()`方法进行剪裁。使用`canvas.quickreject()`判断是否相交。

  ```java
  canvas.save();
  canvas.clipRect();
  canvas.drawBitmap();
  canvas.restore();
  ```



## 布局加载优化

* 使用代码方式构造布局。
  * 使用`X2C`库将`xml`转为代码。
* 使用`AsyncLayoutInflator`异步加载。
  * 不能有依赖主线程的操作。



## XML标签优化

* `<include>`
* `<merge>`
* `<ViewStub>`



## ListView优化

* 使用`ViewHolder`和`Tag`。



## RecyclerView优化

* 避免重复在`onBindViewHolder()`方法中新建变量。
* 图片的加载在`RecyclerView`滑动停止时在进行。