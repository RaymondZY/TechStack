# Reference
* [Android自定义view之measure、layout、draw三大流程](https://blog.csdn.net/zuguorui/article/details/70160323)

# 入口
`ViewTreeImpl.performTraversals()`，先后执行下面的方法：
```java
ViewTreeImpl.performMeasure()
ViewTreeImpl.performLayout()
ViewTreeImpl.performDraw()
```
进而一步一步调用到：
```java
View.measure()
View.layout()
View.draw()
```

# Measure
核心思路：
* `MeasureSpec`对象包装了parent对child的大小的限制要求。
* parent调用child的`View.measure()`方法让child自己计算measure大小。同时使用`MeasureSpec`传入相应的限制要求。
    * child的`View.measure()`方法会调用`View.onMeasure()`，并在里面实现大小的计算。
    * child的`View.measure()`方法会调用`View.measure()`，递归进行子View的Measure。

源码关键路径：
```java
ViewRootImpl.performTraversals()
ViewRootImpl.performMeasure()
View.measure()
FrameLayout.onMeasure()
ViewGroup.measureChildWithMargins()
ViewGroup.getChildMeasureSpec()
```

## MeasureSpec
32位int值的高2位表示mode，低30位表示size。  
mode包括：
* `MeasureSpec.UNSPECIFIED` (0 << 30)
* `MeasureSpec.EXACTLY` (1 << 30)
* `MeasureSpec.AT_MOST` (2 << 30)

api:
```java
MeasureSpec.makeMeasureSpec()
MeasureSpec.getMode()
MeasureSpec.getSize()
```

## 如何确定View的MeasuredSize
源码关键路径：
```java
View.onMeasure()
View.getDefaultSize()
View.getSuggestMinimumWidth() / View.getSuggestMinimumHeight()
```

标准：
|Parent MS|EXACTLY|AT_MOST|UNSPECIFIED|
|-|:-:|:-:|:-:|
|Child Size|specSize|specSize|suggestMinimumSize|

SuggestMinimumWidth(Height)：
```java
return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
```

## 如何确定ViewGroup的MeasuredSize
不用的`ViewGroup`子类实现了不同的`View.onMeasure()`方法，用来计算MeasuredSize。

## 如何确定child的MeasureSpec限制
综合parent自身的MeasureSpec限制及child的LayoutParams需求，计算得到一个最合理的值。

标准：
|Child LP, Parent MS|EXACTLY|AT_MOST|UNSPECIFIED|
|:-:|:-:|:-:|:-:|
|x (>=0)|(x, EXACTLY)|(x, EXACTLY)|(x, EXACTLY)|
|MATCH_PARENT (-1)|(parentSize, EXACTLY)|(parentSize, AT_MOST)|(0, UNSPECIFIED)|
|WRAP_CONTENT (-2)|(parentSize, AT_MOST)|(parentSize, AT_MOST)|(0, UNSPECIFIED)|

说明：
* parentSize是减过child的margin和parent的padding后的值。
    ```java
    protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    ```
* `(0, UNSPECIFIED)`在`targetSdkVersion < M`时，使用的是`0`。否则使用的是`parentSize`。
    ```java
    // View构造方法

    // In M and newer, our widgets can pass a "hint" value in the size
    // for UNSPECIFIED MeasureSpecs. This lets child views of scrolling containers
    // know what the expected parent size is going to be, so e.g. list items can size
    // themselves at 1/3 the size of their container. It breaks older apps though,
    // specifically apps that use some popular open source libraries.
    sUseZeroUnspecifiedMeasureSpec = targetSdkVersion < Build.VERSION_CODES.M;
    ```
    ```java
    // ViewGroup.getChildMeasureSpec()

    resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
    ```

## 为什么有两次Measure
第一次measure时无法确定LayoutParams为`MATCH_PARENT`的child具体的measuredSize。  
需要在Parent确定大小后，再次调用Child的`View.measure()`方法确定具体大小。

# Layout
核心思路：
* parent调用child的`View.layout()`方法确定child的layout坐标。
    * child在`View.layout()`方法中调用`View.setFrame()`方法记录坐标。
    * child在`View.layout()`方法中调用`View.onLayout()`方法。
    * 重写`View.onLayout()`方法，递归进行子View的Layout。

# Draw
核心思路：
* parent调用child的`View.draw()`绘制child。
    * child在`View.draw()`方法中调用`View.drawBackgrond()`方法绘制背景。
    * child在`View.draw()`方法中调用`onDraw()`方法绘制自己的内容。
    * child在`View.draw()`方法中调用`dispatchDraw()`方法，递归进行子View的绘制。

# Api
## `View.invalidate()`与`View.requestLayout()`
* `View.invalidate()`只会执行`View.onDraw()`
    > onDraw() will be called at some point in the future.
* `View.requestLayout()`只会执行`View.onMeasure()`和`View.onLayout()`方法。
